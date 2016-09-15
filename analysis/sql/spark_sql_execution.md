#Spark SQL ִ�н׶�

�����Ǵ�����PhysicalPlan֮��ʼ˵��

	//QueryExecution
	lazy val executedPlan: SparkPlan = prepareForExecution(sparkPlan)

	lazy val toRdd: RDD[InternalRow] = executedPlan.execute()
	
��Ҫ�������������䡣����`executedPlan`�Ƕ������ɵ�PhysicalPlan��һЩ������Ҫ�ǲ���shuffle������internal row�ĸ�ʽת����

	  //QueryExecution
	  protected def prepareForExecution(plan: SparkPlan): SparkPlan = {
		preparations.foldLeft(plan) { case (sp, rule) => rule.apply(sp) }
	  }

	  protected def preparations: Seq[Rule[SparkPlan]] = Seq(
		python.ExtractPythonUDFs,
		PlanSubqueries(sparkSession),
		EnsureRequirements(sparkSession.sessionState.conf),
		CollapseCodegenStages(sparkSession.sessionState.conf),
		ReuseExchange(sparkSession.sessionState.conf),
		ReuseSubquery(sparkSession.sessionState.conf))

##Preparations����

��Python��صĲ����ܡ�

###PlanSubqueries

	case class PlanSubqueries(sparkSession: SparkSession) extends Rule[SparkPlan] {
	  def apply(plan: SparkPlan): SparkPlan = {
		plan.transformAllExpressions {
		  case subquery: expressions.ScalarSubquery =>
			val executedPlan = new QueryExecution(sparkSession, subquery.plan).executedPlan
			ScalarSubquery(
			  SubqueryExec(s"subquery${subquery.exprId.id}", executedPlan),
			  subquery.exprId)
		  case expressions.PredicateSubquery(query, Seq(e: Expression), _, exprId) =>
			val executedPlan = new QueryExecution(sparkSession, query).executedPlan
			InSubquery(e, SubqueryExec(s"subquery${exprId.id}", executedPlan), exprId)
		}
	  }
	}
	
1. ��ScalarSubquery����SparkPlan����ʵ�ع���һƪ[Spark SQL Physical Plan][1]����û�ж�ScalarSubquery������ԭ����ڣ�ʵ���������������SQL��䣬�ǲ����������ڵ�ģ�
ֻ������Ϊ����LogicalPlan��ʱ��ſ��ܻ���֡�����Subquery֮ǰ��һֱû�б�ת���ģ�����ͽ���ת�������Ӧ��SparkPlan�ڵ���ScalarSubquery��execution������֮ǰ��ScalarSubquery��expressions���ǲ�ͬ�ģ��������ڲ�ͬ�İ���ֻ��ͬ����
2. ��PredicateSubquery����SparkPlan����νPredicateSubquery�������Ӳ�ѯ�������Ӳ�ѯ������Ƿ����ĳ��ֵ������ֻ����ν�ʱ��ʽ����Filter Plan�У�WHERE��HAVING���У���ʵ���ϸò�ѯ��ScalarSubqueryһ�������ֱ�ӽ���SQL��䣬�ǲ�����������ġ�
��ʵSubquery���������ֻ����������SQL��䲻�����ɵģ�����������Exist��ListQuery����IN�����Ŀ飩���ù����������InSubquery��������Ǽ���ѯ�еļ�¼��������ν�ʱ��ʽ�Ľ�������Ըýڵ�Ľ����true��false��null���֡�

###EnsureRequirements

	//EnsureRequirements
	def apply(plan: SparkPlan): SparkPlan = plan.transformUp {
		case operator @ ShuffleExchange(partitioning, child, _) =>
		  child.children match {
			case ShuffleExchange(childPartitioning, baseChild, _)::Nil =>
			  if (childPartitioning.guarantees(partitioning)) child else operator
			case _ => operator
		  }
		case operator: SparkPlan => ensureDistributionAndOrdering(operator)
	}
	
1. ��ShuffleExchange���͵Ľڵ������������ֻ�������repartition��ʱ����������ýڵ���ӽڵ��Ѿ����Ա�֤partition����ô�����ӽڵ����ýڵ㡣

	//EnsureRequirements
	private def ensureDistributionAndOrdering(operator: SparkPlan): SparkPlan = {
		val requiredChildDistributions: Seq[Distribution] = operator.requiredChildDistribution
		val requiredChildOrderings: Seq[Seq[SortOrder]] = operator.requiredChildOrdering
		assert(requiredChildDistributions.length == operator.children.length)
		assert(requiredChildOrderings.length == operator.children.length)

		def createShuffleExchange(dist: Distribution, child: SparkPlan) =
		  ShuffleExchange(createPartitioning(dist, defaultNumPreShufflePartitions), child)

		var (parent, children) = operator match {
		  case PartialAggregate(childDist) if !operator.outputPartitioning.satisfies(childDist) =>
			val (mergeAgg, mapSideAgg) = AggUtils.createMapMergeAggregatePair(operator)
			(mergeAgg, createShuffleExchange(requiredChildDistributions.head, mapSideAgg) :: Nil)
		  case _ =>
			// Ensure that the operator's children satisfy their output distribution requirements:
			val childrenWithDist = operator.children.zip(requiredChildDistributions)
			val newChildren = childrenWithDist.map {
			  case (child, distribution) if child.outputPartitioning.satisfies(distribution) =>
				child
			  case (child, BroadcastDistribution(mode)) =>
				BroadcastExchangeExec(mode, child)
			  case (child, distribution) =>
				createShuffleExchange(distribution, child)
			}
			(operator, newChildren)
		}

		def requireCompatiblePartitioning(distribution: Distribution): Boolean = distribution match {
		  case UnspecifiedDistribution => false
		  case BroadcastDistribution(_) => false
		  case _ => true
		}
		...

		children = withExchangeCoordinator(children, requiredChildDistributions)

		...

		parent.withNewChildren(children)
	}
	
2. �������͵Ľڵ�Ĵ��������ϡ�

* DIstribution��ʾ���ݵķֲ���ʽ����ֲ���ʽ���������棬��Ⱥ�ֲ��͵���partition�ڵķֲ����ֱ��ʾ������α��ֲ�����Ⱥ�Ϻ�ÿ��partition�ڲ��ķֲ������`requiredChildDistribution`���в�ͬ��ʵ�֣��򵥿���һ��Aggregation������ʵ�֣���ʵ���Ǿ���������ϵ��tuple����һ���ñ��ʽ�ж���ϵ�������õ�λ��һ��SparkPlan�ڵ㡣
* SortOrder��һ�����ʽ�������ڶ�tuple�����������õ�λ��Expression��

���Է���`requiredChildDistributions`��`requiredChildOrderings`���ȶ��Ǹ�SparkPlan���ӽڵ������Ҳ����ÿ���ӽڵ�����в�ͬ�ķֲ�������

Ȼ������PartialAggregate��ȡ����Plan��һ���ӽڵ�ķֲ���ʽ��ͬʱ�ж��Ƿ�֧��partial aggregations���ò���������combiner�������÷�Χ���㣬��֧�ָ����ԵĲ������࣬��[��һƪ][1]���н��ܣ������������������÷ֲ�����ô��Ϊ������һ��map�˵�aggregation�ڵ��һ���ϲ��ڵ㣬��������map�˵�aggregation�ڵ�����ShuffleExchange�ڵ㡣
**�����֮�����һ��Aggregation������ҪShuffle����֧��partial aggregations������map�˽��в���aggregations��Ȼ��shuffle�����ϲ���**

���������ڵ㣬��������ӽڵ������Ѿ�������ӽڵ��Ӧ�ķֲ��������ֱ�����ӽڵ���棨ʡȥ�����·ֲ�����shuffle�Ĺ��̣�������ӽڵ�ķֲ�ģʽ��`BroadcastDistribution`����Ϊ������һ��`BroadcastExchangeExec`���͵Ľڵ㡣���������ֱ������ShuffleExchange�ڵ㡣

ʡ�ԵĴ���ܳ�����Ҫ���ܾ��ǵ���SparkPlan�ж���ӽڵ��ʱ�򣬲���ָ�����ӽڵ�ķֲ�����ô�ӽڵ���������һ��Ҫ�໥���ݡ��治���ݵ��ж������ǣ���������ͬһ�������ݣ���������ͬ������·����������Ƿ���ͬ��
��������ݣ������ж��ӽڵ����������Ƿ�������Եķֲ����ԣ�����ǾͲ���������Ϊ`requiredChildDistributions`�еķֲ��Ǽ��ݵģ��������������ӽڵ��������Ҳ�໥���ݣ���
����Ѳ��������з�������ȷ���µķ���������������������ӽڵ�����ķ������Ӧ�ֲ����Զ������ݣ���ô����Ĭ�Ϸ�����Ŀ����֮������Щ�ڵ������ķ�����Ŀ�����ɨ����ڵ㣬������µķֲ����Բ�ƥ�䣬��ô�Ͷ����������Shuffle���ֲ�������Ȼ�ö�Ӧ�������ԣ�������Ŀ������ȷ���ģ����γ��µķ�����

�������Ĺ�������Ϊ�ӽڵ�����ExchangeCoordinator�������ʾһ��Э������������������þ���ȷ��Shuffle��partition����Ŀ��

����ʡ�ԵĴ����ʾΪ�ӽڵ����ӱ�Ҫ������ʽ������������ǰ�棬��������ӽڵ�����������Ѿ������Ӧ�����򣬼�ֱ�����ӽڵ㣬������������һ������ڵ㡣���һ����Ǻϲ���������ṹ���£�

![add shuffle and sort][ensureDistributionAndOrdering]

###CollapseCodegenStages

�ù�������֧��codegen�Ľڵ㶥�˲���WholeStageCodegen���ɲο�[Apache Spark as a Compiler: Joining a Billion Rows per Second on a Laptop][2]��

	//WholeStageCodegenExec.scala
	private def insertWholeStageCodegen(plan: SparkPlan): SparkPlan = plan match {
		// For operators that will output domain object, do not insert WholeStageCodegen for it as
		// domain object can not be written into unsafe row.
		case plan if plan.output.length == 1 && plan.output.head.dataType.isInstanceOf[ObjectType] =>
		  plan.withNewChildren(plan.children.map(insertWholeStageCodegen))
		case plan: CodegenSupport if supportCodegen(plan) =>
		  WholeStageCodegenExec(insertInputAdapter(plan))
		case other =>
		  other.withNewChildren(other.children.map(insertWholeStageCodegen))
	}
	private def supportCodegen(plan: SparkPlan): Boolean = plan match {
		case plan: CodegenSupport if plan.supportCodegen =>
		  val willFallback = plan.expressions.exists(_.find(e => !supportCodegen(e)).isDefined)
		  val hasTooManyOutputFields =
			numOfNestedFields(plan.schema) > conf.wholeStageMaxNumFields
		  val hasTooManyInputFields =
			plan.children.map(p => numOfNestedFields(p.schema)).exists(_ > conf.wholeStageMaxNumFields)
		  !willFallback && !hasTooManyOutputFields && !hasTooManyInputFields
		case _ => false
	}

ֻ��һ���������ڶ���case�������WholeStageCodegen����һ��������������������ض����Ͳ��ҽ����һ�����ԡ���ôʲô����֧��codegen�أ����ȱ���̳���CodegenSupport���ʣ�����`supportCodegen`Ϊtrue��
�ܶ�ڵ㶼ʵ���˸�trait�������Ƿ�֧�ֵ��ж�����������ͬ�����ﲻ׸���������ж�`supportCodegen`�����أ�`willFallBack`�����ж��Ƿ��б��ʽ�Ĵ�������֧�ֻع���Ϊʲô�ع��Ĳ�������WholeStageCodegen��
�������Ϊʵ���˸�trait���౾��û����Ӧ��ִ�з�ʽ�������̫���ӻ�����Щ����������޷����ɵ����ɵĴ����У�����û����java��scala���������ɡ�`hasTooManyOutputFields`��֤������������Ͳ�����̫���field��
����`hasTooManyInputFields`����ÿ���ӽڵ����������Ҳ�����ܳ������ޣ�conf.wholeStageMaxNumFields����

###ReuseExchange

�ù���������ظ���Exchange�ġ�Exchange��һ�������࣬�������඼�Ͷ��̻߳���̵����ݽ����йأ�֮ǰ�ᵽ��ShuffleExchange��BroadcastExchangeExec�������е��������ࡣ���֮ǰ����ͬ���͵�Exchange��������������ͬ�����������жϵݹ�������ͣ�����������������С�Ƿ���ͬ����������˵���ṹ��ͬ������������ͬ����
��ô����֮ǰ���ɵĽڵ��滻��������֮ǰ���ɵĽڵ㡣�ù����߼��Ƚϼ�����������Ͳ��������ˡ�

###ReuseSubquery

�������ͬReuseExchange��ֻ�ǲ������󻻳���ExecSubqueryExpression��

##execute��������

ÿ��SparkPlan����һ��`execute`�����������`doExecute`���������������ʵ�־����doExecute������������select XXX from XXX where XXXΪ�����ܡ�������漰����SparkPlan�ڵ���٣�ֻ��FilterExec��ProjectExec��InMemoryTableScanExec��������InMemoryRealtion����

![execution example][execution]

###ProjectExecִ��

	//ProjectExec
	protected override def doExecute(): RDD[InternalRow] = {
		child.execute().mapPartitionsInternal { iter =>
		  val project = UnsafeProjection.create(projectList, child.output,
			subexpressionEliminationEnabled)
		  iter.map(project)
		}
	}

ִ�й��̺ܼ򵥾�������ӽڵ�����RDD[InternalRow]����partitionΪ��λִ��ͶӰ������ע���������ɵĲ������������ݣ�����RDD[InternalRow]������֮ǰ˵��InternalRow��������ʵ�����ݡ�Ҳ����execute��transform����������action������
����RDD[InternalRow]��һ������Ҫ�����þ���֪��ÿ�������Ķ������磺Row[InternalRow]��¼����һ���ǶԵ�x����Ԫ�����ݼ�һ��x��һ���������������ִ�е�ʱ���ⲿ�ִ���ֻ�����������ǵ����ݡ������֮RDD[InternalRow]����������Ӧ���롣
���溯���е�`project`���൱��һ��ͶӰ�����Ĵ������ɺ�����

###FilterExecִ��

	//FilterExec
	protected override def doExecute(): RDD[InternalRow] = {
		val numOutputRows = longMetric("numOutputRows")
		child.execute().mapPartitionsInternal { iter =>
		  val predicate = newPredicate(condition, child.output)
		  iter.filter { row =>
			val r = predicate(row)
			if (r) numOutputRows += 1
			r
		  }
		}
	}
	
`predicate`�൱��һ�����������жϸ���������������������������һ���Ǵ������ɵĺ�����

###InMemoryTableScanExecִ��

�ýڵ��`doExecute`�����Ƚϸ��ӡ���˼·���ǲ��������У���Ϊ����ǰ����Ż���û��Ҫ��������У�����Ϊ�����

##��������ݵ���ϵ

��`collect`����Ϊ����

	//DataSet
	def executeCollect(): Array[InternalRow] = {
		val byteArrayRdd = getByteArrayRdd()

		val results = ArrayBuffer[InternalRow]()
		byteArrayRdd.collect().foreach { bytes =>
		  decodeUnsafeRows(bytes).foreach(results.+=)
		}
		results.toArray
	}
	
	private def getByteArrayRdd(n: Int = -1): RDD[Array[Byte]] = {
		execute().mapPartitionsInternal { iter =>
		  var count = 0
		  val buffer = new Array[Byte](4 << 10)  // 4K
		  val codec = CompressionCodec.createCodec(SparkEnv.get.conf)
		  val bos = new ByteArrayOutputStream()
		  val out = new DataOutputStream(codec.compressedOutputStream(bos))
		  while (iter.hasNext && (n < 0 || count < n)) {
			val row = iter.next().asInstanceOf[UnsafeRow]
			out.writeInt(row.getSizeInBytes)
			row.writeToStream(out, buffer)
			count += 1
		  }
		  out.writeInt(-1)
		  out.flush()
		  out.close()
		  Iterator(bos.toByteArray)
		}
	}
	
`byteArrayRdd`���Ǹ�DataSet��Ӧ��RDD[Array[Byte]]��`getByteArrayRdd`��������`execute`���ɵ�RDD[InternalRow]��RDD[InternalRow]��������ϵ��������������Interator[InternalRow]��UnsafeRow��

###Iterator[InternalRow]��UnsafeRow

InternalRow��Ȼ���������ݣ����ǲ�������Iterator[InternalRow]����������ݣ���Ϊ`hasNext`��`next`�Ⱥ������Ա���д��
������[InMemoryTableScanExec][###InMemoryTableScanExecִ��]��û���ᵽ������`execute`������`GenerateColumnAccessor.generate(columnTypes)`�����ú����ֵ�����`GenerateColumnAccessor.create(columnTypes)`��
`create`����Ҫ���ܾ������ɴ��루�������ַ���������IDE������ⲻ���������ɵĴ�����Ҫ��ʵ����SpecificColumnarIterator�࣬������ColumnarIterator���̳���Interator[InternalRow]�������ࡣ���о���`buffers`��`rowWriter`�ȱ�����

	//GenerateColumnAccessor
	protected def create(columnTypes: Seq[DataType]): ColumnarIterator = {
		...
		val codeBody = s"""
		  import ...

		  public SpecificColumnarIterator generate(Object[] references) {
			return new SpecificColumnarIterator();
		  }

		  class SpecificColumnarIterator extends ${classOf[ColumnarIterator].getName} {

			private ByteOrder nativeOrder = null;
			private byte[][] buffers = null;
			private UnsafeRow unsafeRow = new UnsafeRow($numFields);
			private BufferHolder bufferHolder = new BufferHolder(unsafeRow);
			private UnsafeRowWriter rowWriter = new UnsafeRowWriter(bufferHolder, $numFields);
			private MutableUnsafeRow mutableRow = null;

			private int currentRow = 0;
			private int numRowsInBatch = 0;

			private scala.collection.Iterator input = null;
			private DataType[] columnTypes = null;
			private int[] columnIndexes = null;

			...

			public void initialize(Iterator input, DataType[] columnTypes, int[] columnIndexes) {
			  this.input = input;
			  this.columnTypes = columnTypes;
			  this.columnIndexes = columnIndexes;
			}

			${ctx.declareAddedFunctions()}

			public boolean hasNext() {
			  ...
			}

			public InternalRow next() {
			  currentRow += 1;
			  bufferHolder.reset();
			  rowWriter.zeroOutNullBytes();
			  ${extractorCalls}
			  unsafeRow.setTotalSize(bufferHolder.totalSize());
			  return unsafeRow;
			}
		  }"""

		val code = CodeFormatter.stripOverlappingComments(
		  new CodeAndComment(codeBody, ctx.getPlaceHolderToComments()))
		...
		CodeGenerator.compile(code).generate(Array.empty).asInstanceOf[ColumnarIterator]
	}

������ͻ���ᵽ��ʵ��Interator[InternalRow]�ͻ��������ˡ�������Ҫ���õ�4��������`buffer`��`unsafeRow`��`bufferHolder`��`rowWriter`��`buffer`�����ͺý��͡������������3����������͡�

####UnsafeRow

�����ͼ̳���InternalRow��������������ݣ������������ԭ���ڴ���������Բ���Java���������ݱ�����`baseObject`�����У��ö�����Object���͡�

	//UnsafeRow
	public byte getByte(int ordinal) {
		assertIndexIsValid(ordinal);
		return Platform.getByte(baseObject, getFieldOffset(ordinal));
	}
	private long getFieldOffset(int ordinal) {
		return baseOffset + bitSetWidthInBytes + ordinal * 8L;
	}

ÿ��Tuple����������ɣ�[null bit set] [values] [variable length portion]��`baseOffset`��һ�������ͷ��ռ�õĳ��ȡ�

* [null bit set] ����nullλ�ĸ��٣�������**8-Byte**�����û��field�����򳤶�Ϊ0�������Ϊÿ��field�洢һ��**bit**����0��1��ʾ�Ƿ�Ϊnull��
* [values]����Ϊÿ��field�洢��8-Byte�����ݣ���Ȼ���ڹ̶����ȵ�ԭʼ���ͣ���long��int��double�ȣ�ֱ�Ӵ��롣���ڷ�ԭʼ���ͻ�ɱ䳤�ȵ�ֵ���洢����һ�����offset��ָ��䳤field����ʼλ�ã��͸�field�ĳ��ȣ�[variable length portion]����
����UnsafeRow������Ե�����һ��ָ��ԭʼ���ݵ�ָ�롣

####BufferHolder

����ʵ����������������ӦUnsafeRow�ġ�

	//BufferHolder
	public BufferHolder(UnsafeRow row, int initialSize) {
		int bitsetWidthInBytes = UnsafeRow.calculateBitSetWidthInBytes(row.numFields());
		if (row.numFields() > (Integer.MAX_VALUE - initialSize - bitsetWidthInBytes) / 8) {
		  ...
		}
		this.fixedSize = bitsetWidthInBytes + 8 * row.numFields();
		this.buffer = new byte[fixedSize + initialSize];
		this.row = row;
		this.row.pointTo(buffer, buffer.length);
	}
	//UnsafeRow
	public void pointTo(byte[] buf, int sizeInBytes) {
		pointTo(buf, Platform.BYTE_ARRAY_OFFSET, sizeInBytes);
	}
	public void pointTo(Object baseObject, long baseOffset, int sizeInBytes) {
		assert numFields >= 0 : "numFields (" + numFields + ") should >= 0";
		this.baseObject = baseObject;
		this.baseOffset = baseOffset;
		this.sizeInBytes = sizeInBytes;
	}
	
����3���������ǽ�byte�������UnsafeRow�У�����д��Ĳ�������UnsafeRowWriter�����ġ�

####UnsafeRowWriter

��Բ�ͬ�����ͻ��в�ͬ��д�뷽����Ϊ�˼�࣬����ѡ��ԭʼ���ͣ���˵��ԭ��

	//UnsafeRowWriter
	public void write(int ordinal, long value) {
		Platform.putLong(holder.buffer, getFieldOffset(ordinal), value);
	}
	public long getFieldOffset(int ordinal) {
		return startingOffset + nullBitsSize + 8 * ordinal;
	}
	public void reset() {
		this.startingOffset = holder.cursor;

		holder.grow(fixedSize);
		holder.cursor += fixedSize;

		zeroOutNullBytes();
	}
	//BufferHolder
	public void grow(int neededSize) {
		if (neededSize > Integer.MAX_VALUE - totalSize()) {
		  ...
		}
		final int length = totalSize() + neededSize;
		if (buffer.length < length) {
		  // This will not happen frequently, because the buffer is re-used.
		  int newLength = length < Integer.MAX_VALUE / 2 ? length * 2 : Integer.MAX_VALUE;
		  final byte[] tmp = new byte[newLength];
		  Platform.copyMemory(
			buffer,
			Platform.BYTE_ARRAY_OFFSET,
			tmp,
			Platform.BYTE_ARRAY_OFFSET,
			totalSize());
		  buffer = tmp;
		  row.pointTo(buffer, buffer.length);
		}
	}

UnsafeRowWriter��`write`�������ǰ�����д��BufferHolder��`buffer`�У�Ȼ�����`reset`������BufferHolder�������ύ��UnsafeRow��`baseObject`�С�BufferHolder���ύ���̷�����`grow`�����У�
���Կ���BufferHolderʵ����ֻ��һ��buffer������һ��Iterator��Ҳֻ��һ��UnsafeRow������ֻ��BufferHolderÿ�������µ�buffer��д��UnsafeRow������ȡ�����������һ�ּ����ƺ���ÿ��������UnsafeRow��
�ⲿ�ֵ��߼����ڴ������ɵ��У����Ժ��ҵ�����ԭ���ǻ���һ�µġ�

![iterator InternalRow][iterator]


	
[1]:https://github.com/summerDG/spark-code-ananlysis/blob/master/analysis/sql/spark_sql_physicalplan.md
[2]:https://databricks.com/blog/2016/05/23/apache-spark-as-a-compiler-joining-a-billion-rows-per-second-on-a-laptop.html
[ensureDistributionAndOrdering]:../../pic/sort-distribution.png
[execution]:../../example.png
[iterator]:../../pic/Iterator.png




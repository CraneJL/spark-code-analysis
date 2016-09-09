#Spark Catalyst �����׶�

���Ĵ�ʹ���ߵ��ӽǣ�һ��������SQL�ķ������ⲿ�ִ�SQL��俪ʼ����LogicalPlanΪ�����

![catalyst-analysis][analysis]

�����Թٷ���һ�δ�����Ϊ��������̡�

	import org.apache.spark.sql.SparkSession

	val spark = SparkSession
	  .builder()
	  .appName("Spark SQL Example")
	  .config("spark.some.config.option", "some-value")
	  .getOrCreate()

	// For implicit conversions like converting RDDs to DataFrames
	import spark.implicits._

	case class Person(name: String, age: Long)

	val peopleDF = spark.sparkContext
	  .textFile("examples/src/main/resources/people.txt")
	  .map(_.split(","))
	  .map(attributes => Person(attributes(0), attributes(1).trim.toInt))
	  .toDF()
	peopleDF.createOrReplaceTempView("people")
	val teenagersDF = spark.sql("SELECT name, age FROM people WHERE age BETWEEN 13 AND 19")
	
##����ÿ����¼������

SparkSession��DataSet��DataFrame��̵���ڣ�Builder����������REPL��notebooks�����û�������һ����ص��ǻ�ȡ����ǰ������SparkSession��
�ڶ�����������Spark SQL�е���ʽת������ʽֵ��RDD.toDF��toDS�Ȳ�������ͨ����ʽת��ʵ�ֵģ���������Ȼ��DataFrame������֮ǰ˵����2.0.0
��DataFrame����DataSet[Row]�����Բ�Ӱ�����Ƿ���DataSet��`toDF`��`toDS`����ʹ��������ʽת����

	implicit def rddToDatasetHolder[T : Encoder](rdd: RDD[T]): DatasetHolder[T] = {
		DatasetHolder(_sqlContext.createDataset(rdd))
	}
	case class DatasetHolder[T] private[sql](private val ds: Dataset[T]) {
	  def toDS(): Dataset[T] = ds
	  def toDF(): DataFrame = ds.toDF()
	  def toDF(colNames: String*): DataFrame = ds.toDF(colNames : _*)
	}
	
���Է�����RDD��DataSet��ת�����õ���SQLContext��`createDataset`��������Ԥ�ȱض�û��Encoder[Person]����ʽֵ����ʽת����ͨ��

	implicit def newProductEncoder[T <: Product : TypeTag]: Encoder[T] = Encoders.product[T]
	
��ΪPerson��case class��������Product���͡�����Encoder��`product`��Ȼ�����ExpressionEncoder��`apply`������������õ���[Spark SQL ����֪ʶ][1]�е��ᵽ�ķ�����ơ�

def apply[T : TypeTag](): ExpressionEncoder[T] = {
    val mirror = typeTag[T].mirror
    val tpe = typeTag[T].tpe
    val cls = mirror.runtimeClass(tpe)
    val flat = !ScalaReflection.definedByConstructorParams(tpe)

    val inputObject = BoundReference(0, ScalaReflection.dataTypeFor[T], nullable = true)
    val nullSafeInput = if (flat) {
      inputObject
    } else {
      AssertNotNull(inputObject, Seq("top level non-flat input object"))
    }
    val serializer = ScalaReflection.serializerFor[T](nullSafeInput)
    val deserializer = ScalaReflection.deserializerFor[T]

    val schema = ScalaReflection.schemaFor[T] match {
      case ScalaReflection.Schema(s: StructType, _) => s
      case ScalaReflection.Schema(dt, nullable) => new StructType().add("value", dt, nullable)
    }

    new ExpressionEncoder[T](
      schema,
      flat,
      serializer.flatten,
      deserializer,
      ClassTag[T](cls))
}

`flat`�����ж�������ǲ�����ȫ�ɳ�Ա�������죬����Ǿͽ�������Ա��������Ϊ���ԣ���֮�����쳣������`ScalaReflection.serializerFor`��
���������̡����case class�����ǿ�`serializerFor`��private������Ӧ�����

	private def serializerFor(
		  inputObject: Expression,
		  tpe: `Type`,
		  walkedTypePath: Seq[String]): Expression = ScalaReflectionLock.synchronized {
		  ...
		  tpe match {
		  ...
			  case t if definedByConstructorParams(t) =>
				val params = getConstructorParameters(t)
				val nonNullOutput = CreateNamedStruct(params.flatMap { case (fieldName, fieldType) =>
				  if (javaKeywords.contains(fieldName)) {
					throw new UnsupportedOperationException(s"`$fieldName` is a reserved keyword and " +
					  "cannot be used as field name\n" + walkedTypePath.mkString("\n"))
				  }

				  val fieldValue = Invoke(inputObject, fieldName, dataTypeFor(fieldType))
				  val clsName = getClassNameFromType(fieldType)
				  val newPath = s"""- field (class: "$clsName", name: "$fieldName")""" +: walkedTypePath
				  expressions.Literal(fieldName) :: serializerFor(fieldValue, fieldType, newPath) :: Nil
				})
				val nullOutput = expressions.Literal.create(null, nonNullOutput.dataType)
				expressions.If(IsNull(inputObject), nullOutput, nonNullOutput)
			...
			}
	}

ͨ��������ƣ��������н������������������ֺͶ�Ӧ�����ͣ������жϲ������Ƿ�Ϸ���`Invoke`���������뱻ת��ΪExpression���ࡢ�������Ͷ�Ӧ�������͡��Ӷ���Ч�ĵ��ö�Ӧ������
����Scala�еĳ�Ա������Ҳ������Ϊ���������룬���������൱�ڻ�ȡ��Ա����������ʵ��һ���󶨵Ĺ��̣���ֱ�Ӵ�ԭ�����ж�ȡ���ݣ�ע�����ﲢû�а�������ͣ�Person��ת��Ϊ����������
��ֻ�ǽ���������Person�ľ����Ա�������ú��������˰󶨣����Զ�ȡ������Ȼ�Ǵ�Person�ж�ȡ�������Է������Ǹ��ݹ�Ĺ��̣���Ϊ���ܳ�Ա����������Ȼ����Int��String�Ȼ������ͣ��Ǿ�Ҫ����������
`expressions.Literal(fieldName) :: serializerFor(fieldValue, fieldType, newPath) :: Nil`����ĳЩ�����ķ���ֵ��Ϊ�������Object����֮����Ҫ���ɴ���ѽ��ǿ��ת��Ϊ�������ͣ��ⲿ�ִ�����
Invoke��`doGenCode`�����С�

���Խ�����֮��`serializer`���ǽ����и���Ա������ӳ�䵽���Զ������͵ĳ�Ա������ȡ�����У�����ͨ��`serializer`���Խ��Զ��������е�������ȡ�������в�����
��֮���Բ��뵽`deserializer`���ǽ���Ա�����ĸ�ֵ����ӳ�䵽����Ա���������Ӷ����Խ����������������д�뵽�Զ��������С���Ȼ�����ͬ���߱����л��ͷ����л��Ĺ��ܡ�

> ����Java�Ͳ��ܴ������������⣬�ص�����Scala֧�����Ͳ���֮��ԭ��TypeTag�Ĺ��ͣ�������������Bash�ű���д����ʹ�ô��������൱���㡣������ĺ����Spark����ʦ�뵽���÷��䣬�Լ���Expression��
Schema���Ӷ��ﵽ֧���������͵�Ŀ�ġ��ⲿ�ִ���ܶ࣬������˼·�����Ѿ�����������ٶ�˵һ�䣬Scala����һ���������������22��field�����˵Ļ��͵ü̳�Product�ˡ�

֮��������÷�������Schema��ͬ��Ҳ�ǵݹ���̣���`serializer`��`deserializer`�����������䲻�ð󶨶�ȡ���ƺ����������ǳ�Ա���������������Ķ�Ӧ��ϵ��

�����������������ɵĶ�����ExpressionEncoder��ÿ��DataSet[T]�м�¼������T������һ��ExpressionEncoder��

##����DataSet

Encoder����֮�󣬹�עDataSet�����ɡ�

	def createDataset[T : Encoder](data: RDD[T]): Dataset[T] = {
		Dataset[T](self, ExternalRDD(data, self))
	}
	
��Ҫ��ExternalRDD����ʵ���������þ��ǽ�RDDת��ΪLogicalPlan�Ľڵ㣨������Ӧ����Ҷ�ӽڵ㣩������ɨ��RDD���ݡ�������ExternalRDD֮ǰ�ȹ�עһ��`CatalystSerde.generateObjAttr`��
�÷�����Ҫ�ǵ���AttributeReference��`apply`��������һ�������Ե����ã�������͵������У�

1. �ж����������Ƿ�ָ��ͬһ�����ԣ���Ϊÿ�����Զ�ֻ��һ��ID��
2. �ж������Ƿ���ͬ��
3. ������������
4. �������η��������磺tableName.name��subQueryAlias.name��tableName��subQueryAlias�������η�����

ExternalRDD�����ý�Ϊ�򵥣�

1. ���ɱ�����Ŀ����������������ֻ�������ɶ�ӦAttributeReference�Ŀ�������Ӧrdd�����ǿ�����Ҳ���Ǳ�����ֻ�����ڹ���LogicalPlan����ΪPlan�п��ܻ���
�õ�ͬһAttribute�����ý��в�ͬ�����������ı�ṹ������Щ�������ն������õ�ͬһ��RDD�ģ�
2. ��������һ��Plan��Ӧ��RDD�Ƿ���ͬһ�����ƺ�������Physical Plan�׶Σ���

ExternalRDD������һ��LogicalPlan�ڵ�����࣬������Ҷ�ӽڵ㣬������׽���ͨ����Ϊ����û���κβ�ѯ�߼���������Ӧ�ñ�����Ҷ�ӽڵ����������ݣ��Թ��м�ڵ���д���

���ˣ�DataSet������ϡ�����ʵ�������������Ĳ�������Lazy�ģ�ֻ���ڴ�����ʱ�򣬲Ż�ִ��QueryPlan��Ҳ����˵�����`toDF`��`toDS`������ֻ��ת��������
`createTempView`������Command���������Կ�������ִ�У����Ǹ����DataSet���������ͼ����������������SparkSession����������Ȼ�ò�������֤�������ȫ��Ψһ��

##Query������

`spark.sql(...)`���������ǵ���`SparkSqlParser.parsePlan`������������ѯ��䡣ʵ�ʵ��õ���`AbstractSqlParser.parse`��

	protected def parse[T](command: String)(toResult: SqlBaseParser => T): T = {
		val lexer = new SqlBaseLexer(new ANTLRNoCaseStringStream(command))
		lexer.removeErrorListeners()
		lexer.addErrorListener(ParseErrorListener)

		val tokenStream = new CommonTokenStream(lexer)
		val parser = new SqlBaseParser(tokenStream)
		parser.addParseListener(PostProcessor)
		parser.removeErrorListeners()
		parser.addErrorListener(ParseErrorListener)

		try {
		  try {
			parser.getInterpreter.setPredictionMode(PredictionMode.SLL)
			toResult(parser)
		  }
		  catch {
			case e: ParseCancellationException =>
			  tokenStream.reset() // rewind input stream
			  parser.reset()
			  parser.getInterpreter.setPredictionMode(PredictionMode.LL)
			  toResult(parser)
		  }
		}
		catch {
		  ...
		}
	}

��ʵ�ⲿ�ֵĴ�������˽�Antlr 4 �Ľ������̵Ļ�������׶������Բο�[Antlr v4���Ž̳̺�ʵ��][2]������������˽�ο�[ANTLR 4ȫΪ�ο�����ʼ�][3]��
Antlr���ȶ��ַ������дʷ���������`lexer`��������Spark�����`ParseErrorListener`�滻��ԭ�еĴʷ��������������ʵ���þ��ǽ���ȥ�Ĵ�������ת��Ϊ�쳣��Ϣ��
����`lexer`����token��������������Ȼ������token������`parse`���̣������﷨����`parse`�����м���Spark�Լ��Ľ�����`PostProcessor`�������������������磺
����ʶ�������������������е�������`��������������Ϊ��ͬ���ݿ������Ա��ϰ�߲�ͬ�����Լ������зǱ����֣�select��where�ȣ���tokenȫ��������ʶ����
֮�������﷨�����ɲ��ԣ�SLL��LL��ǰ�߿쵫����������û�����˽⣩���ص����`toResult`������������������̡�

	//AbstractSqlParser
	override def parsePlan(sqlText: String): LogicalPlan = parse(sqlText) { parser =>
		astBuilder.visitSingleStatement(parser.singleStatement()) match {
		  case plan: LogicalPlan => plan
		  case _ =>
			val position = Origin(None, None)
			throw new ParseException(Option(sqlText), "Unsupported SQL statement", position, position)
		}
	}

�����Ըù��̾ͻ�����LogicalPlan��`parser.singleStatement()`���ɶ�Ӧ�﷨����`singleStatement`�����ļ������Ľṹ�����Զ��壩��
����`ASTBuilder.visitSingleStatement`��

	override def visitSingleStatement(ctx: SingleStatementContext): LogicalPlan = withOrigin(ctx) {
		visit(ctx.statement).asInstanceOf[LogicalPlan]
	}

visit�Ǹ�����������ʱ���ɵģ����Դ�����û����ʾ��λ�á���singleStatement�е�statement��ȡ������Ȼ���ղ��һ��һ�������
����`SELECT name, age FROM people WHERE age BETWEEN 13 AND 19`�еı��������ǲ�ѯ��䡣

	//AstBuilder
	override def visitQuerySpecification(
		  ctx: QuerySpecificationContext): LogicalPlan = withOrigin(ctx) {
		val from = OneRowRelation.optional(ctx.fromClause) {
		  visitFromClause(ctx.fromClause)
		}
		withQuerySpecification(ctx, from)
	}
	override def visitFromClause(ctx: FromClauseContext): LogicalPlan = withOrigin(ctx) {
		val from = ctx.relation.asScala.map(plan).reduceLeft(Join(_, _, Inner, None))
		ctx.lateralView.asScala.foldLeft(from)(withGenerate)
	}

��һ���ȡ��from֮���﷨�飨���������View�������Ӳ�ѯ����LogicalPlan�ڵ㣬����ж�����󣬻����Join������ע���ⲽ��û�н���Join���㣬������ȷ�����������߼���Ȼ�󽻸�`withQuerySpecification`������������߼��Ƚ϶࣬����ֻ�������ж�Ӧ�ĳɷַ�����

	//AstBuilder
	private def withQuerySpecification(
		  ctx: QuerySpecificationContext,
		  relation: LogicalPlan): LogicalPlan = withOrigin(ctx) {
		import ctx._

		// WHERE
		def filter(ctx: BooleanExpressionContext, plan: LogicalPlan): LogicalPlan = {
		  Filter(expression(ctx), plan)
		}
		val expressions = Option(namedExpressionSeq).toSeq
		  .flatMap(_.namedExpression.asScala)
		  .map(typedVisit[Expression])

		val specType = Option(kind).map(_.getType).getOrElse(SqlBaseParser.SELECT)
		specType match {
		  ...

		  case SqlBaseParser.SELECT =>
			...
			val withLateralView = ctx.lateralView.asScala.foldLeft(relation)(withGenerate)
			// Add where.
			val withFilter = withLateralView.optionalMap(where)(filter)
			
			val namedExpressions = expressions.map {
				case e: NamedExpression => e
				case e: Expression => UnresolvedAlias(e)
			}
			val withProject = if (aggregation != null) {
			  withAggregation(aggregation, namedExpressions, withFilter)
			} else if (namedExpressions.nonEmpty) {
			  Project(namedExpressions, withFilter)
			} else {
			  withFilter
			}
			...
		}
	}

����Ľ�������ǰ��Ľڵ㣬�������Ⱦ��ǽ���View����Ȼ����ǹ�����䣬֮����������������Aggregate��������������ͶӰ��֮����having��distinct�Ȳ�����
���Է�������߼��Ǻܺ���ģ���ΪView�����ڲ������Ӳ�ѯ������£�����������룩���������룬֮�󾭹�����ȷ������������ԣ�Ȼ���ȡ����Ը����Ե�Aggregate����������Ǳ�����
Ȼ�����ͶӰ�����

��[Spark SQL ����֪ʶ][1]��̸��Generate�ǽ�һ�����ݵķ�������뵱ǰ�ķ���ƴ����һ�𡣵��������Ӳ�ѯ��û��Resolve����Generator��Unresolved�ģ���ʱ������ƴ�ӡ�����ᷢ���ڷ���FROM���ʱ��Ҳ������View��ƴ�ӣ�
��ô����Ĵ����������ʲô��ͬ�أ��ٸ�����

	...��SELECT * FROM (table1,table2,lateralView1)�� lateralView2 ...

`visitFromClause`������table1��2��Join�����lateralView1����ƴ�ӣ������������ƴ�ӣ�Ҳ���ǽ��Ӳ�ѯ��������lateralView2����ƴ�ӡ�

Ȼ�����`val withFilter = withLateralView.optionalMap(where)(filter)`��������where������ʱ�򣬽�WHERE��ߵ�������ӳ�䵽��Ϊ`withFilter`��LogicalPlan�ڵ��У����ҽ�`withLateralView`��Ϊ�ӽڵ�ΪFilter�ṩ���롣
�����Where���е�booleanExpression��Predicated��ν�ʣ�֧��BETWEEN��IN��LIKE��RLIKE��IS NULL���䷴������������Խ���withPredicated���Կ�����Ľ������̡�

	ctx.kind.getType match {
		  case SqlBaseParser.BETWEEN =>
			// BETWEEN is translated to lower <= e && e <= upper
			invertIfNotDefined(And(
			  GreaterThanOrEqual(e, expression(ctx.lower)),
			  LessThanOrEqual(e, expression(ctx.upper))))
			  ...
		}

������Ƿ���BTWEEN������ת��Ϊ������`e >=ctx.lower && e <= ctx.upper`�Ĳ�����

֮���ͶӰ��Project���������ƣ��������ܽ�Ϊ��ȡ�ڵ��е�Expression�������㣬Ȼ������LogicalPlan�ڵ㡣

> Expression������LogicalPlan�ڵ��У����൱�ڸýڵ�ļ��㵥Ԫ����LogicalPlan�ڵ�֮�����ϵ���Կ�����Ϊ���㵥Ԫ�ṩ��������ӿڡ�

![relation between expression and logicalPlan][expresion-logical]

����LogicalPlan֮�󣬾���Ҫ����`DataSet.ofRows`��

	//DataSet object
	def ofRows(sparkSession: SparkSession, logicalPlan: LogicalPlan): DataFrame = {
		val qe = sparkSession.sessionState.executePlan(logicalPlan)
		qe.assertAnalyzed()
		new Dataset[Row](sparkSession, qe, RowEncoder(qe.analyzed.schema))
	}
	
���Ⱦ��ǵ���`executePlan`��ִ��logicalPlan������QueryExecution���󣬵�ʵ����QueryExecution��ߵķ�����������Lazy�ģ����Կ���˵���ﷵ�صľ��Ǹ���׼�������Ļ��������ȴ�������
�ڶ������ڼ���Ƿ��в�֧�ֵĲ��������������table�����ڣ���û�м����������Attribute�����ڵȣ��Ա�������ֹ������롣

Ȼ�����QueryExecution�ķ���`analyzed`��ʼִ�С�

	//QueryExecution
	lazy val analyzed: LogicalPlan = {
		SparkSession.setActiveSession(sparkSession)
		sparkSession.sessionState.analyzer.execute(logical)
	}
	
`analyzer`����Ҳ��Lazy�ģ����Ի����Analyzer��`execute`������Unresolved��Attitude��Relation��ͨ��CataLogת��Ϊ���������õ����Ͷ���

##Analyzer

���ȿ�ʲô��Catalog������һ��SessionCatalog�࣬����ά��Spark SQL�б�����ݿ��״̬�������Ժ��ⲿϵͳ����Hive���������ӣ��Ӷ���ȡ��Hive�е����ݿ���Ϣ��
֮ǰ�ᵽ������`createTempView`���ɱ��������������ͬʱ��������View������Ϣע�ᵽ��Catalog�С�

����Analyzer�����ʱ�򣬿��Զ�`extendedResolutionRules`������д���ù������û�������ӵĹ������ڽ���������һ��������д��field������`extendedCheckRules`��
������������ڼ��Ϸ��ԵĹ���

Analyzer�а��������Ĺ��򣬹���Ϊ6�࣬����Ҫ�������ǣ��滻��Substitution���ͽ�����Resolution�������������һ���������������ࣩ���Ӷ�ʵ�ֶ�LogicalPlan��ת����

###Substitution

`CTESubstitution`�����ǽ�CTE���壨����WITH�飩���Ӳ�ѯ�滻Ϊ�ɴ���LogicalPlan������CTE����������﷨���Ľṹ�������ҽ�������û����ֱ�ӽ�CTE������ӿ�ӵ��﷨���������Դ˴�Ҫ��CTE������ӿ����°����������뵽������ѯ��LogicalPlan�У����ҽ�����WITH�������ɵ�relation����ΪResolved״̬��

> ���磺`WITH q as(SELECT time_taken FROM idp WHERE time_taken=15) SELECT * FROM q; `
> ��û�����и��������ʱ����������LogicalPlan�����ֱ��ӦWITH���`SELECT * FROM q`��Ȼ��ù�������þ�����qΪ�������ݣ���������LogicalPlan���кϲ�����ôWITH���еĲ�ѯ��Ӧ����Ϊ���FROM���е�������

`WindowsSubstitution`�������Ĺ���������`CTESubstitution`��ֻ���﷨������CTE��ͬ��

> Substitution���Ĺ����Ҷ�LogicalPlan�Ľṹ���ı䣬��Resolution�Ĺ���ֻ�ǽ�ԭ�еĽڵ����Ϊʵ�壨��Ϊ�﷨������ı���������һ�����֣���û����������DataSet������ϵ����

###Resolution

��Щ����������ģ������˾ͺܿ��ܵ��³�����߶�ʱ�������в��ꡣ

1. ���ȵ�һ��������`ResolveTableValuedFunctions`����������`range(start, end, step)`����䡣�䴦�����̾�����ƥ�亯��������ʱֻ��range����ƥ�䵽֮��Ϳ���֪�������������͡�
Ȼ��֮ǰ�����ĸ�����������ʽ������������ǿ��ת��ΪexpectedType��

2. �ڶ���������`ResolveRelations`������˼�壬���ǽ�relation��״̬����Ϊresolved������Ϊresoleved�Ĺ�����ʵ���ǽ�������Ϊ�����DataSetʵ�塣

3. ������������`ResolveReferences`���佫UnresolvedAttribute����Ϊ������������о������Ե����ã���AttributeReference����AttributeReference����̳���Unevaluable�����Բ�δ��ֵ��
��ô������ô���������;���������ϵ�������أ���Ϊ����ÿ��DataSet��ʱ���ע��ܶ����ԣ���������ǰ����������ݵ�ʵ�塣���Իᵽע��ı���ȥ��ѯƥ�䣬�Ӷ���UnresolvedAttribute�е����������ӦAttribute��ӳ�䡣
����ʵ����`resolveAsTableColumn`�¡�

	//LogicalPlan
	protected def resolve(
		  nameParts: Seq[String],
		  input: Seq[Attribute],
		  resolver: Resolver): Option[NamedExpression] = {

		var candidates: Seq[(Attribute, List[String])] = {
		  // If the name has 2 or more parts, try to resolve it as `table.column` first.
		  if (nameParts.length > 1) {
			input.flatMap { option =>
			  resolveAsTableColumn(nameParts, resolver, option)
			}
		  } else {
			Seq.empty
		  }
		}

		if (candidates.isEmpty) {
		  candidates = input.flatMap { candidate =>
			resolveAsColumn(nameParts, resolver, candidate)
		  }
		}

		def name = UnresolvedAttribute(nameParts).name

		candidates.distinct match {
		  case Seq((a, Nil)) => Some(a)

		  case Seq((a, nestedFields)) =>
			
			val fieldExprs = nestedFields.foldLeft(a: Expression)((expr, fieldName) =>
			  ExtractValue(expr, Literal(fieldName), resolver))
			Some(Alias(fieldExprs, nestedFields.last)())

		  ...
		}
	}
	private def resolveAsTableColumn(
		  nameParts: Seq[String],
		  resolver: Resolver,
		  attribute: Attribute): Option[(Attribute, List[String])] = {
		assert(nameParts.length > 1)
		if (attribute.qualifier.exists(resolver(_, nameParts.head))) {
		  // At least one qualifier matches. See if remaining parts match.
		  val remainingParts = nameParts.tail
		  resolveAsColumn(remainingParts, resolver, attribute)
		} else {
		  None
		}
	}
	
��������е�`nameParts`��ʾ��ѯ����е���������Ϊʲô�����������أ���Ϊ��`tableName.colName.fieldName`����ʽ��
`attribute`��ʾ���ܵ����ԣ���Ϊ���Ȳ�֪����ʱ��ֻ�ܸ����ͷ�����֣�����tableName��һ����ȥ�Ų顣Ȼ�����`resolveAsColumn`ȥ��֤�Ƿ��`attribute`��������colName�Ƿ�ƥ�䣬����ƥ�䣬�ͷ��ظ�`attribute`����Ҫ�������ֶΣ���fieldName����ӳ�䡣
��Ȼ���н���������������ⲻ��˵��������Բ����ڣ���Ϊ��һ�������`tableName.colName...`����ʽ��Ȼ�����ֱ����`colName...`��ʽƥ�䡣

������û�������ֶΣ�������Ƕ������������ֱ�ӷ��ظ�DataSet��Attrubute����֮�򽫸��������;���ĳ�Ա��������ȡ������`tableName.colName.fieldName`����ϵ��������ôʹ��`tableName.colName.fieldName`���൱��ֱ�ӻ�ȡ`colName`�������ĳ�Ա������

�����Ĺ������ƣ����ǽ�Unresolved��LogicalPlan�ڵ�������������ʵ����ж�Ӧ����ͼ��ʾ��

[unresolved-to-resolved][generateLogicalPlan]

***

> `execute`������RuleExecutor�У���Ҫ���þ�������Analyzer�еĹ��򼯺�Batches������LogicalPlan������ʽ��[Spark SQL ����֪ʶ][1]�����ᵽ�����ǲ���Ӧ�ù���ﵽFixed Point���������ò�������֤����ʱ�����ڣ���



[1]:https://github.com/summerDG/spark-code-ananlysis/blob/master/analysis/sql/spark_sql_preparation.md
[2]:http://blog.csdn.net/dc_726/article/details/45399371
[3]:http://codemany.com/tags/antlr/
[analysis]:../../pic/Catalyst-analysis.png
[expresion-logical]:../../pic/Expression_LogicalPlan.png
[generateLogicalPlan]:../../pic/generate_LogicalPlan.png
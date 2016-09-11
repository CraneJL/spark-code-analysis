#Catalyst Logical Plan Optimizer

����Resolved LogicalPlan֮��Ĺ�������ִ��Logical Plan Optimize���ý׶ε���Ҫ������Ƕ�Logical Plan���м�֦�ϲ��Ȳ�����ɾ�����ü�����߶�һЩ����Ķ��������кϲ�����[Spark Catalyst �����׶�][1]��Ҳ�ж�LogicalPlan�ṹ�ĸı䣬������ֻ�ǽ����LogicalPlan�ϲ���һ���������ǽ���һ��LogicalPlan�Ļ����Ͻ��нṹ�仯��

[logicalPlan optimizer][logicalPlan_optimizer] 

����Optimizer���Ż�����RBO��Rule Based Optimizer��/CBO(Cost Based Optimizer)������Spark Catalyst������RBO��������һЩ�������Rule����Logical Plan���﷨�ṹ�����Ż���������Physical Planʱ�򣬻������Cost��������һ�����Ż���������join������ѡ��С�����join���Լ��������ݴ�С����HashJoin/SortMergeJoin/BroadcastJoin����֮����о���

�Ż����������ڱ�����֮��Ż�ִ�У����Դ�DataSet��һ��Action�������֣������ѡ`collect`��Ȼ�����`QueryExecution.executedPlan`��
QueryExecution���ж��Plan��ҪLazyִ�У�����������`analyzed`����`withCachedData`��Ȼ������`withCachedData`����`optimizedPlan`����֮ǰ�Ĳ���������LogicalPlan��ִ�еģ��������ɵ�Ҳ����LogicalPlan��Ȼ���������`sparkPlan`����PhysicalPlan���̶���`executedPlan`��

##withCachedData

�ڽ���LogicalPlan Optimizer֮ǰ��������Ҫ����`withCachedData`�������þ��ǽ�LogicalPlan�е�ĳЩ�����滻���Ѿ�cached���Ӷ����ٲ���Ҫ�ķ������㡣����˵��Ҳ��Optimizer��һ���֡�

	//CacheManager
	def useCachedData(plan: LogicalPlan): LogicalPlan = {
		plan transformDown {
		  case currentFragment =>
			lookupCachedData(currentFragment)
			  .map(_.cachedRepresentation.withOutput(currentFragment.output))
			  .getOrElse(currentFragment)
		}
	}
	
�������ǰ���ǰ�������`transformDown`��ǰ�������ÿ���ڵ����й���`transformUp`�Ǻ��������ÿ���ڵ����й�����һ��ѯ����LogicalPlan�ڵ�Ľ���Ƿ��Ѿ���cached��������ֱ����cached��Ľ���滻��

	//CacheManager
	def lookupCachedData(plan: LogicalPlan): Option[CachedData] = readLock {
		cachedData.find(cd => plan.sameResult(cd.plan))
	}
	case class CachedData(plan: LogicalPlan, cachedRepresentation: InMemoryRelation)

	//QueryPlan
	def sameResult(plan: PlanType): Boolean = {
		val left = this.canonicalized
		val right = plan.canonicalized
		left.getClass == right.getClass &&
		  left.children.size == right.children.size &&
		  left.cleanArgs == right.cleanArgs &&
		  (left.children, right.children).zipped.forall(_ sameResult _)
	}

�Ƚ���һ��InMemoryRelation����࣬�������һ���ڴ��е�Relation����Ϣ�������б��洢�㼶���Լ���������ӦRDD����RDD�����ݵ�������CatchedBatch��������Ͱ����������ݵ��кţ��Լ�����Buffer��������Array[Array[Byte]]������һ��InternalRow�ı���statsָ��ÿ����¼�еĸ�����Ԫ��ʲô���ͣ���Ϊ������Array[Byte]���洢ÿ�����ݣ�����Ҫָ���������ͣ��Ӷ�������������ȷInternalRow��û��ʲô���ݣ�������������ָ���������͡�
`cachedData`����ΪArrayBuffer[CacheData]���൱��һ��LogicalPlan��InMemoryRelation��ӳ�䡣��CacheManager�е�`cacheQuery`�п��Է��֣������LogicalPlan����QueryExecution�е�`analyzed`��LogicalPlan��Ҳ����˵û�о����Ż�����ġ�*�����Ĵ���Ҳ�ǱȽϺ���ģ���Ϊ�����Ż���LogicalPlan���ܸ��������ѯ�Ĳ�������¼�ʹ��ͬ�Ĳ�ѯҲ����ƥ�䣬����Ż�����Ҫ����ʱ��ģ�ÿ���Ӳ�ѯ���Ż��ֶβ������������Ż�������ȷ���ģ�����������ܴ�Ŀ���*��
LogicalPlan��`canonicalized`��̬���ǽ��Ӳ�ѯ�ı���ȥ���������Ļ�������Ϊ����LogicalPlan������Ϊ�Ӳ�ѯ������ͬ���жϲ���ͬ��Ȼ�����һ�αȽ������Ϣ��

��������������`_.cachedRepresentation.withOutput(currentFragment.output)`����滻ԭ��LogicalPlan����`currentFragment`�������

	//InMemoryRelation
	def withOutput(newOutput: Seq[Attribute]): InMemoryRelation = {
		InMemoryRelation(
		  newOutput, useCompression, batchSize, storageLevel, child, tableName)(
			_cachedColumnBuffers, batchStats)
	}

֮ǰ�����½��ܹ�ÿ��LogicalPlan�������һ��Attribute����������һ���µ�Relation������Attribute�������滻������Ҫǿ��һ�£�InMemoryRelation����Ҳ��һ��LogicalPlan�����࣬�����滻���̾���������ˡ�

##optimizedPlan

	//QueryExecution
	lazy val optimizedPlan: LogicalPlan = sparkSession.sessionState.optimizer.execute(withCachedData)

	//SessionState
	lazy val optimizer: Optimizer = new SparkOptimizer(catalog, conf, experimentalMethods)

	//SparkOptimizer
	class SparkOptimizer(
		catalog: SessionCatalog,
		conf: SQLConf,
		experimentalMethods: ExperimentalMethods)
	  extends Optimizer(catalog, conf) {

	  override def batches: Seq[Batch] = super.batches :+
		Batch("Optimize Metadata Only Query", Once, OptimizeMetadataOnlyQuery(catalog, conf)) :+
		Batch("Extract Python UDF from Aggregate", Once, ExtractPythonUDFFromAggregate) :+
		Batch("User Provided Optimizers", fixedPoint, experimentalMethods.extraOptimizations: _*)
	}

���Է����Ż�������˸����֮���Ϊ3����Ҫ���֣�
1. ����ֻ�跢��partition�����Ԫ���ݾͿ��Իش�Ĳ�ѯ��Ӧ���ڣ������б�ɨ���columns����partition columns������GROUP BY ���Ӻ����column����Ϊ���������ҪPartition������ѯ���ʽ��������������
	* �ۺϱ��ʽ�������partition columns������SELECT col FROM tbl GROUP BY col��
	* ��partition columns�ϵľۺϲ���������DISTINCT������SELECT col1, count(DISTINCT col2) FROM tbl GROUP BY col1��
	* ��partition columns�ϵľۺϲ�����`Max`��`Min`��`First`��`Last`����Ϊ���ܰ�������DISTINCT��������һ��������SELECT col1, Max(col2) FROM tbl GROUP BY col1��
2. ����Python�������ġ�
3. �û��ṩ���Ż���������ʵ����SparkSQL�ڲ����ṩһ���ӿڣ�ExperimentalMethods�������������`extraStrategies`��`extraOptimizations`��������������ȡ�����ò��Ժ��Ż�����

��һ��ķ������ǰ����Partitioned���洢�Ķ���Ԫ�����ļ�����HDFS��Ԫ�����ļ�����CataLog����Ԫ���ݱ��Ľڵ�����ֱ������Ӧ��partition values�滻Ϊ���ݿ�֧�ֵ����͡���������������ת������������Ĳ����ö�������ÿ��Partition��Ԫ������Ϣ�ش�

һ�������Ż���Optimizer��������棬����������е�`super.batches`�����`batches`�������Ż��漰���ܶ���ࡣ

���ȡ�Finish Analysis�����������Ż����룬��������Analyzer�ķ��룬��֤�˽��һ���ԣ���ȷ�ԣ����罫���е�CurrentData����ͬ�����������й淶�����������Ӳ�ѯ����ȥ������

###�Ż�����

	 Batch("Union", Once,
		  CombineUnions) ::
		Batch("Subquery", Once,
		  OptimizeSubqueries) ::
		Batch("Replace Operators", fixedPoint,
		  ReplaceIntersectWithSemiJoin,
		  ReplaceExceptWithAntiJoin,
		  ReplaceDistinctWithAggregate) ::
		Batch("Aggregate", fixedPoint,
		  RemoveLiteralFromGroupExpressions,
		  RemoveRepetitionFromGroupExpressions) ::
		Batch("Operator Optimizations", fixedPoint,
		  ...) ::
		Batch("Decimal Optimizations", fixedPoint,
		  DecimalAggregates) ::
		Batch("Typed Filter Optimization", fixedPoint,
		  CombineTypedFilters) ::
		Batch("LocalRelation", fixedPoint,
		  ConvertToLocalRelation,
		  PropagateEmptyRelation) ::
		Batch("OptimizeCodegen", Once,
		  OptimizeCodegen(conf)) ::
		Batch("RewriteSubquery", Once,
		  RewritePredicateSubquery,
		  CollapseProject) :: Nil
	
���Կ����ܶ��Ż�������ɵ�Batch��[Spark-Catalyst Optimizer][2]�н����˺ܴ�һ���֡������޸Ĳ����Ż�������ܣ����Ҳ�����������

1. OptimizeIn���������㣺
	* ����IN���������е���ͬԪ�أ����罫In (value, seq[Literal])תΪIn (value, ExpressionSet(seq[Literal]))
	* ��������ظ�������е�Ԫ����Ŀ����`conf.optimizerInSetConversionThreshold`��Ĭ��10����ת��ΪINSET���������Զ���ExpressionSetת��ΪHashset�����IN���������ܡ�
2. ColumnPruning�漰�����߼��ܶ࣬������Ҫ��Ϊ��
	* ���Ӳ�ѯ�е�Project/Aggregate/Expand������������Project�����в����������ԣ���ȥ���ಿ��
	* ���Ӳ����а���DeserializeToObject/Aggregate/Expand/Generate��û�е����ԣ���ȥ���ಿ��
	* ����Generate�����������������������Generate����У��������յ�Project�����Լ�����Generate������Ե��Ӽ�����ô˵�����ն�û���õ�Generate�������ǲ���������������Generate������������������ӡ�
	* ����Left-Semi-Join�����ұߵ��Ӳ�ѯ�������޹�����ȥ������ֻ����������Join������
	* ����Project(_, child)������ֻ����child����Project�йص����ԣ��������Ͳ�ͬ�������������������ֻ��child
3. CombineTypedFilters��CombineFilters�Ĳ������ƣ���������TypedFilter����һ�������Filter�����������������Լ����壬ֻҪ��һ������δ���룬����boolean�ĺ������ɣ��������ϲ�������������
4. CombineUnions��Զ��union�������кϲ������磺(select a from tb1) union (select b from tb2) union (select c from tb3) => select a,b,c from union(tb1,tb2,tb3)
5. OptimizeSubqueries��������Ӳ�ѯ��Optimizer���Ż�����
6. RemoveLiteralFromGroupExpressions�ǽ�����Group by�еı��ʽ�еĳ����Ƴ�������Ϊ�䲻��Ӱ����
7. RemoveRepetitionFromGroupExpressions�ǽ�Group by�еı��ʽ����ͬ�ı��ʽ�Ƴ���
8. PushProjectionThroughUnion���ڽ�Project�����ƶ���ÿ����Union���ʽ���Ӷ�������С��Χ
9. ReorderJoin�������е��������ʽ���䵽join�������У�ʹÿ������������һ���������ʽ���������˳����������˳�����磺select * from tb1,tb2,tb3 where tb1.a=tb3.c and tb3.b=tb2.b����ôjoin��˳�����join(tb1,tb3,tb1.a=tb3.c)��join(tb3,tb2,tb3.b=tb2.b)
10. EliminateOuterJoin�����������Ǿ�����fullOuterJoinת��Ϊleft/right outer join��������inner join��**null-unsupplyingν�ʱ�ʾ�������null��Ϊ�����򷵻�false��null��Ҳ����˵��֧�־�����ν�ʹ��˵Ľ���л�����null**������5�������
	* ����full outer��������������ν����null-unsupplying��full outer -> left outer����֮������ұ�������ν�ʣ�full outer -> right outer�������߶��У�full outer -> inner
	* ����left outer�����ұ���������null-unsupplying��left outer -> inner
	* ����right outer���������������null-unsupplying��right outer -> inner
11. InferFiltersFromConstraints�ǽ��ܵ�Filter�����������Լ�������Լ��ɾ������ֻ֧��inter join��leftsemi join�����磺select * from (select * from tb1 where a) as x, (select * from tb2 where b) as y where a, b ==> select * from (select * from tb1 where a) as x, (select * from tb2 where b) as y
12. FoldablePropagation���ȶ��б����Ŀ��۵����ʽ�е����Ժ����������������Ȼ��LogicalPlan�е����и����ԵĽڵ������������
13. ConstantFolding�������Ծ�ֱ̬����ֵ�ı��ʽֱ�����������Add(1,2)==>3
14. ReorderAssociativeOperator������������صĿ�ȷ���ı��ʽֱ�Ӽ�����䲿�ֽ�������磺Add(Add(1,a),2)==>Add(a,3)����ConstantFolding��ͬ���ǣ������Add��Multiply�ǲ�ȷ���ģ����ǿ��Ծ����ܼ�������ֽ��
15. RemoveDispensableExpressions������������ȥ��û��Ҫ�Ľڵ㣬�������ࣺUnaryPositive��PromotePrecision�����߽����Ƕ��ӱ��ʽ�ķ�װ�����޶������
16. EliminateSorts��������ȷsort byĬ��ֻ��֤partition����ֻ��������ȫ�����������²ű�֤ȫ�����򣻸ù���������û�������õ��������ӣ���Ϊ�������ӿ�����ȷ���ģ��﷨��ȷ���߼������⣩����ô�������������Ӿ�û�����壬���磺select a from tb1 sort by 1+1 ==> select a from tb1
17. RewriteCorrelatedScalarSubquery��ScalarSubqueryָ����ֻ����һ��Ԫ�أ�һ��һ�У��Ĳ�ѯ�����Filter��Project��Aggregate�����а�����Ӧ��ScalarSubquery������д֮��˼�������ΪScalarSubquery�����С�����Թ��˴󲿷�����Ԫ�أ���������ʹ��left semi join���ˣ�
	* Aggregate����������select max(a), b from tb1 group by max(a) ==> select max(a), b from tb1 left semi join max(a) group by max(a)����������join��Ч��Ҫ������group by����Ч�ʸ�
	* Project����������select max(a), b from tb1 ==> select max(a), b from tb1 left semi join max(a)
	* Filter������select b from tb1 where max(a) ==> select b (select * from tb1 left semi join max(a) where max(a))
18. EliminateSerialization���ܶ������Ҫ�����л���Ȼ�����л�����������л���������ͺ����л�ǰ������������ͬ����ʡ�����л��ͷ����л��Ĳ���
19. RemoveAliasOnlyProject�������������Ӳ�ѯ������ı������ɵ�Project�����磺select a from (select a from t) ==> select a from t
20. OptimizeCodegen�������ض����������һ���ֲ�ѯ��������Ĵ��룬������ͨ��һ�������ӽ��в������ο�[Apache Spark as a Compiler: Joining a Billion Rows per Second on a Laptop][3]
21. RewritePredicateSubquery����EXISTS/NOT EXISTS��Ϊleft semi/anti join��ʽ����IN/NOT INҲ��Ϊleft semi/anti join��ʽ
22. ConvertToLocalRelation����Project�е�ÿ���������ݽ����ı��ʽӦ����LocalRelation�е���ӦԪ�أ�����select a + 1 from tb1תΪһ��������Ϊ��a+1����relation������tb1.a��ÿ��ֵ���Ǽӹ�1��
23. PropagateEmptyRelation������пձ��LogicalPlan���б任
	* ���Union���Ӳ�ѯ���ǿգ�����ҲΪ�գ�
	* ���Join���Ӳ�ѯ���ڿձ�
		- `Inner => empty(p)`��
		- `LeftOuter | LeftSemi | LeftAnti if isEmptyLocalRelation(p.left) => empty(p)`��
		- `RightOuter if isEmptyLocalRelation(p.right) => empty(p)`��
		- `FullOuter if p.children.forall(isEmptyLocalRelation) => empty(p)`
	* ���ڵ��ڵ�Plan�������ӽڵ�Ϊ�ձ�������Ϊ�ձ�
24. DecimalAggregatesӦ�������ڼ��ٸ���������ģ���Ϊ����������Ҫ���ƾ��ȣ�������Ҫͨ�����䳤������֤���ȣ�������һ����Χ�ڿ��Բ����ƣ�������Ҫÿһ���ĸ������㶼���ƾ���

[1]:https://github.com/summerDG/spark-code-ananlysis/blob/master/analysis/sql/spark_sql_parser.md
[2]:https://github.com/ColZer/DigAndBuried/blob/master/spark/spark-catalyst-optimizer.md
[3]:https://databricks.com/blog/2016/05/23/apache-spark-as-a-compiler-joining-a-billion-rows-per-second-on-a-laptop.html
[logicalPlan_optimizer]:../../pic/Optimizer-diagram.png
# Spark SQL Physical Plan

���Ľ���Spark SQL��PhysicalPlan�����ɣ���һ������Ҫ�ǻ���CBO(Cost Based Optimizer)���Ż���

![generate physical plan][physical-plan]

	lazy val sparkPlan: SparkPlan = {
		SparkSession.setActiveSession(sparkSession)
		// TODO: We use next(), i.e. take the first plan returned by the planner, here for now,
		//       but we will implement to choose the best plan.
		planner.plan(ReturnAnswer(optimizedPlan)).next()
	}
	
`SparkSession.setActiveSession(sparkSession)`��Ҫ����������SparkSession����Ϊ��ʱ��SparkSession�Ѿ�����֮ǰ�Ĳ����������˸ı䣬����ȷ����ͬ�߳̿��Ի������µ�SparkSession�������ǳ�ʼ���������Ǹ���
Ȼ�����`ReturnAnswer(optimizedPlan)`��ReturnAnswer��ʵһ�������LogicalPlan�ڵ㣬�䱻���뵽�Ż����logical plan�Ķ��㡣
`planner`���ڽ�LogicalPlanת��ΪphysicalPlan��

	//QueryPlanner
	def plan(plan: LogicalPlan): Iterator[PhysicalPlan] = {		
		val candidates = strategies.iterator.flatMap(_(plan))
		val plans = candidates.flatMap { candidate =>
		  val placeholders = collectPlaceholders(candidate)

		  if (placeholders.isEmpty) {
			Iterator(candidate)
		  } else {
			placeholders.iterator.foldLeft(Iterator(candidate)) {
			  case (candidatesWithPlaceholders, (placeholder, logicalPlan)) =>
				val childPlans = this.plan(logicalPlan)
				candidatesWithPlaceholders.flatMap { candidateWithPlaceholders =>
				  childPlans.map { childPlan =>
					candidateWithPlaceholders.transformUp {
					  case p if p == placeholder => childPlan
					}
				  }
				}
			}
		  }
		}

		val pruned = prunePlans(plans)
		assert(pruned.hasNext, s"No plan for $plan")
		pruned
	}

��������Ż����LogicalPl���ɺ�ѡPhysicalPlan��������������Ĳ����������ɵ�PhysicalPlan��������ĺ�ѡPhysicalPl��Ե��������LogicalPlan��ת���������Ӳ�ѯ��LogicalPlan��������ʱ���ΪPlanLater��
Ȼ������`this.plan(logicalPlan)`�����ΪPlanLater���Ӳ�ѯ�ݹ�ת��ΪPhysicalPlan������滻����Ӧλ�õ�`placeholder`�����Կ��Է���ת�����ɶ����µġ�����ÿ��ת���������ɺܶ�PhysicalPlan��
�����ܵ�PhysicalPlan��ָ�������ӵģ�Խ�Ǹ��ӵ�SQL��䣬�����Ŀ��ܵ�PhysicalPlan������Խ��ġ���������������������һ��`prunePlans`���ڼ�ȥ�����ʵ�plan���Ӷ�������ϱ�ը���������ڻ�û��ʵ�֡�

���Է��ִ�������ʱʵ�ֵĽ�����ѡ����ЩPhysicalPlan�ĵ�һ������û�н��л���cost��ѡ��Ӧ�ò���֮��ͻ��Ϊѡ�����ŵ�plan�ˡ�

## ����PhysicalPlan�Ĳ��ԣ�Strategies��

	//SparkPlanner
	def strategies: Seq[Strategy] =
		  extraStrategies ++ (
		  FileSourceStrategy ::
		  DataSourceStrategy ::
		  DDLStrategy ::
		  SpecialLimits ::
		  Aggregation ::
		  JoinSelection ::
		  InMemoryScans ::
		  BasicOperators :: Nil)
		  
`extraStrategies`����`experimentalMethods.extraStrategies`�ṩ��[Catalyst Logical Plan Optimizer][1]�н��ܹ����ࡣDDLStrategy���DDL��䣬DataSourceStrategy��FileSourceStrategy���ƣ�ֻ����Ե�����Դ��JDBC���ⲿ���ݿ�����ݣ���HadoopFsRelation�������ﲻ��������������������������һһ�����������ԡ�

### FileSourceStrategy

�ò��Ի�ɨ���ļ����ϣ���Ϊ���ǿ����ǰ�������ָ�����н���partition�����ո������Ի��֣��ļ����Ƕ�Ӧֵ����bucket��bucket����һ�ֻ��ַ�ʽ�����ո��������л��ֳɶ��bucket���ļ�����bucket��ţ�������ͨ�������˽⵽������Խ�����HadoopFsRelation��������ܽ��ڿ���������֮�����������
> �ò��Կ��ܷ����ļ����׶�Ϊ��

> 1. ����filters�����ڷֱ���ֵ����Ϊ��ͬ��filter���������ڲ�ͬ�����ݣ�
> 2. �������е�ͶӰ��Ҫ��������������schema���м�֦���ü�֦�ֽ����ڶ���column��
> 3. ͨ����filters��schema���ݵ�FileFormat��������reader������
> 4. ʹ�÷�Ƭ��֦ν��ö����Ҫ��ȡ���ļ���
> 5. ���ӱ�����ɨ��֮����ֵ��projection����filters

> ����һ���㷨���ļ����������

> 1. �����ǡ�bucketed��������bucket id���ļ���֯����ȷ��Ŀ��Partitions��
> 2. �����ǡ�bucketed����bucketing�����ر��ˣ�
	* ����ļ��ܴ󣬳�������ֵ��������ڸ���ֵ�ֳɶ�Ƭ��
	* �����ļ���С�������У�
	* ������õ��ļ����պ�����㷨װ��buckets��������в����ڼ�����һ���ļ�֮��û�г�����ֵ����ô����ӣ���֮�������µ�bucket�����֮��

��������PhysicalOperation����Ϊ�ò���������Ҫ���������LogicalPlan����ȡȷ����project��filter������

	//patterns.scala
	private def collectProjectsAndFilters(plan: LogicalPlan):
		  (Option[Seq[NamedExpression]], Seq[Expression], LogicalPlan, Map[Attribute, Expression]) =
		plan match {
		  case Project(fields, child) if fields.forall(_.deterministic) =>
			val (_, filters, other, aliases) = collectProjectsAndFilters(child)
			val substitutedFields = fields.map(substitute(aliases)).asInstanceOf[Seq[NamedExpression]]
			(Some(substitutedFields), filters, other, collectAliases(substitutedFields))

		  case Filter(condition, child) if condition.deterministic =>
			val (fields, filters, other, aliases) = collectProjectsAndFilters(child)
			val substitutedCondition = substitute(aliases)(condition)
			(fields, filters ++ splitConjunctivePredicates(substitutedCondition), other, aliases)

		  case BroadcastHint(child) =>
			collectProjectsAndFilters(child)

		  case other =>
			(None, Nil, other, Map.empty)
		}
		
�÷��������ҵ�����ȷ����Project��Filter��Expression.deterministic��ʾ������̶���ʱ�����Ҳ��ȷ���ģ�������Ϊ������ж��ı䡣����������ƥ������Project���ʽ��ȷ����ͶӰ������
����ͨ���������滻�ѱ�����Ϊ��ԭʼ��Expression��Filter�Ĳ������ƣ�ֻ����Ե����������ʽ������ͶӰ���ʽ�����Ҷ��ߴ����������𣬶����������ʽ���滻֮����������ʽ���������������ʽҪ�ϲ�������
��ͶӰ���ʽ��ֱ���滻����Ϊ������ͶӰ���ʽֻ���ܻ�ȸ��ڵ�Ķ࣬���Բ����������Ľ����

�÷���ִ����֮����з��ء�ȷ���ġ�ͶӰ���ʽ��Filter���ʽ���Լ������LogicalPlan��

	//FileSourceStrategy
	object FileSourceStrategy extends Strategy with Logging {
	  def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
		case PhysicalOperation(projects, filters,
		  l @ LogicalRelation(fsRelation: HadoopFsRelation, _, table)) =>

		  val filterSet = ExpressionSet(filters)

		  val normalizedFilters = filters.map { e =>
			e transform {
			  case a: AttributeReference =>
				a.withName(l.output.find(_.semanticEquals(a)).get.name)
			}
		  }

		  val partitionColumns =
			l.resolve(
			  fsRelation.partitionSchema, fsRelation.sparkSession.sessionState.analyzer.resolver)
		  val partitionSet = AttributeSet(partitionColumns)
		  val partitionKeyFilters =
			ExpressionSet(normalizedFilters.filter(_.references.subsetOf(partitionSet)))
		  logInfo(s"Pruning directories with: ${partitionKeyFilters.mkString(",")}")

		  val dataColumns =
			l.resolve(fsRelation.dataSchema, fsRelation.sparkSession.sessionState.analyzer.resolver)

		  // Partition keys are not available in the statistics of the files.
		  val dataFilters = normalizedFilters.filter(_.references.intersect(partitionSet).isEmpty)

		  // Predicates with both partition keys and attributes need to be evaluated after the scan.
		  val afterScanFilters = filterSet -- partitionKeyFilters
		  logInfo(s"Post-Scan Filters: ${afterScanFilters.mkString(",")}")

		  val filterAttributes = AttributeSet(afterScanFilters)
		  val requiredExpressions: Seq[NamedExpression] = filterAttributes.toSeq ++ projects
		  val requiredAttributes = AttributeSet(requiredExpressions)

		  val readDataColumns =
			dataColumns
			  .filter(requiredAttributes.contains)
			  .filterNot(partitionColumns.contains)
		  val outputSchema = readDataColumns.toStructType
		  logInfo(s"Output Data Schema: ${outputSchema.simpleString(5)}")

		  val pushedDownFilters = dataFilters.flatMap(DataSourceStrategy.translateFilter)
		  logInfo(s"Pushed Filters: ${pushedDownFilters.mkString(",")}")

		  val outputAttributes = readDataColumns ++ partitionColumns

		  val scan =
			new FileSourceScanExec(
			  fsRelation,
			  outputAttributes,
			  outputSchema,
			  partitionKeyFilters.toSeq,
			  pushedDownFilters,
			  table)

		  val afterScanFilter = afterScanFilters.toSeq.reduceOption(expressions.And)
		  val withFilter = afterScanFilter.map(execution.FilterExec(_, scan)).getOrElse(scan)
		  val withProjections = if (projects == withFilter.output) {
			withFilter
		  } else {
			execution.ProjectExec(projects, withFilter)
		  }

		  withProjections :: Nil

		case _ => Nil
	}

* `ExpressionSet`����ʵ�����ڹ��˵��淶������ʽ��ͬ�ı��ʽ������"a+1"��"1+a"�淶�������ʽ��һ���ģ�����������һ��ȥ����Ĳ�����`normalizedFilters`�������ǽ��������ʽ�е�������ͳһ���������������

* `partitionColumns`�ǽ�HadoopFsRelation��partition schema��������Щ����Partition������key���ԣ���resolver���룬�Ӷ���������HadoopFsRelation��partition���ԡ������и����ʣ���һ����Ӧ���ڽ�����Realoved LogicalPlan��ʱ���������ʵ������һ���Ĳ�����Щ��©����������ⲿϵͳ�ı�
��ʱֻ�ǲ�ѯ��Catalog��û�У���û�н�����Ӧ���Խ������ӣ���Ϊ������resolved״̬��`partitionSet`����Ҳ��ȥ���ظ������ԡ�

* `partitionKeyFilters`���ҵ��������ʽ��**ֻ**����partition���Եı��ʽ���ϡ������ڽ��й���ʱ����ֱ��ͨ��partition��Ž��У��Ӷ���߲�����Ч�ʡ�

* `dataColumns`��Ӧ���Ǵ洢���ݵ����ԣ���Ҫ�Ƿ�partition���ԣ��������partition���������ݣ���ôҲ�ᱣ����`dataFilters`����Щֻ������partition���Ե�filter��

* `afterScanFilters`��ʾ���Ǿ���ɨ���в���ȷ����Filter�����磺`partitionKeyFilters`ֻ��Ҫ�����keyֵ��Ӧ��partition���ɣ���������������뽫���б�ɨ�������ȷ����

* ����`requiredAttributes`����Ҫ�����ԣ�`readDataColumn`��������Ҫ��ѯ�����еķ�partition���ԡ�Ȼ�������Ӧ��Schema��Ϣ`outputSchema`��

* `pushedDownFilters `�ǽ�`dataFilter`�е�filterԭ�ȵĳ����滻��Scala��ԭ�����ͣ���ΪCatalyst�е����Ͳ��ܺ�����������ֻ�����ڱ�ʾ����

* ʵ����ɨ�����������Գ���`readDataColumn`����`partitionColumns`����`outputAttributes`��������Ҫǿ��`requiredAttributes`��`outputAttributes`��ͬ�����߰��������е�Partition column��ǰ��ֻ�������֡�

* Ȼ�󴴽�ɨ��HadoopFsRelation�����ݵ�PhysicalPlan�ڵ㣬`scan`��**��ô֮ǰ�Ĳ���˵���׶����������ˣ����ǹ���û��Ҫ��ѯ�������С�**

> ֮ǰ����һֱ�и����ʣ�`afterScanFilters`��`dataColumns`��ʲô��ϵ����ʵ������ǰ�ߵ��Ӽ������ߴ��ڵ�Ŀ����Ϊ֮���ɨ����С��Χ��ǰ��֮����Ҫ������������Ϊ��Щ������ֻ���ڵڶ���ɨ���ʱ���������ȷ����
���磺select * from tb1 where a < c and b < c and b = max(c)����a��partition key����ô��һ�鼴ʹ������b < c and b = max(c)Ҳֻ����С�˷�Χb = max(c)���������ϼ��㣬��Ϊa < c��֮�����ȷ�����������ں�������`dataFilter`����Щ������ֻ��ͳһ���¼��㣬��֤��ȷ�ԡ�

֮����ǽ�`afterScanFilters`��and����������Ȼ�������µ�PhysicalPlan�ڵ�`withFilter`�����ӽڵ�Ϊ`scan`�������`withFilter`�ϼ���һ��ͶӰ�ڵ㡣

> ���Է��ָò��Ե��������ܼ򵥣�ֻ��select ... from tb1 where ...���͵Ĳ�����û�ж�����Ӳ�ѯ��Aggregate�ȡ�

### SpecialLimits

	//SparkStrategies
	object SpecialLimits extends Strategy {
		override def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
		  case logical.ReturnAnswer(rootPlan) => rootPlan match {
			case logical.Limit(IntegerLiteral(limit), logical.Sort(order, true, child)) =>
			  execution.TakeOrderedAndProjectExec(limit, order, None, planLater(child)) :: Nil
			case logical.Limit(
				IntegerLiteral(limit),
				logical.Project(projectList, logical.Sort(order, true, child))) =>
			  execution.TakeOrderedAndProjectExec(
				limit, order, Some(projectList), planLater(child)) :: Nil
			case logical.Limit(IntegerLiteral(limit), child) =>
			  execution.CollectLimitExec(limit, planLater(child)) :: Nil
			case other => planLater(other) :: Nil
		  }
		  ...
		}
	}

ReturnAnswer������ڱ��Ŀ�ƪ���ܹ���ֻ�������װ��ʡ�Ե�������case�Ĵ�������ڸ�������С�

> �������ռ�һ�������Window function��֮ǰ�����ˣ����ﲹ�ϣ���Ϊ������漰��GlobalLimit�ĸ��Window�������ڲ�ѯ�������ִ�еĲ�����������ڹ�ȥ�ľۺϲ�����˵��
���ȸ�ϸ������Ч�ʸ��ߡ����磺SELECT Row_Number() OVER (partition by xx ORDER BY xxx desc) RowNumber��OVER�ؼ��ֺ���ľ�����һ��window function����Ϊ��������൱�����ÿ��partition��������
��Ȼ���Ҳ���Ը�window�����������ͻ��õ�WINDOW�ؼ��֣�������SELECT sum(salary) OVER w, avg(salary) OVER w FROM empsalary WINDOW w AS (PARTITION BY depname ORDER BY salary DESC);����window��������ϸ֪ʶ��[SQL Server�еĴ��ں���][2]��
GlobalLimit��ʵ����ָwindow�����е�windows�ӿ飬��ROWS��RANGE�ȡ�

	//logical.Limit
	def unapply(p: GlobalLimit): Option[(Expression, LogicalPlan)] = {
		p match {
		  case GlobalLimit(le1, LocalLimit(le2, child)) if le1 == le2 => Some((le1, child))
		  case _ => None
		}
	}
	
���Է��ָ�unapply�����Ƚ����⣬��Ϊ���������GlobalLimit���Ӳ�ѯ��LocalLimit�����������������Limit��limit���ʽ��һ����˵�������ࡣcase 1�л���һ��������LocalLimit�����Ķ�����Sort���ģ������⼸��ſ���ִ��`execution.TakeOrderedAndProjectExec`��
�ú�����������ʵ���ǰ���LocalLimit�Ӳ�ѯ��˳������ȡǰk��Limit�������������¼������ʵ��һ���Ż����������GlobalLimit��LocalLimit�Ļ�ȡ�ļ�¼������ͬ����ΪGlobalLimit�Ĳ�������LocalLimit�Ļ�����ִ�еģ���ô����window������ô������ʵ���ǲ���Ӱ�쵽���յĽ����
���磺

	select count(*) over (sort by b rows 100) from (select * from tb1 sort by c) limit 100 
	==> select count(*) from (select * from tb1 sort by c) limit 100

case 2��������ǣ�LocalLimit���Ӳ�ѯ�����������������ͶӰ���������ú���ͬcase 1��ֻ���޶�����������ԣ�����û�д�������������ﶼ����`planLater`���б�ǡ�

case 3����GlobalLimit���Ӳ�ѯ�������������������ֱ������CollectLimitExec PhysicalPlan�ڵ㣬ע�������TakeOrderedAndProjectExecҲ��PhysicalPlan�ڵ㡣

### Aggregation

	//SparkStrategies
	object Aggregation extends Strategy {
		def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
		  case PhysicalAggregation(
			  groupingExpressions, aggregateExpressions, resultExpressions, child) =>

			val (functionsWithDistinct, functionsWithoutDistinct) =
			  aggregateExpressions.partition(_.isDistinct)
			...

			val aggregateOperator =
			  if (functionsWithDistinct.isEmpty) {
				aggregate.AggUtils.planAggregateWithoutDistinct(
				  groupingExpressions,
				  aggregateExpressions,
				  resultExpressions,
				  planLater(child))
			  } else {
				...
				aggregate.AggUtils.planAggregateWithOneDistinct(
				  groupingExpressions,
				  functionsWithDistinct,
				  functionsWithoutDistinct,
				  resultExpressions,
				  planLater(child))
			  }

			aggregateOperator

		  case _ => Nil
		}
	}

��������PhysicalAggregation������LogicalAggregate��ͬ�ĵط����ڣ�

1. ������grouping���ʽ�������Ӷ�ʹ��Щ���ʽ�����ھۺϽ׶α����ã�
2. ���ֶ�ε�AggregationҪȥ�أ�
3. ����aggregation��������ļ��������ս���ֿ��ġ����磺��count + 1���еġ�count����AggregateExpression���������ս���ǡ�count.resultAttribute + 1����

	//PhysicalAggregation
	type ReturnType =
		(Seq[NamedExpression], Seq[AggregateExpression], Seq[NamedExpression], LogicalPlan)

	  def unapply(a: Any): Option[ReturnType] = a match {
		case logical.Aggregate(groupingExpressions, resultExpressions, child) =>

		  val aggregateExpressions = resultExpressions.flatMap { expr =>
			expr.collect {
			  case agg: AggregateExpression => agg
			}
		  }.distinct

		  val namedGroupingExpressions = groupingExpressions.map {
			case ne: NamedExpression => ne -> ne
			case other =>
			  val withAlias = Alias(other, other.toString)()
			  other -> withAlias
		  }
		  val groupExpressionMap = namedGroupingExpressions.toMap

		  val rewrittenResultExpressions = resultExpressions.map { expr =>
			expr.transformDown {
			  case ae: AggregateExpression =>
				ae.resultAttribute
			  case expression =>

				groupExpressionMap.collectFirst {
				  case (expr, ne) if expr semanticEquals expression => ne.toAttribute
				}.getOrElse(expression)
			}.asInstanceOf[NamedExpression]
		  }

		  Some((
			namedGroupingExpressions.map(_._2),
			aggregateExpressions,
			rewrittenResultExpressions,
			child))

		case _ => None
	}
	
��һ���������ռ���ͬ�ľۺϲ���������select sum(*)+count(*),count(*),count(*)+1 from tb1 group by a��ʵ����ֻ���������ۺϲ�����sum(*)��count(*)��

�ڶ����Ƕ�grouping���ʽ�з�NamedExpression��������������������γ�Map[Expression,NamedExpression]ӳ�䡣

ԭʼ��`resultExpressions`��һ����������˾ۺϺ�����grouping columnֵ�ͳ����ı��ʽ�ļ��ϣ����ۺ���������󣬻�����`resultExpressions`����ͶӰ�����
����������д������ʽ����`rewrittenResultExpressions`���Ӷ�ʹ�����������ͶӰ��������ƥ�䡣���ǰ�`resultExpressions`��Ԫ�����ͻ���Attribute��
case 2�ǵ����ʽ����AggregateExpressionʱ��Ҫ��grouping expression�еı��ʽ����ͳһ��������ͬ��Ϊ��ͬ�����磺a+1��1+a������ͬ��a+b-c��a-c+b��ͬ�����Լ��ټ��㡣

�ص�Aggregation�����ռ����Ĳ�ͬ��AggregateExpression����Ϊ�����֣�����DISTINCT�ؼ��ֵĺͲ������ġ���Ȼ����DISTINCT��AggregateExpression�������ܳ���1����
����֮��Ĳ���������԰���һ��DISTINCT��aggregation����һ�����͵�PhysicalPlan�ڵ㣬��Բ�����DISTINCT��aggregation������һ��PhysicalPlan�ڵ㡣

������DISTINCT��aggregationʵ���������жϾۺϺ����ʺ�������PhysicalPlan��HashAggregateExec����SortAggregateExec���ж����ݾ�������ۺϺ����ķ������Ͷ��ǿɱ�ģ��������ͣ������������ͣ����ǿɱ�ģ����Ҷ���supportPartialAggregate�ģ���֧��ֻ��3�������Collect��Hive UDAF��Window function�еľۺϺ���������ô����HashAggregateExec��
��֮����SortAggregateExec��ԭ������û�и㶮��

���ں���һ��DISTINCT��aggregation������Ҳ�ǰ�����DISTINCT�Ĳ�������һ��PhysicalPlan�ڵ㣬��һ���н�DISTINCT����һ���̶���ת��Ϊ��Group By������

	//AggUtils.planAggregateWithOneDistinct
	val partialAggregate: SparkPlan = {
		  val aggregateExpressions = functionsWithoutDistinct.map(_.copy(mode = Partial))
		  val aggregateAttributes = aggregateExpressions.map(_.resultAttribute)
		  createAggregateExec(
			requiredChildDistributionExpressions =
			  Some(groupingAttributes ++ distinctAttributes),
			groupingExpressions = groupingExpressions ++ namedDistinctExpressions,
			aggregateExpressions = aggregateExpressions,
			aggregateAttributes = aggregateAttributes,
			initialInputBufferOffset = (groupingAttributes ++ distinctAttributes).length,
			resultExpressions = groupingAttributes ++ distinctAttributes ++
			  aggregateExpressions.flatMap(_.aggregateFunction.inputAggBufferAttributes),
			child = child)
	}
	
����`distinctAttributes`��ʾDISTINCT���������ԣ�����`Max(DISTINCT a)`�е�a�����Է���`requiredChildDistributionExpressions`��`groupingExpressions`���ֱ������DISTINCT���õ����ԣ����ʽ�������Կ���˵DISTINCT������ת��Ϊ��Group By������
Ȼ���ҵ�����DISTINCT�ľۺϺ����������������е�`Max`�����ɶ�Ӧ��Expression��Attribute�����������µ�PhysicalPlan�ڵ㣬���ӽڵ����֮ǰ���ɵ��Ǹ��ڵ㣬ֻ�Ǿۺϲ��������˸ı䣬��Ϊ������Ҫ����Ĳ��֡�

### JoinSelection

����˼�壬�ò��Ե�Ŀ�ľ��ǻ���Joining key��logical plan�Ĵ�Сѡ����ʵ�Physical plan������ͨ��ExtractEquiJoinKeys�ҵ�equi-join key��Ȼ����������ѡ����ԣ�

1. Broadcast�����join����ĳһ��ı�Ԥ���СС����ֵ���û�����SQLConf.AUTO_BROADCASTJOIN_THRESHOLD������SQL�����ָ���������û����Խ�`org.apache.spark.sql.functions.broadcast()`Ӧ����DataFrame��Ҫ�㲥����ô�͹㲥��һ��ı�
2. Shuffle hash join�����ÿ��partition���㹻С���ʺϽ���hash��Ļ�������û����������ʹ��SortMergeJoin��ѡ��ò��ԡ�
3. Sort merge����������join key���ǿ�����ģ����øò��ԡ�

���û��join key���������£�

1. BroadcastNestedLoopJoin�����join����ĳ��ı���Ա��㲥��
2. CartesianProduct������Inner Join��Inner Join�п��԰������˵Ⱥ�֮��������ȽϷ�������ڡ�С�ڡ������ڵȣ��������ֿ��Բ²�Ӧ�þ������ѿ�������
3. BroadcastNestedLoopJoin��������

> ���䣺ExtractEquiJoinKeys�л�����EqualNullSafe���ò�����SQL�о���`<=>`��������Ϊ��ʱ���޷�ȷ����ʽ�����������ʵ�ʲ����л᲻����ֿգ�����������������ø����Ե�Ĭ��ֵ�滻����֤˫����Ϊnull��ʱ�򷵻�true����һ��Ϊnull������false��

���ڱ��С��Ԥ�ⷽ��������˼·���Ǹ���������Ԥ�������в�������ͬ������Ԥ�ⷽʽ��ͬ�����ﲻһһ׸����ֻ��Ҷ�ӽڵ��Ԥ�ⷽʽ������LocalRelation����Ԥ�ⷽʽ���ǽ������ÿ����¼�Ĵ�С������Ԫ���͵ĳ���֮�ͣ����Լ�¼�����������Ԫ���ݿ���ȷ����
����InMemoryRelation������Ѿ�ʵ������ֱ��partition schema��ͳ����Ϣ���㣬��֮��ֱ����һ��Ĭ��ֵ���������ã�������ֻ���������֡�

### InMemoryScans

�ò�����Ҫ�����ڶ�InMemoryRelation��ɨ���������Ҫ����

def pruneFilterProject(
      projectList: Seq[NamedExpression],
      filterPredicates: Seq[Expression],
      prunePushedDownFilters: Seq[Expression] => Seq[Expression],
      scanBuilder: Seq[Attribute] => SparkPlan): SparkPlan = {

    val projectSet = AttributeSet(projectList.flatMap(_.references))
    val filterSet = AttributeSet(filterPredicates.flatMap(_.references))
    val filterCondition: Option[Expression] =
      prunePushedDownFilters(filterPredicates).reduceLeftOption(catalyst.expressions.And)

    if (AttributeSet(projectList.map(_.toAttribute)) == projectSet &&
      val scan = scanBuilder(projectList.asInstanceOf[Seq[Attribute]])
      filterCondition.map(FilterExec(_, scan)).getOrElse(scan)
    } else {
      val scan = scanBuilder((projectSet ++ filterSet).toSeq)
      ProjectExec(projectList, filterCondition.map(FilterExec(_, scan)).getOrElse(scan))
    }
  }

������Ҫ���ӵ�ӳ������������ʽǶ�ף���sum(max(a),min(b)������`AttributeSet(projectList.map(_.toAttribute)) == projectSet`������filter������������ʱ��ѯ��ͶӰ�����Ե��Ӽ�����ôֻҪ��Relation���й��˼��ɣ�����ʡȥ��ͶӰ������
��֮����Ҫ�ڹ��˲����ϼ�һ��ͶӰ�ڵ�ProjectExec����������Ĺ��˻�����ͶӰ�����漰�������ԣ���`(projectSet ++ filterSet).toSeq`��

### BasicOperators

ֱ�ӽ������Ĳ���ת��ΪPhysicalPlan�ڵ㣬û�ж���Ĳ���ѡ��

> �����е�LogicalPlanͬʱ������ֲ��ԣ�����ͨ��ÿ��������ж��ֲ��Կɹ�ѡ�񣬵�ÿ�ֲ���ֻ�᷵��һ�֡����ڵ�SQL��ʱ��û��ʵ�ֻ���Cost�Ĳ���ѡ�񣬶���Ҳû��ʵ�ּ�֦��ȥ�����õĲ��ԣ�������ϱ�ը����������Щ����TODO�С�
���Է���PhysicalPlan��LogicalPlan�ڵ��Ǻܲ�һ���ģ�LogicalPlan�Ľڵ���ǲ����߼����ܺ���⣬����PhysicalPlan�ڵ��Ǿ�����LogicalPlan�еĶ�������ϲ���һ����д����Ӷ�����ʵ�ʲ�ѯ�Ŀ��������Һ�LogicalPlan��ͬ���ǣ�
��û��PhysicalPlan����࣬�����г��ֵ�PhysicalPlanֻ�Ƿ��͵����ƣ�ʵ���ϴ���PhysicalPlan�ľ���SparkPlan��

[1]:https://github.com/summerDG/spark-code-ananlysis/blob/master/analysis/sql/spark_sql_optimize.md
[2]:http://www.cnblogs.com/CareySon/p/3411176.html
[physical-plan]:../../pic/physical_plan.png

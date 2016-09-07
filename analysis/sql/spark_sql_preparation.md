#Spark SQL ����֪ʶ

���Ĳο���[Spark Catalyst��ʵ�ַ���][1]������һ������Catalyst������ͼ����Ȼ���Ĳ����漰���̣�����֮��ķ������Ը�ͼΪָ����

[Catalyst][calalyst_flow]

�������ؽ��ܼ���SQL�еļ�����Ҫ����������������չ����

##Row

��ʾ��ϵ�����һ�����������һ��Trait�������кܶ����ʵ�֡�ʵ���ϱ�������˵����һ�����顣���Ǻ�RDD��ͬ���ǣ�
RDD�е����Ϳ���������ģ���DataFrame��ÿ�����ݵ�����ֻ����Row����Spark1.6֮��DataFrame�ͱ����DataSet[Row]�ı�����

Row��ʾ��ֻ����һ�нṹ�����ݣ��ǽṹ�����Ϸ���Row������`schema`������ָ�������ֶε����ͺ�������
����֧�ֵ����ݽṹ����������ģ����Ǳ���̳���DataType��Sparl SQL���Ѿ�ʵ�������ݿ��ֶεĻ������͡�
��Ҳ����̳�UserDefinedType�������Լ������ͣ��������Ҫʵ���Լ������л��ͷ����л����������������`schema`
�ͻ�ʹ�÷�����Get���������Ҳ�����ͨ���������в����������������Ͳ���ȫ�ģ���Ϊ���ݵ����͸��������ܵ�`schema`��Լ����

DataSet��Spark1.6֮��汾�ĸ��DataSet��RDD��DataFrameһ�������Ƿֲ�ʽ���ݽṹ�ĸ��
��������DataSet���������ض����ͣ�Ҳ���������轫����������������ΪRow����������`Row.fromSeq`����������ת��ΪRow��
��ʵ���ϣ�DataSet����������Ҳ��������Row���Ƶ����ͣ���Seq��Array��Product��Int�ȣ���������Щ���Ͷ���ת��ΪCatalyst
�ڲ���InternalRow��UnsafeRow��

DataSet�ĺ��ĸ������Encoder��������߳����������ʽת���������Ľ綨����ȥ���˽������Ľ綨��������ʵ��һ��Ĭ�ϵĲ������Ǵ���һ����Ӧ����ʽֵ��
��ȡ�䱾���䶼����Ϊ���������һ����������ģ������磺

	private[sql] implicit val exprEnc: ExpressionEncoder[T] = encoderFor(encoder)

���ڴӵ�����˵�����Ϳ��Դ����������ͣ�����ʵ���ϵĶ�������������ʵ�����п��ܣ����Դ������ͽ綨����Java��������ܾͱȽ����ˣ�
��ֻ��ȷ�����͵����½硣scala�г��˿����޶����½磬������������ͼ�綨�������Ľ綨�������ߵ�Ŀ����һ���ģ����������޶��ض������ͣ�
�������½�֮�ڵģ�������ʽ������ġ���������ǰ����Ҫ������ʽת��������`implicit ev: A => B`����������Ҫ������ʽֵ������`implicit ev: B[A]`����

������˵һ��Scala��TypeTag����ࣨ�ܶ�ط����õ����ο�[����][2]����TypeTag�����ڽ��Scala�ķ��ͻ��ڱ����ʱ�򱻲��������⡣����ʵҲ��Java�����⡣
Ϊ�˲�������������TypeTag���������������磺`typeTag[List[Int]]`����ʱ��ֵΪ`TypeTag[scala.List[Int]]`��`typeTag[List[Int]].tpe`��`typeOf[List[Int]]`
��ֵһ������`scala.List[Int]`����TypeTag���Ƶ���ClassTag����CLassTypeֻ��������ʱ�������������Ϣ�����磺`ClassTag[scala.List[Int]]`����`scala.collection.immutable.List`��
`typeTag[T].mirror`���Ի�õ�ǰ�����µ����п������ͣ�������classloader��������Catalyst�õ�����������������ͣ����Թ���Scala�ķ�����Ʋο�[Scala doc][3]��

�ص�Encoder�������þ��ǽ��ⲿ����ת��ΪDataSet�ڲ���InternalRow���������ת���������ͼ��ġ�����InternalRow����һ�����࣬��MutableRow��
����UnsafeRowҲ��MutableRow�����࣬����Ϊ���޸ĵ�InternalRow���ںܶ�ط�������������ԭ��ܼ򵥣�֧��set�Ȳ������ѡ�


##Expression

��SQL����У�����SELECT FROM�ȹؼ������⣬�����󲿷�Ԫ�ض��������ΪExpression������SELECT sum(a), a������sum(a)��a��ΪExpression�������е�ȻҲ����������
ÿһ��DataSet�ڴ�����ʱ�򶼻���һ����Ӧ��ExpressionEncoder����ExpressionEncoder�����������������Expression��صĶ���`serializer: Seq[Expression]`��
`deserializer: Expression`��ǰ�����ڽ�����һ����¼�и�������������ת��ΪCalalyst��InternalRow���������ڽ�InternalRowת��Ϊ��Ӧ���͡�
����Expression�����Ա�ʾ�����ʽ֮�������Ԫ�أ������ԡ��������С������κ�һ��DataSet[T]�����Ȼ�����һ��ExpressionEncoder����ʽֵ��
���ɸ���ʽֵ�����̣���ScalaReflection��������У�Ϊ��

1. ����������T������Ӧ����һ��������Row����Product�����ͣ�
2. ͨ�������ͽ�������Ӧ���������ɶ�Ӧ��ã������飩������Expression������һ��CreateNamedStruct���ͣ��̳���Expression����
���磺���_FUNC_(name1, val1, name2, val2, ...)����һ�����ݣ��ö���Ϳ�����Ч�ر�ʾ�������ҿ���`flatten`��Ϊһ��Expression����Ӧ`serializer`����
ÿһ��Expression���ڽ���һ����namei��vali����
3. ֻҪ����Ŀ������T����ô��һ��������һ����Ӧ��Expression���ڽ������InternalRowת��Ϊ�����͵Ķ���
4. ����3��4���ɵ�`serializer`��`deserializer`���Լ���T��ȡ����Schema���Լ�T��Ӧ��ClassTag����ExpressionEncoder����

* Expression��һ��Tree�ṹ���ṹ�Ͽ�����һ��������������child��Ҳ����û�У�������ͨ���༶��Child Expression����ϳɸ��ӵ�Expression��ǰ���ᵽ�Ķ�ԭʼ���ݽ���ת������һ�����ӵ�Expression��
* Expression������������ֵ������`eval`����������InternalRowȻ�󷵻ؽ����
* ��ȻExpression�Ĺ�������ֵ����ô�����������������͵����ơ�ÿ��Expression����def dataType: DataType���ͱ�������ʾ����������ͣ��Լ�def checkInputDataTypes(): 
TypeCheckResult������У�鵱ǰExpression�����루ΪTree�ṹ����ô�������뼴ΪChild Expression������Ƿ��������Ҫ��
* Expression���������Row���мӹ������ǿ��԰Ѽӹ�������Ϊ���¼���
	* ԭ����def eval(input: InternalRow = null): Any������
	* ���ڰ����ӱ��ʽ��Expression���磺UnaryExpression��BinaryExpression��TernaryExpression�ȣ���Expression�ļ����ǻ���Child Expression���������ж��μӹ��ģ�
	��˶�������Expression����Eval����Ĭ��ʵ�֣�����ֻ��Ҫʵ�ֺ���`def nullSafeEval(input: Any): Any`�����ԡ�
	* ExpressionҲ�����ǲ�֧��eval�ģ���Unevaluable���͵�Expression��һ�������������1)������޷���ֵ�����紦��Unresolved״̬��Expression��
	2)�ǲ�֧��ͨ��eval������ֵ������Ҫͨ��gencode�ķ�ʽ��ʵ��Expression���ܣ������˶�ȫ�ֲ�����Expression�����磺Aggravation��Sorting��Count����;
	3)ExpressionΪRuntimeReplaceable���ͣ�����IfNull��NullIf��Nvl��Nvl2��������������parser�׶�һ����ʱExpression�����Ż��׶Σ��ᱻ�滻Ϊ���Expression�������������Ҫ��ִ���߼������ǵ����滻��ص��߼���
	* Projection���ͣ��������Ǵ�ͳ�����ϵ�Expression�����������Ը���N��Expression��������row��N���ֶηֱ���мӹ������һ���µ�Row����Expression��������
	
�����Expression���з��ࣺ
**��������**���ⲿ�ֻ������Ǽ̳���LeafExpression����û���ӱ��ʽ������ֱ�Ӳ������ݡ�

| Name      | �������� |
| --------- | -------- |
| Attribute | Catalyst������Ϊ��Ҫ�ĸ���������Ϊ������ԣ���sql��������׶λ��в�ͬ����̬������UnresolvedAttribute->AttributeReference->BoundReference������������� |
| Literal   | ������֧�ָ������͵ĳ������� |
| datetimeExpressions | �Ե�ǰʱ�����ͳ�����ͳ�ƣ���������ʱ�������������`CurrentDate`,`CurrentTimestamp` |
| randomExpressions | �����ض�������ֲ�����һЩ���������Ҫ����RDG����������ֲ���|
| ����һЩ���� | �����ȡsql��������е������Ӧ��InputFileName��SparkPartitionID |

**�������㹦��**���ⲿ�ֻ����������ӱ��ʽ�����Ի������Ǽ̳���UnaryExpression��BinaryExpression��BinaryOperator��TernaryExpression��

| Name      | ��ֵ��ʽ | �������� |
| --------- | :-------: | -------- |
| arithmetic | nullSafeEval | ��ѧExpression��֧��`-`,`+`,`abs`, `+`,`-`,`*`,`/`,`%`,`max`,`min`,`pmod`��ѧ����� |
| bitwiseExpressions | nullSafeEval | λ��������֧��IntegralType���͵�`and`,`or`,`not`,`xor`λ���� |
| mathExpressions | nullSafeEval | ��ѧ������֧��`cos`,`Sqrt`֮��30����,�൱��Math�� |
| stringExpressions | nullSafeEval | �ַ���������֧��`Substring`,`Length`֮��30���֣��൱��String�� |
| decimalExpressions | nullSafeEval | Decimal���͵�֧�֣�֧��`Unscaled`,`MakeDecimal`���� |
| datetimeExpressions | nullSafeEval | ʱ�����͵����㣨�����治ͬ���ǣ�����ָ���㣩|
| collectionOperations | nullSafeEval | �����Ĳ�������ʱ֧������`ArrayContains`,`ArraySort`,`Size`��`MapKeys`��`MapValues`5�ֲ��� |
| cast | nullSafeEval | ֧���������͵�ת�� |
| misc | nullSafeEval | ���ܺ�������֧��MD5��crc32֮��ĺ������� |

**�����߼����㹦��**���������ǡ�������ƥ�䡣

| Name      | ��ֵ��ʽ | �������� |
| --------- | :-------: | -------- |
| predicates  | eval/nullSafeEval���� | ֧����Expression֮����߼����㣬����`AND`,`In`,`Or`�����blooean |
| regexpExpressions|nullSafeEval | ֧��LIKE��ز���������blooean |
| conditionalExpressions | eval | ֧��Case����ΪCaseWhen��CaseWhenCodegen����If�����߼��ж����� |
| nullExpressions | eval/RuntimeReplaceable | ��NULL/NA��ص��жϻ���IF�жϹ��ܣ��󲿷ֶ�ΪRuntimeReplaceable���ᱻ�����Ż����� |

**��������**

| Name      | ��ֵ��ʽ | �������� |
| --------- | :-------: | -------- |
| complexTypeCreator | eval | SparkSql֧�ָ������ݽṹ������Array��Map��Struct������Expression֧����sql������������ǣ�����select array��������Projection���͡� |
| Generator | eval | ֧��flatmap���ƵĲ���������Rowת��Ϊ���Row��֧��Explode���Զ���UserDefinedGenerator���֣�����Explode֧�ֽ������map��Ϊ���Row��|

##Attribute

�����Ѿ����ܹ���Attribute��ʵҲ��һ��Expression���̳���NamedExpression�����Ǵ����ֵ�Expression��
Attributeֱ��Ϊ���ԣ���SQL�У����Լ����Ϊ�����Table�е��ֶΣ�Attributeͨ��Name�ֶ�������������
SQL���ͨ��Parse����AST�Ժ�SQL����е�ÿ���ֶζ������ΪUnresolvedAttribute����������Attribute��һ�����࣬����SELECT a�е�a�ͱ�ʾΪUnresolvedAttribute("a")��
SQL����е�`*`������ʾΪStar���̳���NamedExpression�������������ࣺUnresolvedStar��ResolvedStar��������analysis.unresolved�ļ��У���������ʵ��û��ת����ϵ��
ǰ������AST�������������ڲ�ѯ��

�������query��AST�ӹ������к���Ҫ��һ��������ǽ�����AST������Unresolved��Attribute��ת��Ϊresolved״̬�����������ASTBuilder��Analyzer�������ɣ�
ǰ����������unresolved attribute������Star��relation�ȣ���������ͨ��Logic plan����Щunresolved����Щunresolved attribute�������ɹ̶���AttributeReference����relation�ȣ���Բ�ͬ�������ղ�һ�����ù���Ŀ�ľ��ǡ��̶�������

���⣬resolve��������Ҫ���ܾ��ǹ���SQL�������λ���õ���Attribute������Attribute��name�����ϣ�ָ��һ��ID����Ψһ��ʾ��
**���һ��Attribute���������ദ�����ã�ID��Ϊͬһ��**��

> Attribute Resolve����ʱ**�ӵ׵���**����������AST��ÿһ������**����**�ײ�**�Ѿ�resloved��Attribute**����������Attribute��ֵ���Ӷ���֤�������Attribute��ָ��ͬһ�������ǵ�ID�϶���һ����)��

������ô��⣬����Щ���鶼��Ϊ���Ż�������洢��Table�����кܶ�Attribute����ͨ��resolve��������ָ�����������������Ҫʹ�õ�Attribute��������ֻ������洢�ж�ȡ��Ӧ�ֶΣ�
�ϲ����Expression����Щ�ֶζ�ת��Ϊ���ã����resolve�Ժ��Attribute���ǽ���resolvedAttribute,���ǽ���AttributeReference��

����һ���м�ڵ��Expression���������һ��Attribute�����ã�������һ���ֶ�ֵ�ĳ���length(a)������a������UnresolvedAttribute��AttributeReference��ת�����������һ�������Row��
����lengthExpression����ʱ�������޷���AttributeReference�ж�ȡ��Ӧ��Row�е�ֵ��Ϊʲô����ȻAttributeReferenceҲ��Expression����������Unevaluable��Ϊ�˻�ȡ����������Row�ж�Ӧ��ֵ��
��Ҫ��AttributeReference�ٽ���һ��BindReferences��ת��������BoundReference������������ʾ��ǽ�Expression��һ������Scheme���й�����Scheme��һ��AttributeReference������֮������˳��ģ�
ͨ��Expression��AttributeReference��Schema AttributeReference���е�Index��������BoundReference���ڶ�BoundReference����evalʱ�򣬼�����ʹ�ø�index��ȡ������ӦRow�е�ֵ��

##QueryPlan

�������ԣ���SQL����У�����SELECT FROM�ȹؼ������⣬�����󲿷�Ԫ�ض��������ΪExpression����ô��ʲô����ʾʣ�µ�SELECT FROM��Щ�ؼ����أ��Ͼ�Expressionֻ��һЩEval���ܺ������ߴ���Ƭ�Σ���Ҫһ��������������ЩƬ�Σ������������Plan��������˵��QueryPlan��

QueryPlan���ǽ�����Expression��֯������������LogicalPlan��PhysicalPlan��Դ����û�и����ӿڣ���plan.physical���о�����ʽ����Plan������ʽҲ��Tree���ڵ�֮��Ĺ�ϵ�������Ϊһ�ֲ������򣬱���PlanҶ�ӽڵ��ʾ�Ӵ��̶�ȡDB�ļ�����Root�ڵ��ʾ�������ݵ������������Plan�����ʵ����ͼ��

![QueryPlan][plan_img]

��SQL�������ʾ���Plan��Ϊ��`SELECT project FROM table, table WHERE filter`��

ֱ����⣬Expression�ǳ���SELECT FROM֮����Կ�����Item��Plan���ǽ�Expression����һ����ִ��˳��ִ�С�

Expression�����Ƕ�����Row���мӹ������������Any�������͡���Plan�������Ϊdef output: Seq[Attribute]��ʾ��һ��Attribute�����������Project��Table�϶������һ����Seq[Attribute]���ͱ�ʾ��Row��
Filter�о������Ture/False����������˵��Plan��������Filter���͵�Expreesion��Filter���͵�Plan�����ڲ�����Expression���������ж��Ƿ񷵻�Row������Row���ص����Ϳ϶�Ҳ����Seq[Attribute]��ʾ�ġ�
����˵����Filter���Ƿ���Seq[Attribute]��

ͬ��LogicalPlan�ӽṹ�Ϸ�Ҳ�е��ڵ㣬Ҷ�ڵ㣬˫�ڵ㡣

Catalyst�Ƕ�AST�����������У����LogicalPlan������������Expression�Ĺ���������߼���org.apache.spark.sql.catalyst.parser.AstBuilder�Լ���������У�
���������Ĺ�����ParseDriver�У������еĹ��̸��Ӻ��������

LogicalPlanҲ��Tree�νṹ����ڵ��Ϊ�������ͣ�Operator��Command��Command��ʾ�����ѯ��ָ�����ִ�У����磺Command���Ա�������ʾDDL������
Operatorͨ������ɶ༶��Plan��Operator���඼��basicLogicalOperators���档����ֻȡ��ʱ���ö��ġ�

 Name      |   ��������
-------- | --------
|`Project`(projectList: Seq[NamedExpression], child: LogicalPlan)|SELECT����������������projectListΪ�������ÿһ����Ϊһ��Expression�����ǿ�����Star�����ߺܸ��ӵ�Expression|
|`Filter`(condition: Expression, child: LogicalPlan)|����condition����Child�����Rows���й���|
|`Join`(left: LogicalPlan,right: LogicalPlan,joinType: JoinType,condition: Option[Expression])|left��right������������join����|
|`Intersect`(left: LogicalPlan, right: LogicalPlan)|left��right����Plan�����rows����ȡ�������㡣|
|`Except`(left: LogicalPlan, right: LogicalPlan)|��left���������޳���right�еļ�����|
|`Union`(children: Seq[LogicalPlan])|��һ��Childs�ļ���������Union����|
|`Sort`(order: Seq[SortOrder],global: Boolean, child: LogicalPlan)|��child���������sort����|
|`Repartition`(numPartitions: Int, shuffle: Boolean, child: LogicalPlan)|��child��������ݽ������·�������|
|`InsertIntoTable`(table: LogicalPlan,child: LogicalPlan,...)|��child�����rows�����table��|
|`Distinct`(child: LogicalPlan)|��child�����rowsȡ�ز���|
|`GlobalLimit`(limitExpr: Expression, child: LogicalPlan)|��Child��������ݽ���Limit����|
|`Sample`(child: LogicalPlan,....)|����һЩ��������child�����Rows����һ��������ȡ��|
|`Aggregate`(groupingExpressions: Seq[Expression],aggregateExpressions: Seq[NamedExpression],child: LogicalPlan)|��child���row����aggregate����������groupby֮��Ĳ���|
|`Generate`(generator: Generator,join: Boolean,outer: Boolean,ualifier: Option[String],generatorOutput: Seq[Attribute],child: LogicalPlan)|�������ڵݹ�Ȳ�������������������ʽ���룬����������ʽ�����������`flatMap`�����������������������һ��|
|`Range`(start: Long,end: Long,step: Long,numSlices: Option[Int],output: Seq[Attribute])|��������ݵķ�Χ����Լ��|
|`GroupingSets`(bitmasks: Seq[Int],groupByExprs: Seq[Expression],child: LogicalPlan,aggregations: Seq[NamedExpression])|�൱�ڰѶ��Group By�����ϲ�����|

�������Command�࣬��Щ�඼�̳���Command������������Operator�ࡣ

| Name      |   �������� |
| :-------- | --------|
|`DataBase`������|֧��ShowDatabase�Լ�UseDatabase�Լ�Create�Ȳ���|
|`Table`������|���13�֣�����Create��Show��Alter��|
|`View`������|CreateViewCommand֧��View�Ĵ���|
|`Partition`������|֧��Partition����ɾ���Ȳ���|
|`Resources`������|����AddJar��AddFile֮�����Դ����|
|`Functions`������|֧������������ɾ�������Ȳ���|
|`Cache`������|֧�ֶ�Table����cache��uncache����|
|`Set`����|ͨ��SetCommandִ�жԲ�������������Shuffle��Partition����������ʱ�޸�|

LogicalPlan��Ҫ��ת��Ϊ���յ�PhysicalPlan�����������п�ִ�е�����������ЩCommand���͵�Plan������`def run(sparkSession: SparkSession): Seq[Row]`������¶��Spark SQL��
����ͨ������Table��run�������Table�Ĵ����Ȳ�����

##Tree�Ĳ���

TreeNode�ڵ㱾������ΪProduct����Scala��Product���������������֮һ���������������Tuple��List��Option��case��ȣ����һ��Case Class`�̳�Product��
��ô�����ͨ��`productElement`��������`productIterator`��������`Case Class`��**������Ϣ**���������ͱ���������������Expression��Plan��������`Product`���ͣ�
��˿���ͨ��TreeNode�ڲ������`mapProductIterator`�����Խڵ�������б�����

��Plan��Expression���б�����Ŀ�ģ�������Ϊ���ռ�һЩ��Ϣ���������Tree����map/foreach�����������Ϊ�˶�Tree�ڵ��ڲ�����Ϣ�����޸ģ�
�����PlanTree��ÿ��Plan�ڵ��ڲ����õ�Attribute����Revole������������Ϊ��Tree�����ݽṹ�����޸ģ�����ɾ��Tree���ӽڵ㣬�Լ����ӽڵ���кϲ���
����Catasylt Optitimze���д���Tree�ṹ���޸ġ�

��Tree����ת���Ĳ����õ���`rule`������Scala��ƫ����ʵ�ֵģ�[ƫ����ʹ��][4]��ƫ������Ҫ����ƥ�䣩��

��Expression��LogicalPlan��**����**ͨ�����ᱻ����ͬһ��Object�У����Object�е�aplly�������������������ͬ������̳���Rule[T]��T�����������ͣ������ڡ���ѧScala���е�18.12���������е���ƣ���

	abstract class Rule[TreeType <: TreeNode[_]] extends Logging {

	  val ruleName: String = {
		val className = getClass.getName
		if (className endsWith "$") className.dropRight(1) else className
	  }

	  def apply(plan: TreeType): TreeType
	}
	
������Խ�һ��`Rule`���Ϊһ��`Batch(name: String,rules: Rule[TreeType]*)`��������װ��`RuleExecutor`�У��Ӷ�ͨ��`RuleExecutor`������`Rule`�Ŀ�ִ�нӿ��ṩ���ⲿʹ�ã�
����Optimize���ԣ�����һ�Ѷѵ�Batch��ɡ���Batch�е�ÿ��Rule����������ɶ�LogicalPlan�����Ż�����ִ��`plan`��ֱ���ڵ���������������ǰ�ﵽfix point��

�����Ż��ܿ��ܻ����ĺܳ�ʱ�䣬����ÿ��Batch����Strategy��������������Once��FixedPoint��ǰ�߱�����Batchֻ����ִ��һ�Σ����߻��趨�Դ����������

Spark SQL��Plan Tree�����ڲ�Expression Tree�ı�����Ϊ�����׶Σ�

1. ��AST����Parse����������Unresolve Plan��
2. ��Unresolve Plan����Analysis(����Resolve)����������Logical Plan��
3. ��Logical Plan����Optimize����������Optimized Logical Plan��
4. �Լ�������Planning����������Physical Plan��

> �������ÿһ�׶ζ����Լ���ΪӦ��һ��BatchRule����plan���мӹ�

[1]:https://github.com/ColZer/DigAndBuried/blob/master/spark/spark-catalyst.md
[2]:http://stackoverflow.com/questions/12218641/scala-what-is-a-typetag-and-how-do-i-use-it
[3]:http://docs.scala-lang.org/overviews/reflection/overview.html
[4]:http://blog.csdn.net/u010376788/article/details/47206571
[calalyst_flow]:../pic/Catalyst-Optimizer-diagram.png
[plan_img]:../pic/plan.png
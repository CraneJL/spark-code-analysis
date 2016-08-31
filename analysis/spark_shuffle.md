#Spark������Shuffleʵ��

##Job��Stage��Task��Dependency

###��һ��Job����Shuffle����ʼ

Job���Կ�����һ��transformation������һ��action�����ļ��ϣ����仰˵ÿ��action�����ᴥ��һ��Job���鿴Դ��
���Է���ÿ��action������������ǵ���`sc.runJob(this,...)`����SparkContext�µ�runJob��������ʵ���е�runJob��
���ط������ն���������µĹ��̣�

	def runJob[T, U: ClassTag](
		  rdd: RDD[T],
		  func: (TaskContext, Iterator[T]) => U,
		  partitions: Seq[Int],
		  resultHandler: (Int, U) => Unit): Unit = {
		if (stopped.get()) {
		  throw new IllegalStateException("SparkContext has been shutdown")
		}
		val callSite = getCallSite
		val cleanedFunc = clean(func)
		logInfo("Starting job: " + callSite.shortForm)
		if (conf.getBoolean("spark.logLineage", false)) {
		  logInfo("RDD's recursive dependencies:\n" + rdd.toDebugString)
		}
		dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
		progressBar.foreach(_.finishAll())
		rdd.doCheckpoint()
	}
`resultHandler`��������֮ǰ��runJob����ģ���Ҫ���������ûص����������ؽ����ע�����ﲢ�������÷�������ֵʵ�ֽ�����ء�
`func`���������ÿ��partition����action�����ĺ�����`partitions`�Ǹ���partition��Ź��ɵ����顣dagScheduler����Spark����
��ĵ�������ͨ������Ҳ����֪��Job��һ��DAGͼ��`dagScheduler.runJob`��һ������������Job���֮��rdd�����checkpoint��
��������DAGScheduler.runJob

	def runJob[T, U](
		  rdd: RDD[T],
		  func: (TaskContext, Iterator[T]) => U,
		  partitions: Seq[Int],
		  callSite: CallSite,
		  resultHandler: (Int, U) => Unit,
		  properties: Properties): Unit = {
		val start = System.nanoTime
		val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
		val awaitPermission = null.asInstanceOf[scala.concurrent.CanAwait]
		waiter.completionFuture.ready(Duration.Inf)(awaitPermission)
		waiter.completionFuture.value.get match {
		  case scala.util.Success(_) =>
			logInfo("Job %d finished: %s, took %f s".format
			  (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
		  case scala.util.Failure(exception) =>
			logInfo("Job %d failed: %s, took %f s".format
			  (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
			// SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
			val callerStackTrace = Thread.currentThread().getStackTrace.tail
			exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
			throw exception
		}
	}
���Է��ָ÷����Ƕ����ģ�����ľ���`waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)`�����ύ��ҵ����ʵ
���Ѿ������ˡ�

	//DAGScheduler
	def submitJob[T, U](
		  rdd: RDD[T],
		  func: (TaskContext, Iterator[T]) => U,
		  partitions: Seq[Int],
		  callSite: CallSite,
		  resultHandler: (Int, U) => Unit,
		  properties: Properties): JobWaiter[U] = {
		// Check to make sure we are not launching a task on a partition that does not exist.
		val maxPartitions = rdd.partitions.length
		partitions.find(p => p >= maxPartitions || p < 0).foreach { p =>
		  throw new IllegalArgumentException(
			"Attempting to access a non-existent partition: " + p + ". " +
			  "Total number of partitions: " + maxPartitions)
		}

		val jobId = nextJobId.getAndIncrement()
		if (partitions.size == 0) {
		  // Return immediately if the job is running 0 tasks
		  return new JobWaiter[U](this, jobId, 0, resultHandler)
		}

		assert(partitions.size > 0)
		val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
		val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
		eventProcessLoop.post(JobSubmitted(
		  jobId, rdd, func2, partitions.toArray, callSite, waiter,
		  SerializationUtils.clone(properties)))
		waiter
	}
ǰ�������ж�rdd��partition�ʹ����partitions����Ŀһ�²ſ��Դ���֮��������Job����һ��JobId�������rddû��
partition��ֱ���ж�Ϊ����ɹ���ֱ�ӷ���һ��JobWaiter���󣩣���֮����JobId�͸�rdd����һ��JobWaiter����Ȼ����
`eventProcessLoop`����DAG������������һ��JobSubmitted����Ϣ���������ڵİ汾��Spark�Ѿ���Netty������Akka������
������Akka����д���������JobWaiterҲ��һ�����˳�ȥ��`waiter`�����װ�˺ܶ���Ϣ�����������������ڽ��ս����
�ص�������`eventProcessLoop`��һ��DAGSchedulerEventProcessLoop�������ڽ��յ�һ��event������DAGScheduler��
handleJobSubmitted������

	private[scheduler] def handleJobSubmitted(jobId: Int,
		  finalRDD: RDD[_],
		  func: (TaskContext, Iterator[_]) => _,
		  partitions: Array[Int],
		  callSite: CallSite,
		  listener: JobListener,
		  properties: Properties) {
		var finalStage: ResultStage = null
		try {
		  // New stage creation may throw an exception if, for example, jobs are run on a
		  // HadoopRDD whose underlying HDFS files have been deleted.
		  finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
		} catch {
		  case e: Exception =>
			logWarning("Creating new stage failed due to exception - job: " + jobId, e)
			listener.jobFailed(e)
			return
		}
		val job = new ActiveJob(jobId, finalStage, callSite, listener, properties)

		submitStage(finalStage)
	}
����ɾȥ����û�õĴ��루����log��Ϣ��webͳ��ҳ�棩���÷�������Ҫ�������Ǹ���rdd�ͺ�jobId����finalStage������
����`submitStage(finalStage)`�����ύ���С������漰��stage��ÿ��Jobֻ��һ��finalStage����������DAG���һ��Stage��
�����м�����������ж������0����ShuffleStage���ϸ�˵��ShuffleMapStage����ShuffleStage���漰���������Ż��еġ�
����createResultStage

	private def createResultStage(
		  rdd: RDD[_],
		  func: (TaskContext, Iterator[_]) => _,
		  partitions: Array[Int],
		  jobId: Int,
		  callSite: CallSite): ResultStage = {
		val parents = getOrCreateParentStages(rdd, jobId)
		val id = nextStageId.getAndIncrement()
		val stage = new ResultStage(id, rdd, func, partitions, parents, jobId, callSite)
		stageIdToStage(id) = stage
		updateJobIdStageIdMaps(jobId, stage)
		stage
	}
	private def getOrCreateParentStages(rdd: RDD[_], firstJobId: Int): List[Stage] = {
		getShuffleDependencies(rdd).map { shuffleDep =>
		  getOrCreateShuffleMapStage(shuffleDep, firstJobId)
		}.toList
	}
	private[scheduler] def getShuffleDependencies(
		  rdd: RDD[_]): HashSet[ShuffleDependency[_, _, _]] = {
		val parents = new HashSet[ShuffleDependency[_, _, _]]
		val visited = new HashSet[RDD[_]]
		val waitingForVisit = new Stack[RDD[_]]
		waitingForVisit.push(rdd)
		while (waitingForVisit.nonEmpty) {
		  val toVisit = waitingForVisit.pop()
		  if (!visited(toVisit)) {
			visited += toVisit
			toVisit.dependencies.foreach {
			  case shuffleDep: ShuffleDependency[_, _, _] =>
				parents += shuffleDep
			  case dependency =>
				waitingForVisit.push(dependency.rdd)
			}
		  }
		}
		parents
	}
�����ǵ���`getOrCreateParentStages(rdd, jobId)`�����ɸ�Stage�����Է�����`getOrCreateParentStages`���Ǹ�Stage������
shuffle�������ɵġ�����`getShuffleDependencies`�����Է���ֻ����ShuffleDependency������������������²ŻὫ����Ϊ������������
�����Ҳ����խ������ֻ��׷���丸������ShuffleDependency������������ֻ��׷��һ�����������磺A<--B<--C��ʾC������Shuffle������B��
B����A������ֻ�������B<--C��`getOrCreateParentStages`��Ϊÿ��Shuffle��������һ��ShuffleStage��Ҳ����ShuffleMapStage��
ShuffleMapStage�������Ƚ���Ҫ�ĺ�����`isAvailable`�жϵ�ǰStage�Ƿ�������ɣ��ж�����һĿ��Ȼ��`addOutputLoc`����Ҫ�Ƕ�
`_numAvailableOutputs`�����޸ģ�`outputLocs`������partition�����Ϊ�±꣬ÿ����Ŷ�Ӧ�Ŀ���MapStatus����Ϊһ��partition�������ж�Σ���
MapStatus����֮����н��ܣ��ú�����Ҫ����`createShuffleMapStage`�е��ã�ÿ��һ��partition��shuffle������ͻ����һ�Σ�������partition���
�����ʱ����stageҲ������ˡ�

	def isAvailable: Boolean = _numAvailableOutputs == numPartitions
	def addOutputLoc(partition: Int, status: MapStatus): Unit = {
		val prevList = outputLocs(partition)
		outputLocs(partition) = status :: prevList
		if (prevList == Nil) {
		  _numAvailableOutputs += 1
		}
	}
����������MapStatus��

	private[spark] sealed trait MapStatus {
	  def location: BlockManagerId

	  def getSizeForBlock(reduceId: Int): Long
	}
��ֻ����һ������location��������������е�λ�ã�Ҳ����Map��������λ�á����ж�reduceId��Ӧ��Block��Ԥ���С����ʵԤ����ص�Ŀ�������Ԥ��ֵΪ0
��Block�����м��㡣����ResultStage�Ƿ���ɵ��ж����������������

	//ResultStage
	def activeJob: Option[ActiveJob] = _activeJob

	private[spark] class ActiveJob(
		val jobId: Int,
		val finalStage: Stage,
		val callSite: CallSite,
		val listener: JobListener,
		val properties: Properties) {

	  /**
	   * Number of partitions we need to compute for this job. Note that result stages may not need
	   * to compute all partitions in their target RDD, for actions like first() and lookup().
	   */
	  val numPartitions = finalStage match {
		case r: ResultStage => r.partitions.length
		case m: ShuffleMapStage => m.rdd.partitions.length
	  }

	  /** Which partitions of the stage have finished */
	  val finished = Array.fill[Boolean](numPartitions)(false)

	  var numFinished = 0
	}
ResultStage����activeJob��ȡ��Job��Ȼ��Job����һ��finished������ʾ����partition�Ƿ���ɣ�����finish���޸�����
`DAGScheduler.handleTaskCompletion(event: CompletionEvent)`����ɵġ�

����������`submitJob`����

	private def submitStage(stage: Stage) {
		val jobId = activeJobForStage(stage)
		if (jobId.isDefined) {
		  logDebug("submitStage(" + stage + ")")
		  if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
			val missing = getMissingParentStages(stage).sortBy(_.id)
			logDebug("missing: " + missing)
			if (missing.isEmpty) {
			  logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
			  submitMissingTasks(stage, jobId.get)
			} else {
			  for (parent <- missing) {
				submitStage(parent)
			  }
			  waitingStages += stage
			}
		  }
		} else {
		  abortStage(stage, "No active job for stage " + stage.id, None)
		}
	}
	private def getMissingParentStages(stage: Stage): List[Stage] = {
		val missing = new HashSet[Stage]
		val visited = new HashSet[RDD[_]]
		// We are manually maintaining a stack here to prevent StackOverflowError
		// caused by recursively visiting
		val waitingForVisit = new Stack[RDD[_]]
		def visit(rdd: RDD[_]) {
		  if (!visited(rdd)) {
			visited += rdd
			val rddHasUncachedPartitions = getCacheLocs(rdd).contains(Nil)
			if (rddHasUncachedPartitions) {
			  for (dep <- rdd.dependencies) {
				dep match {
				  case shufDep: ShuffleDependency[_, _, _] =>
					val mapStage = getOrCreateShuffleMapStage(shufDep, stage.firstJobId)
					if (!mapStage.isAvailable) {
					  missing += mapStage
					}
				  case narrowDep: NarrowDependency[_] =>
					waitingForVisit.push(narrowDep.rdd)
				}
			  }
			}
		  }
		}
		waitingForVisit.push(stage.rdd)
		while (waitingForVisit.nonEmpty) {
		  visit(waitingForVisit.pop())
		}
		missing.toList
	}
�����жϸ�Stage�Ƿ��ǵȴ�����Stageû����ɣ������С�ʧ�ܵ�״̬��Ȼ����`getMissingParentStages(stage)`��ȡ
finalStage������δ����ĸ�Stage�����ҵ����ύ��Stage�����ҰѸ�finalStage����waitingStage�����û�и�Stage��
��ô������`submitMissingTasks(stage, jobId.get)`�ύ��Stage��

`getCacheLocs(rdd)`�ǻ�ȡ��Stage�����RDD������Partition��cacheλ�ã���Ҫ���ǽ���`cacheLocs`���������ʾ��
��RDD����partirion��cacheλ�õ�ӳ�䣬����һ��HashMap��key��RDD��id��val��һ�����飬�±�Ϊpartition��id����
����cacheλ�á������û�д������partition����ô�жϸ�RDD��������Ӧ��Stage�Ƿ���ɣ����û����ɣ��Ͱ�û����
�ɵĸ�Stage����`missing`��

����������`submitMissingTasks`

	private def submitMissingTasks(stage: Stage, jobId: Int) {
		//step1
		stage.pendingPartitions.clear()

		val partitionsToCompute: Seq[Int] = stage.findMissingPartitions()

		val properties = jobIdToActiveJob(jobId).properties

		runningStages += stage
		//step 2
		val taskIdToLocations: Map[Int, Seq[TaskLocation]] = try {
		  stage match {
			case s: ShuffleMapStage =>
			  partitionsToCompute.map { id => (id, getPreferredLocs(stage.rdd, id))}.toMap
			case s: ResultStage =>
			  partitionsToCompute.map { id =>
				val p = s.partitions(id)
				(id, getPreferredLocs(stage.rdd, p))
			  }.toMap
		  }
		} catch {
		  ...
		}

		stage.makeNewStageAttempt(partitionsToCompute.size, taskIdToLocations.values.toSeq)
		listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo, properties))
		
		//step3
		
		var taskBinary: Broadcast[Array[Byte]] = null
		try {
		  // For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep).
		  // For ResultTask, serialize and broadcast (rdd, func).
		  val taskBinaryBytes: Array[Byte] = stage match {
			case stage: ShuffleMapStage =>
			  JavaUtils.bufferToArray(
				closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef))
			case stage: ResultStage =>
			  JavaUtils.bufferToArray(closureSerializer.serialize((stage.rdd, stage.func): AnyRef))
		  }

		  taskBinary = sc.broadcast(taskBinaryBytes)
		} catch {
		  ...
		}
		
		//step4
		val tasks: Seq[Task[_]] = try {
		  stage match {
			case stage: ShuffleMapStage =>
			  partitionsToCompute.map { id =>
				val locs = taskIdToLocations(id)
				val part = stage.rdd.partitions(id)
				new ShuffleMapTask(stage.id, stage.latestInfo.attemptId,
				  taskBinary, part, locs, stage.latestInfo.taskMetrics, properties)
			  }

			case stage: ResultStage =>
			  partitionsToCompute.map { id =>
				val p: Int = stage.partitions(id)
				val part = stage.rdd.partitions(p)
				val locs = taskIdToLocations(id)
				new ResultTask(stage.id, stage.latestInfo.attemptId,
				  taskBinary, part, locs, id, properties, stage.latestInfo.taskMetrics)
			  }
		  }
		} catch {
		  ...
		}
		
		//step5
		if (tasks.size > 0) {
		  stage.pendingPartitions ++= tasks.map(_.partitionId)
		  taskScheduler.submitTasks(new TaskSet(
			tasks.toArray, stage.id, stage.latestInfo.attemptId, jobId, properties))
		  stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
		} else {
		  // Because we posted SparkListenerStageSubmitted earlier, we should mark
		  // the stage as completed here in case there are no tasks to run
		  markStageAsFinished(stage, None)

		  submitWaitingChildStages(stage)
		}
	}
�ù��̴����Ϊ5����

��1����ָ����Stage��������partition����Ϊ�������¼��㣬���Դ��ڲ���partition�Ѿ����꣩��

Ȼ���2��������Բ�ͬ��Stageѡ�������ִ�е�λ�ã����ž����������У�ÿ����һ��Stage�������󣩣�������`listenerBus`����
`SparkListenerStageSubmitted`��Ϣ��ֻ���ڽ��յ������Ϣ��ſ��Բ��������Ƿ�����л���

��3���ǽ����������������Ϣ�������л����ҷ��͸�����������Բ�ͬ��Stage���͵���Ϣ���в�ͬ������ShuffleStage��ResultStage����ҪRDD��Ϣ������ÿ��
������һ��RDD���õĿ���������ShuffleStage��Ҫ���ⷢ��������ShuffleDependency������ResultStage�������`func`��Ϣ��

	class ShuffleDependency[K: ClassTag, V: ClassTag, C: ClassTag](
		@transient private val _rdd: RDD[_ <: Product2[K, V]],
		val partitioner: Partitioner,
		val serializer: Serializer = SparkEnv.get.serializer,
		val keyOrdering: Option[Ordering[K]] = None,
		val aggregator: Option[Aggregator[K, V, C]] = None,
		val mapSideCombine: Boolean = false)

���Է���ShuffleDependency������partitioner��������Ҫ����ʲô����������Map��Bucket�������, ��serializer����������ô
��Map��������� ���л�, ��keyOrdering��aggregator����������ô����Key���з�Bucket,�Ѿ���ô���кϲ�,�Լ�mapSideCombine
���������Ƿ���Ҫ����Map��reduce��

��4���������ÿ������ķ�Ƭ����Task���󣬷ֱ���`ShuffleMapTask`��`ResultTask`�����߶��������������λ�á�partition���ݡ�
��Stage����Ϣ������ǰ�����ɵĹ㲥��Ϣ�ȡ���������`ShuffleMapTask`���У���Ҫ��������`runTask`���Թ㲥������з����л���Ȼ������
����ShuffleManager�ľ���ShuffleWriter�����ݽ��д����ⲿ�ֻ���[Spark��shuffle���Ʒ���][1]�з�������
��`ResultTask`��`runTask`�Ĵ�����Լ򵥣����˷����л��㲥��Ϣ�⣬ʣ��������ǵ���`func`�������partition�Ľ����

��5���ǽ���������ͨ����������������������[Task���Ȼ���]()�������ύ�������Stage�������Ѿ���ϣ���ô�ͽ�����Ϊ��ɣ�
���ҿ�ʼ���ȵȴ�����Stage��

���ڼ��������Ѿ�������ϣ���ʼ������ν���������н�����ݸ�֮ǰ������`waiter`��

	private[scheduler] def handleTaskCompletion(event: CompletionEvent) {
		val task = event.task
		val taskId = event.taskInfo.id
		val stageId = task.stageId
		val taskType = Utils.getFormattedClassName(task)
		  
		listenerBus.post(SparkListenerTaskEnd(
		   stageId, task.stageAttemptId, taskType, event.reason, event.taskInfo, taskMetrics))
		   
		val stage = stageIdToStage(task.stageId)
		event.reason match {
		  case Success =>
			stage.pendingPartitions -= task.partitionId
			task match {
			  case rt: ResultTask[_, _] =>
				// Cast to ResultStage here because it's part of the ResultTask
				// TODO Refactor this out to a function that accepts a ResultStage
				val resultStage = stage.asInstanceOf[ResultStage]
				resultStage.activeJob match {
				  case Some(job) =>
					if (!job.finished(rt.outputId)) {
					  updateAccumulators(event)
					  job.finished(rt.outputId) = true
					  job.numFinished += 1
					  // If the whole job has finished, remove it
					  if (job.numFinished == job.numPartitions) {
						markStageAsFinished(resultStage)
						cleanupStateForJobAndIndependentStages(job)
						listenerBus.post(
						  SparkListenerJobEnd(job.jobId, clock.getTimeMillis(), JobSucceeded))
					  }

					  // taskSucceeded runs some user code that might throw an exception. Make sure
					  // we are resilient against that.
					  try {
						job.listener.taskSucceeded(rt.outputId, event.result)
					  } catch {
						...
					  }
					}
				  case None =>
					...
				}
				...
			}
		}
	}
����ɾȥ�˺ܶ��߼�������ֻʣ�ɹ�����ResultTask���������������δ����ﲢû��waiter����Ϣ��ʵ���������`job.listener`����ÿ��Job��Ӧ��Waiter����ΪJobWaiter��JobListener��
���࣬��waiter����ô����Job���أ���ʵ��֮ǰ��`handleJobSubmitted`���������У��Ϳ��Է��ֱ����listener����������finalStage
��listener�����˶�Ӧ��ActiveJob����

��������listenerBus���Ժ��������ܣ�����������ɵ���Ϣ�����ResultTask����ô˵����JobҲ����ˣ����Խ�Job���Ϊ��ɡ�
��listenerBus����Job��ɵ���Ϣ�����`job.listener.taskSucceeded(rt.outputId, event.result)`������Ľ�����ظ�`listener`����`waiter`����
JobWaiter��`taskSucceeded`�����е���`resultHandler`����������˵�Ļص�������ɽ���ķ��أ�������Ĵ�����ʾ����

	override def taskSucceeded(index: Int, result: Any): Unit = {
		// resultHandler call must be synchronized in case resultHandler itself is not thread safe.
		synchronized {
		  resultHandler(index, result.asInstanceOf[T])
		}
		if (finishedTasks.incrementAndGet() == totalTasks) {
		  jobPromise.success(())
		}
	}

###Shuffle Jobִ�й���

Shuffle�����������̣�Shuffle Map��Shuffle reduce��������MapReduce�е�map��reduce��Shuffle Map����ShuffleMapStage��
ShuffleMapTask������д����Ӧ�ļ��У������ļ�λ����MapOutput���ظ�DAGScheduler��������Stage��Ϣ��Reduce�����ò�ͬ���͵�RDD
��ʵ�ֵġ�

####Shuffle Map����

���Ƚ���DAGScheduler�е�`mapOutputTracker`��MapOutputTrackerMaster���󣩡�MapOutputTracker���ڼ�¼ÿ��Stage map�����λ�ã�MapStatus���󣩣�
�൱���Ǵ洢Ԫ������Ϣ������Driver�˺�Executor���в�ͬ��ʵ�֣����Է�ΪMapOutputTrackerMaster��MapOutputTrackerWorker��Master������
ע��ShuffleId��MapOutput��Ϣ����Executor��������ȡ��Щ��Ϣ��

	private[spark] class MapOutputTrackerMaster(conf: SparkConf,
		broadcastManager: BroadcastManager, isLocal: Boolean)
	  extends MapOutputTracker(conf) {
	  def registerShuffle(shuffleId: Int, numMaps: Int) {
		if (mapStatuses.put(shuffleId, new Array[MapStatus](numMaps)).isDefined) {
		  throw new IllegalArgumentException("Shuffle ID " + shuffleId + " registered twice")
		}
		shuffleIdLocks.putIfAbsent(shuffleId, new Object())
	  }
	  def registerMapOutputs(shuffleId: Int, statuses: Array[MapStatus], changeEpoch: Boolean = false) {
		mapStatuses.put(shuffleId, Array[MapStatus]() ++ statuses)
		if (changeEpoch) {
		  incrementEpoch()
		}
	  }
	  def getSerializedMapOutputStatuses(shuffleId: Int): Array[Byte] = {
		var statuses: Array[MapStatus] = null
		var retBytes: Array[Byte] = null
		var epochGotten: Long = -1
		
		shuffleIdLock.synchronized {
		  
		  val (bytes, bcast) = MapOutputTracker.serializeMapStatuses(statuses, broadcastManager,
			isLocal, minSizeForBroadcast)
		  // Add them into the table only if the epoch hasn't changed while we were working
		  epochLock.synchronized {
			if (epoch == epochGotten) {
			  cachedSerializedStatuses(shuffleId) = bytes
			  if (null != bcast) cachedSerializedBroadcast(shuffleId) = bcast
			} else {
			  logInfo("Epoch changed, not caching!")
			  removeBroadcast(bcast)
			}
		  }
		  bytes
		}
	  }
	}
Master�����ݱȽ϶࣬����ֻȡ��3���Ƚ���Ҫ�ĺ�����`registerShuffle`��`registerMapOutputs`���Ƿֱ�ע��ShuffleId�Ͷ�Ӧ��
MapOutput��Ϣ��`getSerializedMapOutputStatuses`���������л�MapOutput��Ϣ������ɾȥ�˴�cache�˵���Ϣ�в��ҵĹ��̣���
����`registerShuffle`��`getSerializedMapOutputStatuses`���ڴ���ShuffleMapStage��ʱ����õģ���֮ǰ�ᵽ��
`createShuffleMapStage`������`registerMapOutputs`���������������õģ���`handleTaskCompletion`�����С�

	private[scheduler] def handleTaskCompletion(event: CompletionEvent) {
		...
		event.reason match {
		  case Success =>
			stage.pendingPartitions -= task.partitionId
			task match {
			  ...
			  case smt: ShuffleMapTask =>
				val shuffleStage = stage.asInstanceOf[ShuffleMapStage]
				updateAccumulators(event)
				val status = event.result.asInstanceOf[MapStatus]
				val execId = status.location.executorId
				if (failedEpoch.contains(execId) && smt.epoch <= failedEpoch(execId)) {
				  ...
				} else {
				  shuffleStage.addOutputLoc(smt.partitionId, status)
				}

				if (runningStages.contains(shuffleStage) && shuffleStage.pendingPartitions.isEmpty) {
				  markStageAsFinished(shuffleStage)
				  
				  mapOutputTracker.registerMapOutputs(
					shuffleStage.shuffleDep.shuffleId,
					shuffleStage.outputLocInMapOutputTrackerFormat(),
					changeEpoch = true)

				  clearCacheLocs()

				  if (!shuffleStage.isAvailable) {
					submitStage(shuffleStage)
				  } else {
					// Mark any map-stage jobs waiting on this stage as finished
					if (shuffleStage.mapStageJobs.nonEmpty) {
					  val stats = mapOutputTracker.getStatistics(shuffleStage.shuffleDep)
					  for (job <- shuffleStage.mapStageJobs) {
						markMapStageJobAsFinished(job, stats)
					  }
					}
					submitWaitingChildStages(shuffleStage)
				  }
				}
			}
		}
	}
RPCͨ�ŵĹ�������MapOutputTrackerMasterEndpoint����ʵ�ֵģ����ṩ��һ����Ϣ�շ��Ĵ���ʽ����ΪMapOutputTracker
���������һ��MapOutputTrackerMasterEndpoint���͵ĳ�Ա����`trackerEndpoint`�����Կ��Խ��俴����һ��������ڡ�
����ڵ���������SparkEnv�е�create������ע����ɵģ���`createDriverEnv`��`createExecutorEnv`���ã�������Executor��
���RPC��ȥ���ݵģ�����ͨ��MapOutputTracker�е�`getStatuses(shuffleId: Int)`����ʵ�ֵģ�������̲���׸����

#####Shuffle����

Map����ʲô���������������ShuffleManager�����ġ�

	private[spark] class BaseShuffleHandle[K, V, C](
		shuffleId: Int,
		val numMaps: Int,
		val dependency: ShuffleDependency[K, V, C])
	  extends ShuffleHandle(shuffleId)
	  
	private[spark] trait ShuffleManager {
	  
	  def registerShuffle[K, V, C](
		  shuffleId: Int,
		  numMaps: Int,
		  dependency: ShuffleDependency[K, V, C]): ShuffleHandle
		  
	  def getWriter[K, V](handle: ShuffleHandle, mapId: Int, context: TaskContext): ShuffleWriter[K, V]
	  
	  def getReader[K, C](
		  handle: ShuffleHandle,
		  startPartition: Int,
		  endPartition: Int,
		  context: TaskContext): ShuffleReader[K, C]
		  
	  def unregisterShuffle(shuffleId: Int): Boolean
	  
	  def shuffleBlockResolver: ShuffleBlockResolver
	  
	  def stop(): Unit
	}
���ȿ������ShuffleHandle��ʵ��, ��ֻ��һ��shuffleId, numMaps��ShuffleDep�ķ�װ; �ٿ�ShuffleManager�ṩ�Ľӿڡ�

+registerShuffle/unregisterShuffle���ṩ��Shuffle��ע���ע���Ĺ��ܣ�������̸����MapOutputTrackerһ�£�Ȼ��
����һ��ShuffleHandle��������shuffle���з�װ��
+getWriter��MapTask���ã�����������ݡ�
+getReader��Reduce���̽��е��ã���ShuffleRDD���ã����ڶ�ȡMap��������ݡ�������������ʼλ�ã�����Ϊÿ��Map�����
�ļ���ʵֻ��һ����ֻ����Բ�ͬ��Reduce�����ƫ������ͬ���ⲿ����[Spark��shuffle���Ʒ���][1]���ۡ�

���й����Ѿ��������ShuffleMapTask��runTask�н��ܹ��ˡ�

####Shuffle Reduceʵ��

Reduce��Ӧ��Ӧ����֮ǰ����ResultTask��runTask�����ݣ����в�����RDD������CoGroupedRDD��CustomShuffledRDD��ShuffledRDD��
ShuffledRowRDD��SubtractedRDD����ShuffleRDDΪ������compute������ͬ��һ��ķ�Shuffle RDD��

	override def compute(split: Partition, context: TaskContext): Iterator[(K, C)] = {
		val dep = dependencies.head.asInstanceOf[ShuffleDependency[K, V, C]]
		SparkEnv.get.shuffleManager.getReader(dep.shuffleHandle, split.index, split.index + 1, context)
		  .read()
		  .asInstanceOf[Iterator[(K, C)]]
	}

[1]:https://github.com/summerDG/spark-code-ananlysis/blob/master/analysis/spark_sort_shuffle.md
#Spark��������Ȼ���
##��TaskScheduler��ʼ
TaskScheduler����Ҫ���þ��ǻ����Ҫ��������񼯺ϣ������䷢�͵���Ⱥ���д������һ��л㱨��������״̬�����á�
����������Master�ˡ�����������4�����ã�

* ��������Executor��������Ϣ��ʹMaster֪����Executer��BlockManager�������š���
* ����ʧ�ܵ�����������ԡ�
* ����stragglers���Ϻ��ȵ����񣩷ŵ������Ľڵ�ִ�С�
* ��Ⱥ�ύ���񼯣�������Ⱥ���С�

###TaskScheduler����
���ڵ�������������SparkContext�н��д����ġ�

    // SparkContext��ʼ������
    val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode)
    _schedulerBackend = sched
    _taskScheduler = ts
`createTaskScheduler(this, master, deployMode)`�����ò���ʽ���������TaskScheduler���������Ƿ�����һ��`sched`
������SchedulerBackend���ͣ������ͱ�ʾ��ͬ�ĺ�˵��ȣ�ʲô�Ǻ�˵��ȣ����磺Local��Standalone��Yarn��Mesos�ȡ�
�����ⲿ�ģ���Standalone��Ӧ���ĺ�ˣ��䶼�̳���CoarseGrainedSchedulerBackend������������ֻ�Ǹ������ȵĺ�ˣ������
ϸ���ȵ�ʵ���ڲ�ͬ���ϵͳģ����ʵ�֡������ⲿ�������ͺ�˵�����������ClusterManager����ɵģ��ⲿ��ClusterManager
���̳���ExternalClusterManager��ͨ��ExternalClusterManager��`createTaskScheduler`��`createSchedulerBackend`�����
��TaskScheduler�ͻ���SchedulerBackend�����ɡ����������mesosΪ����ȥmesosģ����ȥ��ʵ�֡�
����TaskScheduler������Ҫ��ʵ�־���TaskSchedulerImpl��Localģʽ��Standaloneģʽ��Mesosģʽ�����õ����ʵ�֡������о�
������Ϸ�ʽ��

* Localģʽ��TaskSchedulerImpl+LocalSchedulerBackend
* Spark Standalone��Ⱥģʽ��TaskSchedulerImpl+StandaloneSchedulerBackend
* Mesos��Ⱥģʽ��TaskSchedulerImpl+MesosFineGrainedSchedulerBackend��MesosCoarseGrainedSchedulerBackend
* Yarn��Ⱥģʽ��YarnClusterScheduler+YarnClusterSchedulerBackend

TaskScheduler��SchedulerBackend��������֮�󣬻�����`scheduler.initialize(backend)`���������ס�

�����и�����û�д�������֮ǰ˵��TaskScheduler�л����������Ϣ����ôͨ�Ź�����������ʵ�ֵ��أ�����HeartbeatReceiver�У�
HeartbeatReceiver��һ��SparkListener��������һ��RPCͨ�ŵ���ڣ�����`receiveAndReply`�л���

	case TaskSchedulerIsSet =>
      scheduler = sc.taskScheduler
      context.reply(true)

��佫��TaskScheduler���г�ʼ������ô�����Ľ����Լ������ʵ���ˡ�ͬʱ���ڶ�ʧExecutor�Ĵ���Ҳ����HeartbeatReceiver����ɵġ�

###TaskScheduler����

TaskScheduler������Ҳ����SparkContext��ʼ��������ɵġ���TaskSchedulerImpl��StandaloneSchedulerBackendΪ�����������������̡�
TaskScheduler�������������Ȼ�����`backend`��������һ��`speculationScheduler`�ػ��̣߳����ڼ�⵱ǰJob��speculatable tasks��
speculatable tasks��Ӧ�ó����ǣ����һ������ִ��ʱ���������ô�ͽ����ж�Ϊspeculatable tasks���Ϻ����ˣ���Ȼ��������������
����һ̨�����ϣ���ʱͬһ�������������ִ�У�����һ̨�����ɹ�����֮�󣬻�֪ͨ����һ̨������������

����StandaloneSchedulerBackend��`start`����������ͨ����������Dirver��rpc���ʵ㡣SchedulerBackend���������£�

* ��cluster manager���������µ�Executor����ر��ض���Executor��
* �ر��ض�Executor�ϵ��ض�����
* �ռ�Worker��Ϣ����������ĳ��Դ�����������Դ������CPU�������ȡ�

TaskScheduler����ͨ���ռ���`backend`�ռ�����Դ��Ϣ������������������Դ���䡣������`backend`��ʼ����ʱ�����Ѿ����ò�ͬ
�ĵ���ģʽ�����˶�Ӧ������أ�Pool�������Խ�������Ϊһ����ѭ�ض������㷨������

###TaskScheduler�ύ����

����TaskScheduler�ĺ����ܶ࣬����Ϊ��Ҫ�ıϾ����ύ�������Ⱥ���д���

	override def submitTasks(taskSet: TaskSet) {
		val tasks = taskSet.tasks
		this.synchronized {
		  val manager = createTaskSetManager(taskSet, maxTaskFailures)
		  val stage = taskSet.stageId
		  val stageTaskSets =
			taskSetsByStageIdAndAttempt.getOrElseUpdate(stage, new HashMap[Int, TaskSetManager])
		  stageTaskSets(taskSet.stageAttemptId) = manager
		  val conflictingTaskSet = stageTaskSets.exists { case (_, ts) =>
			ts.taskSet != taskSet && !ts.isZombie
		  }
		  schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)

		  ...
		backend.reviveOffers()
	}

����ע�⵽�������ÿ�����񼯣�һ��Stage�������������ɶ�Ӧ��TaskSetManager����������ÿ�����������ʧ�ܻ�
�������ԣ��Լ����ڸ�����ͨ���ӳٵ�����ʹ�価����֤�Ǳ��ص��ȡ��������Ҫ��������`resourceOffer`������Ҫ���ж�
task��locality�����յ���`addRunningTask`��������е����񣬲�����DAGScheduler����������������Դ�룺

	def resourceOffer(
		  execId: String,
		  host: String,
		  maxLocality: TaskLocality.TaskLocality)
		: Option[TaskDescription] =
	  {
		if (!isZombie) {
		  val curTime = clock.getTimeMillis()

		  var allowedLocality = maxLocality

		  dequeueTask(execId, host, allowedLocality) match {
			case Some((index, taskLocality, speculative)) =>
			  val task = tasks(index)
			  val taskId = sched.newTaskId()
			  copiesRunning(index) += 1
			  val attemptNum = taskAttempts(index).size
			  val info = new TaskInfo(taskId, index, attemptNum, curTime,
				execId, host, taskLocality, speculative)
			  taskInfos(taskId) = info
			  taskAttempts(index) = info :: taskAttempts(index)
			  
			  // Serialize and return the task
			  val startTime = clock.getTimeMillis()
			  val serializedTask: ByteBuffer = try {
				Task.serializeWithDependencies(task, sched.sc.addedFiles, sched.sc.addedJars, ser)
			  } catch {
				  ...
			  }
			  addRunningTask(taskId)

			  sched.dagScheduler.taskStarted(task, info)
			  return Some(new TaskDescription(taskId = taskId, attemptNumber = attemptNum, execId,
				taskName, index, serializedTask))
			case _ =>
		  }
		}
		None
	}

���������TaskLocality����������ζ�����أ���ʵ��һ��ö���࣬����ΪPROCESS_LOCAL��NODE_LOCAL��NO_PREF��
RACK_LOCAL��ANY���ֱ��ʾͬ�������ͬ��node(������)�ϣ�ͬ���ܣ����⣬���������ȼ��ѴﵽС��`dequeueTask`
��ʹ����ӵȴ�����������г��У����������������locality����Ϊ������Executor Id�ͽڵ�Id�����Կ����жϣ�����
����speculative����Ȼ�����ɶ�Ӧ��������Ϣ����������������л���������������������е����񼯺���֮��ͨ��
DAGScheduler���������񡣵����������������ָ����Driver����Ϊ�������ˡ�������Slave�����ͨ�ŵ��أ���ʵ���õ�
����`backend`����������һ����������

`submitTasks`�д���TaskSetManager��֮��Ὣ�����`schedulableBuilder`�У���ʵ����`schedulableBuilder`��`rootPool`
�С�����TaskSetManager�����ύ�ˣ���ʲôʱ��Żᴥ��ִ�в����أ�

˳��`rootPool`->`TaskSchedulerImpl.resourceOffers`->`CoarseGrainedSchedulerBackend.launchTasks`->`CoarseGrainedSchedulerBackend.makeOffers`->`CoarseGrainedSchedulerBackend.receive`��

	//CoarseGrainedSchedulerBackend
	override def receive: PartialFunction[Any, Unit] = {
		  case StatusUpdate(executorId, taskId, state, data) =>
			scheduler.statusUpdate(taskId, state, data.value)
			if (TaskState.isFinished(state)) {
			  executorDataMap.get(executorId) match {
				case Some(executorInfo) =>
				  executorInfo.freeCores += scheduler.CPUS_PER_TASK
				  makeOffers(executorId)
				  ...
			  }
			}

		  case ReviveOffers =>
			makeOffers()
		...
	}
	
�������������״̬����֮������յ�`ReviveOffers`��Ϣ֮�������Ϣ��ͨ��`reviveOffers`�������͵ģ�����������`submitTasks`
�����һ�䷢����`backend.reviveOffers()`������Ҳ������ʱ������������Դ�Ĳ�����

����`makeOffers`

	//CoarseGrainedSchedulerBackend
	private def makeOffers() {
		  // Filter out executors under killing
		  val activeExecutors = executorDataMap.filterKeys(executorIsAlive)
		  val workOffers = activeExecutors.map { case (id, executorData) =>
			new WorkerOffer(id, executorData.executorHost, executorData.freeCores)
		  }.toSeq
		  launchTasks(scheduler.resourceOffers(workOffers))
	}

���Ƚ���Ծ�Ľڵ㼰����Ŀ��к�������Ϣ��ȡ������������Դ������������Ϣ������`launchTasks`���ȷ�����Դ����Ĺ��̡�

	//TaskSchedulerImpl
	def resourceOffers(offers: Seq[WorkerOffer]): Seq[Seq[TaskDescription]] = synchronized {
		
		// Randomly shuffle offers to avoid always placing tasks on the same set of workers.
		val shuffledOffers = Random.shuffle(offers)
		// Build a list of tasks to assign to each worker.
		val tasks = shuffledOffers.map(o => new ArrayBuffer[TaskDescription](o.cores))
		val availableCpus = shuffledOffers.map(o => o.cores).toArray
		val sortedTaskSets = rootPool.getSortedTaskSetQueue

		var launchedTask = false
		for (taskSet <- sortedTaskSets; maxLocality <- taskSet.myLocalityLevels) {
		  do {
			launchedTask = resourceOfferSingleTaskSet(
				taskSet, maxLocality, shuffledOffers, availableCpus, tasks)
		  } while (launchedTask)
		}

		if (tasks.size > 0) {
		  hasLaunchedTask = true
		}
		return tasks
	}
	
����Ĵ���ֻ�����˺��Ĵ��룬���ȸ÷�����Կ��õ�Worker����˳����Ϊ��ɱ���ÿ�ζ���������������ǰ���Worker��
Ȼ�����ɶ�Ӧ������������Ϣ������`rootPool`�еĵ��ȷ�ʽ�������������Ȼ�����`resourceOfferSingleTaskSet`���
ÿ������������Դ��

	private def resourceOfferSingleTaskSet(
		  taskSet: TaskSetManager,
		  maxLocality: TaskLocality,
		  shuffledOffers: Seq[WorkerOffer],
		  availableCpus: Array[Int],
		  tasks: Seq[ArrayBuffer[TaskDescription]]) : Boolean = {
		var launchedTask = false
		for (i <- 0 until shuffledOffers.size) {
		  val execId = shuffledOffers(i).executorId
		  val host = shuffledOffers(i).host
		  if (availableCpus(i) >= CPUS_PER_TASK) {
			try {
			  for (task <- taskSet.resourceOffer(execId, host, maxLocality)) {
				tasks(i) += task
				...
			  }
			} catch {
			  ...
			}
		  }
		}
		if (!launchedTask) {
		  taskSet.abortIfCompletelyBlacklisted(executorIdToHost.keys)
		}
		return launchedTask
	}
	
�÷�����Ҫ�ǵ��������ᵽ��TaskSetManager��`resourceOffer`�����Ի᷵�����л����������Ϣ��Ȼ����뵽`tasks`�У�
����TaskSchedulerImpl��`resourceOffers`Ҳ����ˡ���������`launchTasks`�Ĵ�����̡�

	//CoarseGrainedSchedulerBackend
	private def launchTasks(tasks: Seq[Seq[TaskDescription]]) {
		  for (task <- tasks.flatten) {
			val serializedTask = ser.serialize(task)
			if (serializedTask.limit >= maxRpcMessageSize) {
			  ...
			}
			else {
			  val executorData = executorDataMap(task.executorId)
			  ...
			  executorData.executorEndpoint.send(LaunchTask(new SerializableBuffer(serializedTask)))
			}
		  }
	}

����ȡ�����л��������Ϣ��Ȼ��ȡ��������������Executor��Ȼ�����Executor���������������Ϣ������Ϣ��Executor�˵�
CoarseGrainedExecutorBackend���յ�

	//CoarseGrainedExecutorBackend
	override def receive: PartialFunction[Any, Unit] = {
		...
		case LaunchTask(data) =>
		  if (executor == null) {
			exitExecutor(1, "Received LaunchTask command but executor was null")
		  } else {
			val taskDesc = ser.deserialize[TaskDescription](data.value)
			executor.launchTask(this, taskId = taskDesc.taskId, attemptNumber = taskDesc.attemptNumber,
			  taskDesc.name, taskDesc.serializedTask)
		  }
		...
	}
	
�յ������������Ϣ�󣬻��Ƚ���������Ϣ�ķ����л���Ȼ���`executor`��������Executor��һ���߳���ɡ�

	//Executor
	def launchTask(
		  context: ExecutorBackend,
		  taskId: Long,
		  attemptNumber: Int,
		  taskName: String,
		  serializedTask: ByteBuffer): Unit = {
		val tr = new TaskRunner(context, taskId = taskId, attemptNumber = attemptNumber, taskName,
		  serializedTask)
		runningTasks.put(taskId, tr)
		threadPool.execute(tr)
	}

�÷�������̳߳���ѡ������߳���������

	//ThreadPoolExecutor
	public void execute(Runnable command) {
	
			int c = ctl.get();
			if (workerCountOf(c) < corePoolSize) {
				if (addWorker(command, true))
					return;
				c = ctl.get();
			}
			if (isRunning(c) && workQueue.offer(command)) {
				int recheck = ctl.get();
				if (! isRunning(recheck) && remove(command))
					reject(command);
				else if (workerCountOf(recheck) == 0)
					addWorker(null, false);
			}
			else if (!addWorker(command, false))
				reject(command);
	}

��1���жϱ�ʾ������������е��߳���С�ڸýڵ�ĺ�����������һ���߳���ִ�����񡣵�2�������ʾ�������������Գɹ�����
���У��������ʱ���Ѿ�������̣߳����ܴ��ڵ�һ�μ��֮����߳̽���������������÷���֮���̳߳��Ƿ񱻹ر��ˡ���
�������Ǽ����޷�����������У���ô�;���Ϊ�䴴���̣߳����ʧ�ܣ��ͷ����������񡣵�3������Ϊ���񴴽��̣߳����Խ���
`addWorker`��

	private boolean addWorker(Runnable firstTask, boolean core) {
			retry:
			...

			boolean workerStarted = false;
			boolean workerAdded = false;
			Worker w = null;
			try {
				final ReentrantLock mainLock = this.mainLock;
				w = new Worker(firstTask);
				final Thread t = w.thread;
				if (t != null) {
					mainLock.lock();
					try {
						int c = ctl.get();
						int rs = runStateOf(c);

						if (rs < SHUTDOWN ||
							(rs == SHUTDOWN && firstTask == null)) {
							if (t.isAlive()) // precheck that t is startable
								throw new IllegalThreadStateException();
							workers.add(w);
							int s = workers.size();
							if (s > largestPoolSize)
								largestPoolSize = s;
							workerAdded = true;
						}
					} finally {
						mainLock.unlock();
					}
					if (workerAdded) {
						t.start();
						workerStarted = true;
					}
				}
			} finally {
				if (! workerStarted)
					addWorkerFailed(w);
			}
			return workerStarted;
	}

ɾ�����ִ��룬�Ȼ�ȡ���̵߳�����״̬��û�йرջ���û������������£�����������뵽�߳��б�Ȼ���������̣߳�
������������������Worker������к���ʵ������`firstTask`�����к�������TaskRunner������к��������Խ���ú���һ
̽���������ڴ���̫�࣬�����������һ�£��������и������ȡ����Ľ��������ResultTask����ShuffleMapTask��`run`�����������[Spark������Shuffleʵ��][1]����
Ȼ��������л������������佻��`ExecutorBackend`������CoarseGrainedExecutorBackend.statusUpdate������`ExecutorBackend`
��������ظ�Driver�˵�SchedulerBackend��CoarseGrainedSchedulerBackend.receive�������յ������״̬����`statusUpdate`
������

def statusUpdate(tid: Long, state: TaskState, serializedData: ByteBuffer) {
    var failedExecutor: Option[String] = None
    synchronized {
      try {
        ...
        taskIdToTaskSetManager.get(tid) match {
          case Some(taskSet) =>
            if (TaskState.isFinished(state)) {
              ...
            }
            if (state == TaskState.FINISHED) {
              taskSet.removeRunningTask(tid)
              taskResultGetter.enqueueSuccessfulTask(taskSet, tid, serializedData)
            } else if (Set(TaskState.FAILED, TaskState.KILLED, TaskState.LOST).contains(state)) {
              ...
            }
          case None =>
            ...
        }
	  }
	  ...
	}
	...
}

�������������ɹ������Ĵ�����������Է����������Ƴ��������ţ�Ȼ��ȡ����������

	def enqueueSuccessfulTask(
		  taskSetManager: TaskSetManager,
		  tid: Long,
		  serializedData: ByteBuffer): Unit = {
		getTaskResultExecutor.execute(new Runnable {
		  override def run(): Unit = Utils.logUncaughtExceptions {
			try {
			  val (result, size) = serializer.get().deserialize[TaskResult[_]](serializedData) match {
				case directResult: DirectTaskResult[_] =>
				  if (!taskSetManager.canFetchMoreResults(serializedData.limit())) {
					return
				  }
				  
				  directResult.value()
				  (directResult, serializedData.limit())
				case IndirectTaskResult(blockId, size) =>
				  ...
			  }

			  ...

			  scheduler.handleSuccessfulTask(taskSetManager, tid, result)
			} catch {
			  ...
			}
		  }
		})
	}

ȡ���������󣬽���TaskScheduler������ɹ���������ʵ�ǵ���TaskSetManager��`handleSuccessfulTask`�������÷���
����DAGScheduler��`taskEnded`����������CompletionEvent���͵���Ϣ�������֮����TaskSetManager��DAGScheduler������Ϣ����
DAGScheduler�յ�����Ϣ��ͻ����`handleTaskCompletion`�������д�����[Spark������Shuffleʵ��][1]����

����������ύ������������̾���ͨ�ˡ�

[1]:https://github.com/summerDG/spark-code-ananlysis/blob/master/analysis/spark_shuffle.md
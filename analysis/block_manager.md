#Spark Block Manager����

Spark�е�RDD-Cache��broadcast��ShuffleWriter��ExternalSorter�ȶ��ǻ���BlockManagerʵ�ֵġ�BlockManager��������ÿ���ڵ��ϣ�
����driver��executor������Ҫ���ṩ�ӿ����������ִ洢��memory��disk��off-heap���б��ػ�Զ�̵�block��

##BlockManager����

BlockManager����SparkEnv�д������ɵġ������һ��SparkContext��SparkEnv������ǰ���Ǵ���������Sparkϵͳ�Ļ����������ߵ�������
�����������ÿ�����е�Sparkʵ�壨master��worker������������ʱ�������������л�����RpcEnv��block manager��map output tracker�ȣ���
��ȻSparkEnvҲ����SparkContext�е�`create`�д����ġ�

�����Block��HDFS��̸����Block�����б�������HDFS���ǶԴ��ļ����з�Block���д洢��Block��С�̶�Ϊ512M�ȣ���Spark�е�Block���û�
�Ĳ�����λ�� һ��Block��Ӧһ������֯���ڴ棬һ���������ļ����ļ�������ˣ���û�й̶�ÿ��Block��С��������

	private[spark] class BlockManager(
		executorId: String,
		rpcEnv: RpcEnv,
		val master: BlockManagerMaster,
		serializerManager: SerializerManager,
		val conf: SparkConf,
		memoryManager: MemoryManager,
		mapOutputTracker: MapOutputTracker,
		shuffleManager: ShuffleManager,
		val blockTransferService: BlockTransferService,
		securityManager: SecurityManager,
		numUsableCores: Int)
	  extends BlockDataManager with BlockEvictionHandler with Logging {
	}
	
��BlockManager�Ĺ��캯�������Է�������Ҫ�����ˣ�executorId��rpcEnv��blockManagerMaster��serializerManager��SparkConf��
[memoryManager][3]��[mapOutputTracker][1]��[shuffleManager][2]��blockTransferService������һ����ȡһ��blocks����
securityManager��numUsableCores�����ú�������

##Block���֪ʶ

����̸����Block���û��Ĳ�����λ�������������Ӧ��key��������BlockID��value����ΪManagerBuffer����һ��BlockDataManager
������ʣ��ӿڣ���BlockManager�ͼ̳��������������ṩ�˶�Block�Ĳ�������ȡ������ӡ�

	trait BlockDataManager {
	  def getBlockData(blockId: BlockId): ManagedBuffer

	  def putBlockData(
		  blockId: BlockId,
		  data: ManagedBuffer,
		  level: StorageLevel,
		  classTag: ClassTag[_]): Boolean

	  def releaseLock(blockId: BlockId): Unit
	}

`getBlockData`�Ǵ�BlockManager�л�ȡ��ӦId��ManagerBuffer��`putBlockData`�ǽ�һ��ManagerBuffer����Id���մ洢�㼶��ӵ�
BlockManager�С�

*BlockId*�䱾������˵����һ���ַ�����ֻ����Բ�ͬ��Block������ʽ��ͬ��*ManagerBuffer*ʵ�����Ƕ�ByteBuffer�ķ�װ��

	public abstract class ManagedBuffer {

	  public abstract long size();
	  
	  public abstract ByteBuffer nioByteBuffer() throws IOException;
	  
	  public abstract InputStream createInputStream() throws IOException;

	  public abstract ManagedBuffer retain();
	  
	  public abstract ManagedBuffer release();
	  
	  public abstract Object convertToNetty() throws IOException;
	}

����Buffer���ܳ��ô������ڴ��в������ͷţ�����������`retain`��`release`���������ֱ�������ӻ��ͷŶԸ�Buffer��������Ŀ���Ը�Buffer
�Ķ�����ʽӿھ���`createInputStream`��`nioByteBuffer`��ǰ���ǽ�buffer�е�������ΪInputStream��¶��������������Ϊ
NIO ByteBuffer��¶��ע�����nioByteBuffer������ÿ�ε��ý��᷵��һ���µ�ByteBuffer,�����Ĳ�����Ӱ�� ��ʵ��Buffer��offset��long��
����ӿ���3��ʵ�֣�FileSegmentManagedBuffer��NettyManagedBuffer��NioManagedBuffer��BlockManagerManagedBuffer���������Ϳ���֪������������
��ȡByteBuffer����Դ��ͬ��FileSegmentManagedBuffer�Ǳ�����һ��File���͵ı����������Ƕ�ȡ`file`�����������ByeBuffer��
NettyManagedBuffer��ͨ��ByteBuf��ȡ�ģ�NioManagedBuffer�����ֱ�Ӿ���ByteBuffer��BlockManagerManagedBuffer��ͨ��ChunkedByteBuffer��spark.util.io����

##Block״̬ά��

��������StorageLevel����Spark�У���ӦRDD��cache�кܶ�levelѡ������̸����StorageLevel����������ݡ��������������洢�ļ���

* DISK�����ļ��洢��
* Memory���ڴ�洢,������ڴ�ָ����Jvm�еĶ��ڴ�,��onHeap��
* OffHeap����JVM��Heap���ڴ�洢������˼��*���л�*��cache��off-heap�ڴ档

����DISK��Memory���ּ����ǿ���ͬʱ���ֵġ�

����OffHeap���Ͼ���ʵ���������Ʒ�������˵���䣺JVM�����ֱ�ӽ����ڴ���䶼����JVM������ʹ�õ���JVM���ڴ�ѣ����������к�
�༼��������JVM�����з��ʲ���JVM������ڴ棬��OffHeap�ڴ档OffHeap���ĺô����ǽ��ڴ�Ĺ�������JVM��GC����������������Լ�
���й����ر��Ǵ�����Զ����������ڵĶ�����˵OffHeap��ʵ�ã����Լ���GC�Ĵ���������ʵ���ǻ���Spark��Off-heapʵ�ֵģ�˵������
ͨ��Javaֱ�ӵ�JVM֮����ڴ棬��ȥ�ǻ���Alluxio����ǰTachyon�������ģ���SPARK-12667�б��Ƴ��ˣ��������������[SPARK-13992][4]����

StorageLevel���ṩ���������������

* _deserialized���Ƿ���Ҫ�����л���
* _replication��������Ŀ��

�洢����BlockManager�����Ǹ��ֶ����Ƿ�֧�����л�Ӱ���˶��������ķ����Լ��ڴ��ѹ����`_deserialized`�����Ϊ`true`֮�󣬾Ͳ���
�����ݽ������л��ˡ�

***

����BlockManager��ÿ��`putBlockData`��һ������ɹ���ÿ��`getBlockData`��һ���������Ͽ��Է��ؽ������Ϊput�еȴ��Ĺ��̣����ҿ��������ʧ�ܡ�

BlockManagerͨ��BlockInfo��ά��ÿ��Block��״̬����BlockManager��`blockInfoManager`��BlockInfoManager���ͣ�������BlockId��BlockInfo��ӳ�䣬
BlockInfoManager��ʵ���Ƕ�ӳ������һ���װ��ֻ�������������������д���Ͷ���������֤����д��ͬ����д����Ҫ�ڶ���ʱ��д����
��������һ��`putBlockData`����ʵ�������յ��õ���`doPutBytes`

	private def doPutBytes[T](
		  blockId: BlockId,
		  bytes: ChunkedByteBuffer,
		  level: StorageLevel,
		  classTag: ClassTag[T],
		  tellMaster: Boolean = true,
		  keepReadLock: Boolean = false): Boolean = {
		doPut(blockId, level, classTag, tellMaster = tellMaster, keepReadLock = keepReadLock) { info =>
		  ...

		  val size = bytes.size

		  if (level.useMemory) {
			val putSucceeded = if (level.deserialized) {
			  val values =
				serializerManager.dataDeserializeStream(blockId, bytes.toInputStream())(classTag)
			  memoryStore.putIteratorAsValues(blockId, values, classTag) match {
				case Right(_) => true
				case Left(iter) =>
				  iter.close()
				  false
			  }
			} else {
			  memoryStore.putBytes(blockId, size, level.memoryMode, () => bytes)
			}
			if (!putSucceeded && level.useDisk) {
			  diskStore.putBytes(blockId, bytes)
			}
		  } else if (level.useDisk) {
			diskStore.putBytes(blockId, bytes)
		  }

		  val putBlockStatus = getCurrentBlockStatus(blockId, info)
		  val blockWasSuccessfullyStored = putBlockStatus.storageLevel.isValid
		  ...
		}.isEmpty
	}

	private def doPut[T](
		  blockId: BlockId,
		  level: StorageLevel,
		  classTag: ClassTag[_],
		  tellMaster: Boolean,
		  keepReadLock: Boolean)(putBody: BlockInfo => Option[T]): Option[T] = {

		val putBlockInfo = {
		  val newInfo = new BlockInfo(level, classTag, tellMaster)
		  if (blockInfoManager.lockNewBlockForWriting(blockId, newInfo)) {
			newInfo
		  } else {
			...
			return None
		  }
		}
		
		var blockWasSuccessfullyStored: Boolean = false
		val result: Option[T] = try {
		  val res = putBody(putBlockInfo)
		  blockWasSuccessfullyStored = res.isEmpty
		  res
		} finally {
		  ...
		}
		...
		result
	}
	
`doPutBytes`��Ҫ������`doPut`����ɵģ�`doPutBytes`�ṩ��BlockInfo�Ĵ�������Block�ľ���洢����`doPut`������
����BlockInfo��`doPutBytes`�л����ж�Block�Ĵ洢�㼶��Ȼ������Ƿ���Ҫ���л������������þ����BlockStore��`memoryStore`
��`diskStore`��[֮������](##BlockStore)����`putBytes`����ʵ��Block�Ĵ洢������Ĵ���ʡ������Ը�����ע��ȹ��̡�

BlockInfo�е�Block��״̬��ͨ���������Ƶģ���ͨ����ȡ��ǰ��Block��reader����������ʱ��д����writer��д��ʱ����������ƶ�дͬ����
ͨ����֤ͬʱֻ��һ��writer����֤����д��Block�Ƿ�洢�ɹ���ͨ��`doPutBytes`�е�`blockWasSuccessfullyStored`���������Ƶģ�
����ͨ��`getCurrentBlockStatus`����ȡ��ӦId��Block�Ĵ洢״̬�����Ǹ�Block�ڴ���˶��٣����̴��˶��٣��Լ�ʲô�洢�㼶�����жϵġ�

##BlockStore

BlockStore��Block�����Ĵ洢��������Spark�У�BlockStore�Ȳ���һ������Ҳ����һ���࣬����ֻ������ͳ�ơ�����ΪMemoryStore��DiskStore
�������͡�

###MemoryStore

MemoryStore��������ӳ���`onHeapUnrollMemoryMap`��`offHeapUnrollMemoryMap`��ǰ���Ǵ������Ե�Id��*��Unroll��*һ��Block�����ڴ��ӳ�䣬
������˼��ͬ����ͬ����ǰ����on-heapģʽ�����ߵ���off-heapģʽ����ôʲô�ǡ�Unroll������Ϊ�е�ʱ���ڴ��е�����Ҳ�����
���л��洢�Լ��ٶ��ڴ�����ģ��������ջ���Ҫ�����л����������л�֮��ռ�õĵ��ڴ�С�������л�֮���Ȼ�����ͣ���ô��ҪԤ����һ�����ڴ�
����֤�����л�֮��������д洢�ռ䣬�������ݷ����л�֮��������ڴ���ǡ�Unroll���ڴ棨[���˵���⣬������Spark1.3][5]��������һ���������Block������һ����ȫ�������ڴ�ģ�����
��������ʽһ����һ���ֶ���ģ�����Ҳ��Ҫһ�����ڴ����ڰ�Block�е����ݡ�չ�������Լ�����⣩����ô���ܱ�֤�������OOM�أ��������������Եؼ���Ƿ����㹻
�Ŀ����ڴ棬������Խ�ѹ�����ڡ�ת�������ڴ��ڼ�ʹ����ʱ�ġ�Unroll���ڴ档

����������ô�洢���ݣ�����̫���������֮�����ڽ����ݴ���memoryʱҪ����Ƿ����㹻���ڴ棬Ϊʲô���л���Ҫ�����ڴ��أ�����Ӧ����Blockû��һ����
��Unroll����һ��ʼĬ�ϵĸ�һ�����ڴ�����Block�ġ�Unroll������Unroll���������ٲ��������ڴ档
˳��`MemoryStore.reserveUnrollMemoryForThisTask`->`UnifiedMemoryManager.acquireUnrollMemory`->`UnifiedMemoryManager.acquireUnrollMemory`��һ�������ڴ�ĺ�����

	override def acquireStorageMemory(
		  blockId: BlockId,
		  numBytes: Long,
		  memoryMode: MemoryMode): Boolean = synchronized {
		assertInvariants()
		assert(numBytes >= 0)
		val (executionPool, storagePool, maxMemory) = memoryMode match {
		  case MemoryMode.ON_HEAP => (
			onHeapExecutionMemoryPool,
			onHeapStorageMemoryPool,
			maxOnHeapStorageMemory)
		  case MemoryMode.OFF_HEAP => (
			offHeapExecutionMemoryPool,
			offHeapStorageMemoryPool,
			maxOffHeapMemory)
		}
		if (numBytes > maxMemory) {
		  return false
		}
		if (numBytes > storagePool.memoryFree) {
		  val memoryBorrowedFromExecution = Math.min(executionPool.memoryFree, numBytes)
		  executionPool.decrementPoolSize(memoryBorrowedFromExecution)
		  storagePool.incrementPoolSize(memoryBorrowedFromExecution)
		}
		storagePool.acquireMemory(blockId, numBytes)
	}
	
������UnifiedMemoryManager��������Ϊ�������off-heap��on-heap�ڴ�����룬���Է������������в�ͬ���ڴ�أ���Ȼ�����Կ���
�����ڴ棬�������Ƿ��ֻ���off-heap�������ڴ棨�����ڴ洢�������������off-heap�ڴ��ʹ�ò�����ָ���У��������ж������
�ڴ��С�Ƿ񳬹������ޣ����û������������ǰ���ڴ洢���ڴ���еĿռ��Ƿ������������������ͽ��������е��ڴ���е�һ��
���ڴ��ó������ڴ洢������StorageMemoryPool��`acquireMemory`�����������ڳ��ڴ�Ĳ�����MemoryStore��`evictBlocksToFreeSpace`
����ɵģ����������ҵ�ĳ�������滻�飨������д��ռ�У���֤û��reader��writer����ɾ�������Ӧ����Ϣ��

��MemoryStore�е�`putIteratorAsBytes`�����off-heap�ڴ�����ôʵ�ֵ��أ�

	//MemoryStore
	private[storage] def putIteratorAsBytes[T](
		  blockId: BlockId,
		  values: Iterator[T],
		  classTag: ClassTag[T],
		  memoryMode: MemoryMode): Either[PartiallySerializedBlock[T], Long] = {
		  ...
		val allocator = memoryMode match {
		  case MemoryMode.ON_HEAP => ByteBuffer.allocate _
		  case MemoryMode.OFF_HEAP => Platform.allocateDirectBuffer _
		}
		...
	}
	//Platform
	public static ByteBuffer allocateDirectBuffer(int size) {
		try {
		  Class<?> cls = Class.forName("java.nio.DirectByteBuffer");
		  Constructor<?> constructor = cls.getDeclaredConstructor(Long.TYPE, Integer.TYPE);
		  constructor.setAccessible(true);
		  Field cleanerField = cls.getDeclaredField("cleaner");
		  cleanerField.setAccessible(true);
		  final long memory = allocateMemory(size);
		  ByteBuffer buffer = (ByteBuffer) constructor.newInstance(memory, size);
		  Cleaner cleaner = Cleaner.create(buffer, new Runnable() {
			@Override
			public void run() {
			  freeMemory(memory);
			}
		  });
		  cleanerField.set(buffer, cleaner);
		  return buffer;
		} catch (Exception e) {
		  throwException(e);
		}
		throw new IllegalStateException("unreachable");
	}

����`allocateDirectBuffer`�е�`cleaner`��`freeMemory`�����˰ɡ���ʹ�������off-heap�ڴ�����ġ��������ﷵ
�ص�`allocator`�Ѿ���һ��off-heap�ڴ��ByteBuffer�ˡ�

###DiskStore

DiskStore�������ļ����洢Block������Disk���洢�����ȱ���Ҫ���һ��������Ǵ����ļ��Ĺ�������Ŀ¼�ṹ����ɣ�Ŀ¼������ȣ�
��Spark�Դ����ļ��Ĺ�����ͨ��DiskBlockManager�����й���ģ���˶�DiskStore���з���֮ǰ�����ȱ����DiskBlockManager���з�����

��Spark��������Ϣ�У�ͨ��"SPARK_LOCAL_DIRS"��������Spark���й�������ʱĿ¼���м�����Ҫǿ��һ�£�

* `SPARK_LOCAL_DIRS`���õ��Ǽ��ϣ����������ö��LocalDir����","�ֿ��������Hadoop�е���ʱĿ¼��һ���������ڶ�������д���localdir���Ӷ���ɢ���̵Ķ�дѹ����
* spark���й��������ɵ����ļ����̲��ɹ��ƣ����������׾ͻ����һ��localDir�����ļ����࣬���¶�дЧ�ʺܲ���������⣬Spark��ÿ��LocalDir��
������64����Ŀ¼������ɢ�ļ����������Ŀ¼����������ͨ��`spark.diskStore.subDirectories`�������á�

DiskBlockManagerͨ��hash���ֱ�ȷ��localDir�Լ�subdir��

	def getFile(filename: String): File = {
		// Figure out which local directory it hashes to, and which subdirectory in that
		val hash = Utils.nonNegativeHash(filename)
		val dirId = hash % localDirs.length
		val subDirId = (hash / localDirs.length) % subDirsPerLocalDir

		// Create the subdirectory if it doesn't already exist
		val subDir = subDirs(dirId).synchronized {
		  val old = subDirs(dirId)(subDirId)
		  if (old != null) {
			old
		  } else {
			val newDir = new File(localDirs(dirId), "%02x".format(subDirId))
			if (!newDir.exists() && !newDir.mkdir()) {
			  throw new IOException(s"Failed to create local dir in $newDir.")
			}
			subDirs(dirId)(subDirId) = newDir
			newDir
		  }
		}
		
		new File(subDir, filename)
	}
	
DiskBlockManager�ĺ��Ĺ���������������ṩ`getFile`�ӿڣ�����filenameȷ��һ���ļ���·����ʣ�����ľ���Ŀ¼����ȹ��������Ƚϼ�����Ͳ�������ϸ������

����DiskStore�����ǽ����л�������ݣ�����BlockManager�������л���д��DiskBlockManager��ȡ�����ļ�λ���С�

##BlockManager�ķ���ṹ

BlockManager��ɢ�ڸ����ڵ��ϣ�����Ҫ��һ��Master���ռ������ڵ��Block��Ϣ��������Ȼ����RPC���ã�ÿ���ڵ㱣��һ����Master�����ã����䷢����Ϣ����

###BlockManagerMaster����

������SparkEnv�д����ģ��书��ͨ������������һĿ��Ȼ����Ҫ�����У�

* �Ƴ���Executor��ĳRDD������Block��ĳShuffle������Block���ض�Block��ĳ�㲥����������Block��
* ��ȡÿ��BlockManager Id��������ڴ�״̬��洢״̬��
* ����ĳBlockManager����ĳBlock��״̬��
* ע��Blockmanager��
* ��ȡĳBlock��λ�á�

###BlockManagerSlaveEndpoint

ÿ��BlockManager�ж���һ��BlockManagerSlaveEndpoint���͵ı���`slaveEndpoint`���ں�Masterͨ�ţ������޷Ǿ����յ�Master����Ϣ������Ӧ�������ﲻ��׸����

###BlockTransferService

�ᷢ����BlockManager�л���һ��BlockTransferService���͵ı���`blockTransferService`���������Ǵ�Զ�������ϰ�����Ǩ�ƹ�����
��Ҫ�ĺ�����

* `fetchBlocks`����ָ����Զ�̵������Ͻ�ָ����Block��������Shuffle���̻��õ���
* `uploadBlock`���ѱ�̨�����ϵ�һ��Block���͵�ָ����Զ��������
* `fetchBlockSync`��`uploadBlockSync`������������������һ�������ǻ�������

������ļ���ȡ������OneForOneBlockFetcher��`start`��������NettyBlockTransferService��`fetchBlocks`���ã��С����Ͳ�����NettyBlockTransferService��`uploadBlock`�С�


[1]:https://github.com/summerDG/spark-code-ananlysis/blob/master/analysis/spark_shuffle.md
[2]:https://github.com/summerDG/spark-code-ananlysis/blob/master/analysis/spark_sort_shuffle.md
[3]:https://github.com/ColZer/DigAndBuried/blob/master/spark/spark-memory-manager.md
[4]:https://github.com/apache/spark/pull/11805
[5]:https://0x0fff.com/spark-architecture/
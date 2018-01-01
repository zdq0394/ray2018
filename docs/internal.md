# An Overview of the Internals
## Connecting to Ray
Ray脚本的init有两种方式：
* a standalone fashion
* 连接到已经存在的Ray cluster

### Running Ray standalone
在脚本中执行`ray.init()`即可以standalone的方式运行Ray。
当调用`ray.init()`时，所有相关的进程都会启动：a local scheduler，a global scheduler，an object store和manager，a Redis server，和一定数量的worker processes。当脚本退出后，这些进程都将终止。

**该种方式仅适用于单一节点**

### Connecting to an existing Ray cluster
要连接到已经存在的Ray cluster，只需将Redis server的地址作为参数（redis_address=）传入到`ray.init`中。当调用`ray.init`时，没有新的进程会产生，当脚本结束时，Ray的进程继续存在。

**In this case, all processes except workers that correspond to actors are shared between different driver processes.**

## Defining a remote function
Ray系统的一个中心组件就是中心化的**control plane**。**Control plane**通过一个或者多个Redis servers实现。Redis is an in-memory key-value store。

**Centralized control plane**有两个作用：
* system's control state的持久存储。
* 进程之间通信的message bus（使用Redis的publish-subscribe功能）

考虑如下的remote function：
```python
@ray.remote
def f(x):
    return x + 1
```
当remote function定义后，函数立即被序列化pickled，并被分配一个unique ID，并存储到Redis server中。
每一个worker process都有一个单独的后台线程监听centralized control state中remote functions的增加。
当一个新的remote function添加后，该线程获取该remote function，并unpickle remote function，然后执行该function。
### Notes and limitations
* 因为remote functions一旦定义完毕，就立即pickle，所以remote function不能引用在function之后的方法和变量。比如：
```python
@ray.remote
def f(x):
    return helper(x)

def helper(x):
    return x + 1
```
helper方法在f之后，将抛出如下错误：
```python
Traceback (most recent call last):
    File "<ipython-input-3-12a5beeb2306>", line 3, in f
NameError: name 'helper' is not defined
```
## Calling a remote function
当一个remote function被调用后，`a number of things happen`：
1. a task object被创建：
  * remote function ID
  * 传入function的参数的IDs或者values，python primitives比如integers或者短的strings将被pickled，并作为task object的一部分。大的复杂的对象将通过internal `ray.put`方法存储到object store中，相应的IDs被包含到task object中。直接传入的参数的Object IDs也作为task object的一部分。
  * task ID。
  * task返回值的IDs。
2. task object被发送到local scheduler中：on the same node as the driver or worker。
3. local scheduler决定将task调度到本地还是global scheduler。
  * 如果task的所有依赖都在local object store中，并且本地有足够的CPU和GPU资源，local scheduler将把task分配给本地一个worker上。
  * 如果这些条件不满足，task将会传递给global scheduler。Task将被添加到task table中，task table是中央control state的一部分。 A global scheduler将获知这些更新，并更新task table中的task state，从而把任务分配给一个local scheduler。然后，local scheduler将获取通知，并pull the task object。
4. 当task被调度到一个local scheduler上，不论是被它自己，还是被global scheduler，local scheduler都将task入队列，等待执行。当资源足够时，task将被分配给一个worker去执行，按照先进先出的原则。
5. 当task被分配给一个worker后，worker执行这个任务，并将任务的返回值存入object store。Object store将更新`object table`,`object table`也是centralized control state的一部分。当task的返回值被存储到object store之后，将被使用`Apache Arrow`序列化到一个blob of bytes。

## Getting an object ID
当`ray.get`调用时，会发生以下事件：
```python
ray.get(x_id)
```
1. driver或者worker去位于同一节点上的object store请求相关的对象。每个object store都包含两个组件：一个包含不可变对象的共享内存的key-value存储和一个管理器——负责对象在节点之间的transfer。
    如果object store中不存在这个object，manager将检查object table，以确定其它的object store是否拥有这个object。如果存在，则通过manager直接请求这个object。如果不存在，centralized control state将通知requesting manager，当object被创建之后。如果这个object已经被evicted，worker将请求local scheduler重建该对象。
2. 当object在local object store存在后，driver/worker将映射相关的内存区域到它自己的地址空间（to avoid copying the object），并反序列化为Python对象。
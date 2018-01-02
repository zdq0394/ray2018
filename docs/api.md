# The Ray API
## Starting Ray
Ray的使用方式主要有两种：
* 可以在一个脚本中start所有相关的Ray进程，也可以在一个脚本中shutdown所有的进程。
* 可以连接到已经存在的Ray Cluster。

### Starting and stopping a cluster within a script
一个use case就是调用`ray.init()`启动Ray所有相关的进程，当脚本退出时，关闭所有的进程。
这些进程包括：local和global schedulers，object stores和object manager，redis server等。
```python
ray.init()
```
这种方式仅限于单机版。

如果节点上存在GPUs，可以通过参数`num_gpus`指定，同样，也可以通过参数`num_cpus`指定CPU数量。
```python
ray.init(num_cpus=20, num_gpus=2)
```

默认，Ray使用`psutil.cpu_count()`确定CPU的数量。 Ray也试图确定GPU的数量。

Instead of thinking about the number of “worker” processes on each node, we prefer to think in terms of the quantities of CPU and GPU resources on each node and to provide the illusion of an infinite pool of workers。

为了避免竞争，Tasks将会基于节点上**可用的资源**被分配到workers，而不是根据**可用的worker数量**。

### Connecting to an existing cluster
如果一个Ray Cluster已经启动，连接这个集群很简单，需要做的就是使用Ray cluster中Redis server的地址。 
在这种情况下，我们的脚本不会启动或者关闭任何Ray进程。
Ray cluster以及所有的Ray进程将会在多个脚本后者用户之间共享。
只需直到Ray Cluster中redis server的地址，可以通过如下语句：
```python
ray.init(redis_address="12.345.67.89:6379")
```

在这种情况下，不能在`ray.init`指定`num_cpus`或者`num_gpus`。

```python
ray.init(redis_address=None, node_ip_address=None, object_id_seed=None, num_workers=None, driver_mode=0, redirect_output=False, num_cpus=None, num_gpus=None, resources=None, num_custom_resource=None, num_redis_shards=None, redis_max_clients=None, plasma_directory=None, huge_pages=False, include_webui=True)
```

方法`ray.init`连接到Ray cluster或者创建一个cluster并连接。
## Defining remote functions
`Remote functions`用来创建`tasks`。
使用python装饰器模式`@ray.remote`，可以定义一个`remote function`。

使用`f.remote`可以调用`remote function`。
对`remote function`的调用会创建一个task——task将被调度到Ray cluster的某些worker process上执行，调用将立即返回一个object ID——代表task的最终返回值。
可以使用object ID获取返回值，而不管这个task时在哪里执行的。

当task执行时，它的输出将被序列化成a string of bytes，并存储到object store中。

`Remote functions`的参数可以是值，也可以是object ID。

```python
@ray.remote
def f(x):
    return x + 1

x_id = f.remote(0)
ray.get(x_id)  # 1

y_id = f.remote(x_id)
ray.get(y_id)  # 2
```

如果要让`remote function`返回多个object IDs，可以向装饰器传递参数`num_return_vals`。
```python
@ray.remote(num_return_vals=2)
def f():
    return 1, 2

x_id, y_id = f.remote()
ray.get(x_id)  # 1
ray.get(y_id)  # 2
```
装饰器remote用来定义个`remote function`。
```python
ray.remote(*args, **kwargs)
```
* num_return_vals (int) – The number of object IDs that a call to this function should return.
* num_cpus (int) – The number of CPUs needed to execute this function.
* num_gpus (int) – The number of GPUs needed to execute this function.
* resources – A dictionary mapping resource name to the required quantity of that resource.
* max_calls (int) – The maximum number of tasks of this kind that can be run on a worker before the worker needs to be restarted.
* checkpoint_interval (int) – The number of tasks to run between checkpoints of the actor state.

## Getting values from object IDs
可以以`Object IDs`作为参数，调用`ray.get`将object IDs转换为objects。`ray.get`可以既可以接受一个objectID作为参数，也可以接受 `a list of object IDs`作为参数。 
```python
@ray.remote
def f():
    return {'key1': ['value']}

# Get one object ID.
ray.get(f.remote())  # {'key1': ['value']}

# Get a list of object IDs.
ray.get([f.remote() for _ in range(2)])  # [{'key1': ['value']}, {'key1': ['value']}]
```

### Numpy arrays
Numpy arrays比其他数据类型效率更高。只要可能，首先使用numpy array。

`Any numpy arrays that are part of the serialized object will not be copied out of the object store. `
它们将一直存在于object store，反序列化后的对象里包含有指针，指向`object store's memeory`的相应地址。

`Object store`中的objects是不可变的，这意味着如果一个`remote function`的返回值是`numpy arrays`的话，如果要改变它的值，需要首先复制一份。

```python
ray.get(object_ids, worker=<ray.worker.Worker object>)
```
该方法从`object store`中获取`remote object`或者`a list of remote objects`。
该方法会block，直到相应的objects在local object store是可用的。

## Putting objects in the object store
把一个object放入object store的最主要的方式就是通过一个task的返回值。
也可以直接使用`ray.put`把一个object放入obejct store。
```python
x_id = ray.put(1)
ray.get(x_id)  # 1
```
使用`ray.put`的最大场景就是需要把`a large object`传递给多个tasks。首先把object存入object store，然后把返回的object ID传递给task，这样只需要复制一次大对象即可。否则每个任务都要复制一次。
```python
import numpy as np

@ray.remote
def f(x):
    pass

x = np.zeros(10 ** 6)

# Alternative 1: Here, x is copied into the object store 10 times.
[f.remote(x) for _ in range(10)]

# Alternative 2: Here, x is copied into the object store once.
x_id = ray.put(x)
[f.remote(x_id) for _ in range(10)]
```

## Waiting for a subset of tasks to finish
有时候经常需要基于不同的task完成的时间对正在执行的task进行调整。`It is often desirable to adapt the computation being done based on when different tasks finish. `

Ray引入了语句`ray.wait`。

```python
ray.wait(object_ids, num_returns=1, timeout=None, worker=<ray.worker.Worker object>)
```
`Return a list of IDs that are ready and a list of IDs that are not`。

如果设置了`timeout`，以下条件任何一个满足，函数会返回：
* the requested number of IDs are ready
* the timeout is reached
whichever occurs first。
如果没设置`timeout`，函数直到请求的num_returns个对象可用之后才返回。

```python
@ray.remote
def f():
    return 1

# Start 5 tasks.
remaining_ids = [f.remote() for i in range(5)]
# Whenever one task finishes, start a new one.
for _ in range(100):
    ready_ids, remaining_ids = ray.wait(remaining_ids)
    # Get the available object and do something with it.
    print(ray.get(ready_ids))
    # Start a new task.
    remaining_ids.append(f.remote())
```
## Viewing errors
`ray.error_info(worker=<ray.worker.Worker object>)`返回failed task的信息。


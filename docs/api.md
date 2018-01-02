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



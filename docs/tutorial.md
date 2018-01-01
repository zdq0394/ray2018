# 简单教程
使用Ray，需要首先理解：
* Ray如何异步执行task以达到并行。
* Ray如何使用object IDs代表不可变的远程对象（remote objects）。

## Overview
Ray是一个基于Python的分布式执行引擎。
同样的代码可以在一台机器上执行达到高效的多进程效果。
并且可以在大规模计算集群上使用。

当使用Ray时，牵涉到如下几个进程：
* 多个worker进程执行tasks并将结果存储到到object stores。每一个worker是一个独立的进程。
* 每个节点一个object store，用来在共享内存中存储immutable objects，并且让workers高效的共享同一节点上的对象——最小化拷贝过程和反序列化过程。
* 每个节点一个local scheduler，用来分配task到同一节点上的workers。
* 一个global scheduler，从local scheduler接收任务，并将任务分配到其它的local schedulers。
* 一个driver是一个python process，由用户控制。比如，如果用户正在run一个script或者python shell，那么driver就是一个运行script和shell的python process。一个driver和一个worker类似，可以向local scheduler提交task，从object store获取obejcts，不同点是local scheduler不会分配task到driver上。
* Redis server，维持系统的state。比如，它保持着哪个objects在哪个machines上，和task的specifications。

## Starting Ray
可以通过如下python脚本启动Ray。
```python
import ray
ray.init()
```
这样就启动了Ray。

## Immutable remote objects
In Ray，可以创建objects并在objects上面进行计算。这些objects称为`remote objects`，并且可以通过`object ID`引用。
`Remote objects`存储在`object store`中。在集群中，每个节点有一个`object store`。
在集群中的设置中，我们并不知道每个对象存在哪个节点上。

一个`object ID`是引用`remote object`的一个唯一值。

`Remote objects`被认为是**不可变的（immutable）**。也就是说，它们的值一旦创建后就不再改变。
这样允许`remote objects`在多个object store中复制，而不用同步它们。

### Put and Get
命令`ray.get`和`ray.put`可以用来在Python objects和object IDs之间相互转换。
```python
x = "example"
ray.put(x)  # ObjectID(b49a32d72057bdcfc4dda35584b3d838aad89f5d)
```

命令`ray.put(x)`被一个worker或者driver执行。它把一个Python object拷贝到local object store（ 这里local意思是指同一个节点）。Object一旦被存入object store，它的值不能再被改变。

另外，`ray.put(x)`返回一个object ID，可以用来引用新创建的remote object。如果我们把object ID保存到变量`x_id`：`x_id = ray.put(x)`，可以把变量`x_id`传送到remote functions，remote functions将在remote objects上执行。

命令`ray.get(x_id)`将object ID作为参数，根据对应的remote object创建Python object。
对于一些objects，比如arrays，我们可以利用共享内存，避免复制object。
对于其它的objects，从object store拷贝对象到worker process's 堆空间中。
如果和object ID对应的remote object和调用者不在同一个节点，remote object会首先transferred到本地的object store中。
```python
x_id = ray.put("example")
ray.get(x_id)  # "example"
```

如果和object ID对应的remote object还未创建，命令`ray.get(x_id)`将等待直到remote object被创建。

命令`ray.get`的一个非常常见的case是获a list of object IDs。
在这种case下，可以调用`ray.get(object_ids)`，object_ids is a list of object IDs。

```python
result_ids = [ray.put(i) for i in range(10)]
ray.get(result_ids)  # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```
## Asynchronous Computation in Ray
Ray可以将任何Python functions异步的执行。
Ray将Python function指明为remote function。

比如，正常一个Python function如下：
```python
def add1(a, b):
    return a + b
```
一个remote function则如下形式：
```python
@ray.remote
def add2(a, b):
    return a + b
```

### Remote functions
调用`add1(1, 2)`返回`3`，并且使Python interpreter阻塞直到计算完成。

调用`add2.remote(1, 2)`将立即返回一个object ID并且创建一个task；Task将被调度到系统中，并且异步执行（有可能在一个不同的机器上执行）。当任务执行结束，task的返回值将被存储到object store中。
```python
x_id = add2.remote(1, 2)
ray.get(x_id)  # 3
```

如下例子展示了异步任务如何使用并行化计算。
```python
import time

def f1():
    time.sleep(1)

@ray.remote
def f2():
    time.sleep(1)

# The following takes ten seconds.
[f1() for _ in range(10)]

# The following takes one second (assuming the system has at least ten CPUs).
ray.get([f2.remote() for _ in range(10)])
```
`Submitting a task`和`executing the task`之间有显著的不同。当一个remote function被调用，执行该function的task被提交到local scheduler中，并立即返回task的object IDs。但是任务并不会立即执行，直到任务被真正调度到一个worker上。
Task execution并不是done lazily。
系统将data移到task中，当input dependencies和依赖的资源满足后，task将立即执行。

当一个任务提交后，每个参数都将通过`值传递`或者object ID传送。
比如，如下3个例子具有相同的行为：
```python
add2.remote(1, 2)
add2.remote(1, ray.put(2))
add2.remote(ray.put(1), ray.put(2))
```
Remote functions不返回实际的值，返回object IDs。

当`remote function`实际执行后，它在Python objects上执行。也就是说，`remote function`以`object IDs`为参数被调用后，系统将从object store中获取相应的objects。

`remote function`可以返回多个object IDs。
```python
@ray.remote(num_return_vals=3)
def return_multiple():
    return 1, 2, 3

a_id, b_id, c_id = return_multiple.remote()
```
### Expressing dependencies between tasks
Task之间的依赖关系，可以通过object ID来表达。
比如Task A的object ID作为另一个Task B的参数。
比如：
```python
@ray.remote
def f(x):
    return x + 1

x = f.remote(0)
y = f.remote(x)
z = f.remote(y)
ray.get(z) # 3
```

```python
import numpy as np

@ray.remote
def generate_data():
    return np.random.normal(size=1000)

@ray.remote
def aggregate_data(x, y):
    return x + y

# Generate some random data. This launches 100 tasks that will be scheduled on
# various nodes. The resulting data will be distributed around the cluster.
data = [generate_data.remote() for _ in range(100)]

# Perform a tree reduce.
while len(data) > 1:
    data.append(aggregate_data.remote(data.pop(0), data.pop(0)))

# Fetch the result.
ray.get(data)
```
### Remote Functions Within Remote Functions
目前为止，我们都是在driver中调用`remote functions`。但是worker processes也可以调用`remote functions`，如下例子：
```python
@ray.remote
def sub_experiment(i, j):
    # Run the jth sub-experiment for the ith experiment.
    return i + j

@ray.remote
def run_experiment(i):
    sub_results = []
    # Launch tasks to perform 10 sub-experiments in parallel.
    for j in range(10):
        sub_results.append(sub_experiment.remote(i, j))
    # Return the sum of the results of the sub-experiments.
    return sum(ray.get(sub_results))

results = [run_experiment.remote(i) for i in range(5)]
ray.get(results) # [45, 55, 65, 75, 85]
```

当`remote function` `run_experiment`在一个worker上执行，它调用`remote function` `sub_experiment`多次。
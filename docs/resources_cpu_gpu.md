# Resource (CPUs, GPUs)
本文描述Ray如何管理Resources（CPUs和GPUs）。
Ray集群中的每个node都知道自己的resource capacities；每个任务都指定了resource requirements。
## CPUs and GPUs
Ray内置支持CPUs和GPUs。
### Specifying a node’s resource requirements
可以通过命令行指定node的resource requirements。
* --num-cpus
* --num-cpus
```sh
# To start a head node.
ray start --head --num-cpus=8 --num-gpus=1

# To start a non-head node.
ray start --redis-address=<redis-address> --num-cpus=4 --num-gpus=2
```

也可以通过ray.init指定node的resource requirements。
```python
ray.init(num_cpus=8, num_gpus=1)
```
如果不指定num_cpus，Ray通过`psutil.cpu_count()`获取CPU数量。
如果没有设置num_gpus，Ray也将试图自动获取。

### Specifying a task’s CPU and GPU requirements
可以指定一个task的CPU和GPU requirements，向ray.remote装饰器传送参数num_cpus和num_gpus。
```python
@ray.remote(num_cpus=4, num_gpus=2)
def f():
    return 1
```
任务f将被调度到至少拥有4个CPU和2个GPU的节点。当一个f任务执行时，4个CPU和2个GPU将保留给该任务。
保留给任务f的GPU ids可以通过`ray.get_gpu_ids()`获得。Ray将为该进程自动设置环境变量`CUDA_VISIBLE_DEVICES`。
当任务结束后，资源将被释放。

然而，如果调用`ray.get`时，任务被阻塞。
比如，考虑下面的remote function：
```python
@ray.remote(num_cpus=1, num_gpus=1)
def g():
    return ray.get(f.remote())
```
当一个g任务正在执行时，它将会释放它的CPU资源给f任务，如果`ray.get`被阻塞。当`ray.get`返回后，它将重新获取CPU资源。 在任务的整个生命周期内，它都将retain它的GPU资源，because the task will most likely continue to use GPU memory.

如下可以指定一个`actor`对GPU的需求：
```python
@ray.remote(num_gpus=1)
class Actor(object):
    pass
```
当`Actor`实例创建后，它将被调度到至少含有1个GPU的节点上，并且该GPU将在actor的整个生命周期内保留给该actor（即使该actor没有执行任务）。当actor终止时，GPU资源才被释放。**Note that currently only GPU resources are used for actor placement.**

## Custom Resources
尽管Ray内置了对CPUs和GPUs，可以在启动时指定任意的自定义资源。所有的自定义资源的行为都和GPUs一样。

可以如下启动节点，指定自定义资源：
```sh
ray start --head --resources='{"Resource1": 4, "Resource2": 16}'
```

也可以通过如下方式：
```python
ray.init(resources={'Resource1': 4, 'Resource2': 16})
```

指定任务对自定义资源的需求，可以按照如下方式：
```python
@ray.remote(resources={'Resource2': 1})
def f():
    return 1
```

## Current Limitations
目前（Ray 0.3）还有如下问题：
* Actor Resource Requirements:目前仅支持GPUs。
* Recovering from Bad Scheduling: Currently Ray does not recover from poor scheduling decisions. For example, suppose there are two GPUs (on separate machines) in the cluster and we wish to run two GPU tasks. There are scenarios in which both tasks can be accidentally scheduled on the same machine, which will result in poor load balancing.
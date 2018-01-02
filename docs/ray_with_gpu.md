# Using Ray with GPUs
GPU对机器学习的应用非常关键。
Ray使得`remote function`和`actor`可以说明它们的CPU需求——通过`ray.remote`装饰器。
## Starting Ray with GPUs
可以通过如下方式启动Ray，并指明有多少GPU可用。
```python
ray.init(num_gpus=4)
```
或者
```sh
ray start --head --num-gpus=4
```

## Using Remote Functions with GPUs
如果`remote functions`需要使用GPU，可以通过remote decorator指定：
```python
@ray.remote(num_gpus=1)
def gpu_method():
    return "This function is allowed to use GPUs {}.".format(ray.get_gpu_ids())
```
在remote function内部，调用`ray.get_gpu_ids()`将返回`a list of integers` 指出哪些GPUs可以被该`remote function`使用。

尽管该方法实际上没有使用任何GPU，Ray还是会把它调度到至少含有一个GPU的节点上。
## Using Actors with GPUs
如果`actor`需要使用GPUs，也可以通过`ray.remote`装饰器指定。
```python
@ray.remote(num_gpus=1)
class GPUActor(object):
    def __init__(self):
        return "This actor is allowed to use GPUs {}.".format(ray.get_gpu_ids())
```
当这个actor被实例化后，在actor的整个生命周期，GPUs都将被保留给它。
```python
@ray.remote(num_gpus=1)
class GPUActor(object):
    def __init__(self):
        self.gpu_ids = ray.get_gpu_ids()
        os.environ["CUDA_VISIBLE_DEVICES"] = ",".join(map(str, self.gpu_ids))
        # The call to tf.Session() will restrict TensorFlow to use the GPUs
        # specified in the CUDA_VISIBLE_DEVICES environment variable.
        self.sess = tf.Session()
```
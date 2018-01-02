# Actors
Ray中的`remote functions`是`functional`并且是`side-effect free`。如果限制仅使用`remote functions`，则类似与提供了一个分布式函数式编程，在很多场景下，这是非常好的。但是实际上会有些受限。

Ray引入了`actors`。
一个actor实质上是一个stateful worker(or a service)。一个actor实例化后，一个新的worker就被创建了，actor的方法将被调度到该特定的worker上，并且可以修改worker的状态。

## Defining and creating an actor
如下例，`ray.remote`装饰器指出，Counter的实例是一个`actor`。
```python
@ray.remote
class Counter(object):
    def __init__(self):
        self.value = 0

    def increment(self):
        self.value += 1
        return self.value
```
真正创建一个actor，可以通过`Counter.remote()`。
```python
a1 = Counter.remote()
a2 = Counter.remote()
```
当一个actor实例化后，随机发生了如下事件：
1. cluster的一个node被选中，该node上的local scheduler将创建一个worker process，用来运行actor的方法。
2. 一个Counter的对象在worker中建立，Counter的构造函数会执行。 

## Using an actor
通过调用actor的方法，可以把`task`调度到actor上（actor所在的worker）。
```python
a1.increment.remote()  # ray.get returns 1
a2.increment.remote()  # ray.get returns 1
```
当`a1.increment.remote()`调用后，执行了一下操作：
1. 一个task被创建。
2. 这个task被driver's local scheduler直接分配到负责actor的local scheduler上去。也就是说，本次调度是`bypass` global scheduler的。
3. 一个object ID返回。

然后，可以通过在object ID上调用`ray.get`获取object的值。

只有actor的methods创建的task会调度到actor上，一般的`remote function`不会。

同一个actor的方法生成的多个task在同一个actor上顺序执行，它们之间可以共享状态。
```python
# Create ten Counter actors.
counters = [Counter.remote() for _ in range(10)]

# Increment each Counter once and get the results. These tasks all happen in
# parallel.
results = ray.get([c.increment.remote() for c in counters])
print(results)  # prints [1, 1, 1, 1, 1, 1, 1, 1, 1, 1]

# Increment the first Counter five times. These tasks are executed serially
# and share state.
results = ray.get([counters[0].increment.remote() for _ in range(5)])
print(results)  # prints [2, 3, 4, 5, 6]
```

## A More Interesting Actor Example
一个通常的模式是使用actors包装其它library/service管理的可变的状态。
Gym为测试和训练`reinforcement learning agents`的多个仿真环境提供了一个接口。
这些仿真器是有状态的`stateful`，使用这些仿真器的task必须改变这些状态。
可以使用actors来包装这些仿真器的状态。

```python
import gym

@ray.remote
class GymEnvironment(object):
    def __init__(self, name):
        self.env = gym.make(name)
        self.env.reset()

    def step(self, action):
        return self.env.step(action)

    def reset(self):
        self.env.reset()
```
然后可以如下方式初始化和调度actor：
```python
pong = GymEnvironment.remote("Pong-v0")
pong.step.remote(0)  # Take action 0 in the simulator.
```
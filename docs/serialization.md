# Serialization in the Object Store
## Serialization
本文描述了哪些python对象可以被Ray序列化入object store，哪些不能。
对象一旦被放入object store，就是**不可变**的。

如下情况下，Ray将把对象置入object store：
1. **remote**函数的返回值。
2. ray.put(x)函数的参数x。
3. **remote**函数的参数，除了简单类型——ints或者floats。

一个`Python`对象可能包含任意数量的`指针`，任意深度的嵌套。
要把Python对象放入object store，或者在进程之间传送，首先必须转换为**字节序列**（stirng of bytes）。
这个过程称为**序列化**（serialization）。
与此相对的过程称为**反序列化**（deserialization）。
**序列化和反序列化通常是分布式计算中的瓶颈**。

`Pickle`是Python中进行序列化和反序列化的一个库。
Pickle及其变种cloudpickle是通用的序列化器。它可以序列化一大类的Python objects。然而，对于数字型的对象效率却不高。
如果多个进程都要访问a Python list of numpy arrays，每个进程都必须unpickle the list并创建它自己的arrays副本。这会造成巨大的内存压力，即使所有的进程都是read-only，本可以共享内存。

Ray对此进行了优化，对于numpy arrays，Ray使用`Apache Arrow`格式。当反序列化numpy arrays的列表时，仍然会创建一个Python list of numpy array objects。但是不会复制每个numpy array，每个numpy array持有一个指针，指向共享内存中的array。
这带来如下优势：
* 反序列化非常快。
* 内存是共享的，不必拷贝，多个进程即可读取数据。

## What Objects Does Ray Handle
Ray目前并不支持序列化任意的Python对象。
Ray使用`Arrow`可以序列化如下对象：
* Primitive types: ints, floats, longs, bools, strings, unicode, and numpy arrays.
* Any list, dictionary, or tuple whose elements can be serialized by Ray.

对于更通用的对象，Ray首先将对象unpacking as a dictionary of its fields.
该行为并非总是正确的。
如果Ray不能将对象序列化为a dictionary of its fields，Ray将使用`pickle`。

## Notes and limitations
*  a list that contains two copies of the same list will be serialized as if the two lists were distinct.
```python
l1 = [0]
l2 = [l1, l1]
l3 = ray.get(ray.put(l2))

l2[0] is l2[1]  # True.
l3[0] is l3[1]  # False.
```

*  不支持objects that recursively contain themselves (this may be common in graph-like data structures).
```python
l = []
l.append(l)

# Try to put this list that recursively contains itself in the object store.
ray.put(l)
```
* 为了性能最大化，尽可能使用`numpy arrays`

## Last Resort Workaround
可以通过使用pickle定制序列化和反序列化代码。
```python
import pickle

@ray.remote
def f(complicated_object):
    # Deserialize the object manually.
    obj = pickle.loads(complicated_object)
    return "Successfully passed {} into f.".format(obj)

# Define a complicated object.
l = []
l.append(l)

# Manually serialize the object and pass it in as a string.
ray.get(f.remote(pickle.dumps(l)))  # prints 'Successfully passed [[...]] into f.'
```
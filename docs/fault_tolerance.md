# Fault Tolerance
## Machine and Process Failuers
目前，每个local scheduler和plasma manager都向monitor进程发送heartbeats。
如果monitor在一定时间内（大约10秒钟）没有收到心跳信息，就将该进程标记为`dead`，并将相关的状态信息从Redis server中清除。
如果一个manager被标记为`dead`，object table将会更新，以删除manager的所有occurrences，这样一来，其它的manager不会从已经`dead`的manager获取对象。
如果一个local scheduler被标记为`dead`，则task table中，运行在该local scheduler的所有task将被标记为`lost`，所有的actor将会在其它的local schedulers上面重建。

## Lost Objects
如果一个对象丢失了后者从未创建，那么创建该对象的task将重新执行以创建该对象。
如果必要的话，task的参数——如果是由其它task创建的——相应的task也要重新执行。

## Actors
当一个local scheduler被标记为`dead`之后，所有相关的actors，如果是活着的，将在其它的local scheduler重建。
默认，所有的actor methods将会按照顺序重新执行
如果actor checkpointing is enabled，actor的状态将从最近的check point加载，check point点之后的methods将会重新执行。

## Unhandled Failures
### Process Failuers
* Ray does not recover from the failure of any of the following processes: a Redis server, the global scheduler, the monitor process.
* If a driver fails, that driver will not be restarted and the job will not complete.
### Lost Objects
* If an object is constructed by a call to `ray.put` on the driver, is then evicted, and is later needed, Ray will not reconstruct this object.
* If an object is constructed by an actor method, is then evicted, and is later needed, Ray will not reconstruct this object.

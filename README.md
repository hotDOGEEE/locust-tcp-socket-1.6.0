# locust-tcp-socket-1.6.0
locust tcp socket长连接压测总结

----------

## Goals

  - 总结内容包括
    - [x] locust自定义socket client
    - [x] socket.recv中应该注意的部分
    - [x] 大量用户情况下，如何方便的做到最优的用户生成
    - [x] 一个流程中涉及到多个socket的情况，以及调用问题
    
## locust自定义socket client

相信大多数资料，都是写的locust对于http这种一问一答形式的流程做压测的。

但是http只是众多协议请求中的一种，除开http以外，很多特殊的协议和流程都需要我们自己去构建。

而构建的过程其实也并不复杂，我们只需要理解原来使用客户端的类，对他的基类进行重写，加入locust中指定的events事件就可以了。

最后用重构出来的类实例化，后续调用即可。

## socket.recv中应注意的部分

一般我们直接使用socket中的recv，当然是可以接受到数据的。

但使用过就会发现，如果你只是做一次recv，往往并不能解决业务场景真正需要的效果。

tcp的传输是流形式，包不是接一次就完，也不一定你发过去后第一时间就会反给你结果。

提供一种最简单的解决思路，能够满足大多数场景：阻塞死等一秒，然后一次性进行接收，对收到的整体内容进行解包之类的校验，判断通过没有。
如果一秒收不到，当超时处理。

这种情况去验流程当然是可以的，不过他的问题也很明显：
1. 为阻塞场景，不管是做c还是s，死阻塞都是不可取的。
2. 会导致接受服务器反包不及时，无法精确判断响应时间。
3. 哪怕一次的请求中包含了所有数据，也无法从逻辑上保证数据为当前请求的数据，是不是不多不少。
4. 遇到失败的场景，返回超时，会导致一直卡在recv无法进行后续步骤，如果设置timeout，socket也会断开后续无法继续使用。

这里分享一个非阻塞的recv，可以直接套用
```
@count_time
def recv_cat(s):
    buffer = [s.recv(1024)] #一开始的部分,用于等待传输开始,避免接收不到的情况.
    if buffer[0] in (0,-1): #返回0,-1代表出错
        return False
    s.setblocking(0) #非阻塞模式
    while True: #循环接收
        try:
            data=s.recv(1024) #接收1024字节
            buffer.append(data) #拼接到结果中
        except BlockingIOError as e: #如果没有数据了
            break #退出循环
    s.setblocking(1) #恢复阻塞模式
    return b"".join(buffer)
```

装饰器函数
```
def count_time(func):
    # 用于统计时间的装饰器
    def int_time(s):
        start_time = time.time()  # 程序开始时间
        buffer = func(s)
        over_time = time.time()  # 程序结束时间
        total_time = int((over_time - start_time) * 1000)
        events.request_success.fire(request_type="socket_delay", name="last_pag", response_time=total_time,
                                    response_length=0)
        return buffer
    return int_time
```

## 大量用户情况下，如何方便的做到最优的用户生成

做并发嘛，那肯定是需要大量数据的。如果不采用分布式，只跑单核的话，那随便怎样生成都可以。

但如果是多核分布式，我们要保证并发的时候不能有文件读写的IO冲突，不能有多个核同时在使用一套数据的冲突。

避免这种情况的发生，一般都会用中间件的方式来进行。 这里放一个介绍的连接 https://zhuanlan.zhihu.com/p/41535243

不过，其实中间件也不是解决这个问题的唯一办法。

很多时候，做压测我们并不必要去用数据库中本身真实的内容。

直接在脚本中生成一个不会重复的ID，是我采用的一种非常方便的解决办法。

分布式是在不同核下进行的，进程ID也不同，我们可以用任何你想用的方式生成一个只属于当前slave的ID。再用这个ID+编号的形式完成数据的生成。

你可以生成一个列表，一个队列，一个生成器。本人前期也是这样处理的。

然而在你的用户有几万，几十万甚至上百万的时候，这些用户数据就会占据一定的内存了，而这些用户数据在跑完了以后并不会再被用到。

使用迭代器，保证内存不会越用越多，是最佳的解决方案。

```
class Iter_Uname:
    """
    名称迭代器
    """
    def __init__(self):
        self.rand_name = str()
        abc_lens = random.randint(1, 10)
        for i in range(abc_lens):
            self.rand_name += random.choice(string.ascii_letters)

    def __iter__(self):
        self.count = 0
        return self

    def __next__(self):
        full_name = self.rand_name + str(self.count)
        self.count += 1
        return full_name
```
将他直接作为一个单独的class放在文件中，保证他只会被实例化一次。在每次需要用户名的时候next(iter(Iter_Uname()))，就能以内存最优的方式管理你的用户数据了。

## 一个流程中涉及到多个socket的情况，以及调用问题

这个问题更像是针对项目的总结了，并没很有泛用性，就当是我仅做一个个人的总结。

其实这种情况没什么好纠结的，需要关心的socket，我们用重定义过的就好，不需要关心的，我们用本身自带的。

调试过程中，我们要支持协议具体的class，传入自定义的socket，没有传的情况下再自己去生成。

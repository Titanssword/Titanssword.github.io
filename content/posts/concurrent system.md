---
title: Rate Limit 
date: 2018-06-21
tags:
- concurrent system
- rate limit

categories:
- system
description: "网站架构核心技术之限流"
---
# Rate limit 限流技术

在高并发系统中经常使用三种技术保护系统：缓存，降级和限流。缓存的目的是提升系统访问速度和增大系统能处理的容量，降级是当服务出问题是暂时屏蔽掉，等高峰期后或问题解决后在重启，然而针对有些场景并不能用缓存和降级来解决，比如稀缺资源（秒杀，抢购），写服务（评论，下单），频繁的复杂查询（评论的最后几页），这时候 需要限流这项技术来限制这些场景的并发/请求量。限流的目的是通过对并发访问/请求进行限速或者一个时间窗口内的的请求进行限速来保护系统，一旦达到限制速率则可以拒绝服务（定向到错误页或告知资源没有了）、排队或等待（比如秒杀、评论、下单）、降级（返回兜底数据或默认数据，如商品详情页库存默认有货）

## 常见限流算法 （算法层面限流）

计数器， 令牌桶， 漏桶

### 计数器
主要用来限制总并发数，比如数据库连接池，线程池，秒杀的并发数；超过了瞬时请求数或者在一定时间段内的请求数设定的阈值，则进行简单粗暴的限流，主要针对的还是总体的请求数量，而不是考虑的平均速率的限流。

### 令牌桶 Token Bucket
随着时间流逝,系统会按恒定1/QPS时间间隔(如果QPS=100,则间隔是10ms)往桶里加入Token(想象和漏洞漏水相反,有个水龙头在不断的加水),如果桶已经满了就不再加了.新请求来临时,会各自拿走一个Token,如果没有Token可拿了就阻塞或者拒绝服务.令牌桶的另外一个好处是可以方便的改变速度. 一旦需要提高速率,则按需提高放入桶中的令牌的速率. 一般会定时(比如100毫秒)往桶中增加一定数量的令牌, 有些变种算法则实时的计算应该增加的令牌的数量
![token_bucket](https://github.com/Titanssword/Notes/blob/master/pic/rate%20limit/token_bucket.JPG?raw=true)


长期来看，符合流量的速率是受到令牌添加速率的影响，被稳定为：r
因为令牌桶有一定的存储量，可以抵挡一定的流量突发情况
M是以字节/秒为单位的最大可能传输速率。 M>r
T max = b/(M-r) 承受最大传输速率的时间
B max = T max * M 承受最大传输速率的时间内传输的流量

Guava RateLimiter提供了令牌桶算法实现：平滑突发限流(SmoothBursty)和平滑预热限流(SmoothWarmingUp)实现。

### 漏桶 Leaky Bucket
漏桶(Leaky Bucket)算法思路很简单,水(请求)先进入到漏桶里,漏桶以一定的速度出水(接口有响应速率),当水流入速度过大会直接溢出(访问频率超过接口响应速率),然后就拒绝请求,可以看出漏桶算法能强行限制数据的传输速率.
![Leaky Bucket](https://github.com/Titanssword/Notes/blob/master/pic/rate%20limit/leaky%20bucket.png?raw=true)

### 两桶对比
- 令牌桶是按照固定速率往桶中添加令牌，请求是否被处理需要看桶中令牌是否足够，当令牌数减为零时则拒绝新的请求；
- 漏桶则是按照常量固定速率流出请求，流入请求速率任意，当流入的请求数累积到漏桶容量时，则新流入的请求被拒绝；
- 令牌桶限制的是平均流入速率（允许突发请求，只要有令牌就可以处理，支持一次拿3个令牌，4个令牌），并允许一定程度突发流量；
- 漏桶限制的是常量流出速率（即流出速率是一个固定常量值，比如都是1的速率流出，而不能一次是1，下次又是2），从而平滑突发流入速率；
- 令牌桶允许一定程度的突发，而漏桶主要目的是平滑流入速率；
- 两个算法实现可以一样，但是方向是相反的，对于相同的参数得到的限流效果是一样的。

## 应用级限流
一般来说，对于一个应用系统都会有极限并发数和请求数，TPS和QPS。如果超过了阈值则系统会无法响应或者崩溃，所以一般在进行开发大规模系统的时候，要设置过载保护，以免大量请求使系统瘫痪。
1. 限流总资源数。
有的资源是稀缺资源（如数据库连接、线程）， 可以使用池化技术来限制总资源数：连接池、线程池。比如分配给每个应用的数据库连接是100，那么本应用最多可以使用100个资源，超出了可以等待或者抛异常。
如果接口可能会有突发访问情况，但又担心访问量太大造成崩溃，如抢购业务；这个时候就需要限制这个接口的总并发/请求数总请求数了；因为粒度比较细，可以为每个接口都设置相应的阀值。可以使用Java中的AtomicLong进行限流：
```
try {
      if(atomic.incrementAndGet() > 限流数) {
          //拒绝请求  
      }    
      //处理请求
      } finally {    
        atomic.decrementAndGet();
      }

```
适合对业务无损的服务或者需要过载保护的服务进行限流，如抢购业务，超出了大小要么让用户排队，要么告诉用户没货了，对用户来说是可以接受的。而一些开放平台也会限制用户调用某个接口的试用请求量，也可以用这种计数器方式实现。这种方式也是简单粗暴的限流，没有平滑处理，需要根据实际情况选择使用；
2. 限流某个接口的时间窗请求数
即一个时间窗口内的请求数，如想限制某个接口/服务每秒/每分钟/每天的请求数/调用量。如一些基础服务会被很多其他系统调用，比如商品详情页服务会调用基础商品服务调用，但是怕因为更新量比较大将基础服务打挂，这时我们要对每秒/每分钟的调用量进行限速；
时间窗最大请求数，指定的时间范围内允许的最大请求数
优点：这个算法能够满足绝大多数的流控需求，通过时间窗最大请求数可以直接换算出最大的QPS（QPS = 请求数/时间窗）
缺点：这种方式可能会出现流量不平滑的情况，时间窗内一小段流量占比特别大
```
LoadingCache<Long, AtomicLong> counter =
        CacheBuilder.newBuilder()
                .expireAfterWrite(2, TimeUnit.SECONDS)
                .build(new CacheLoader<Long, AtomicLong>() {
                    @Override
                    public AtomicLong load(Long seconds) throws Exception {
                        return new AtomicLong(0);
                    }
                });
long limit = 1000;
while(true) {
    //得到当前秒
    long currentSeconds = System.currentTimeMillis() / 1000;
    if(counter.get(currentSeconds).incrementAndGet() > limit) {
        System.out.println("限流了:" + currentSeconds);
        continue;
    }
    //业务处理
}

```
我们使用Guava的Cache来存储计数器，过期时间设置为2秒（保证1秒内的计数器是有的），然后我们获取当前时间戳然后取秒数来作为KEY进行计数统计和限流，这种方式也是简单粗暴，刚才说的场景够用了。
3. 平滑限流某个接口的请求数
之前的限流方式都不能很好地应对突发请求，即瞬间请求可能都被允许从而导致一些问题；因此在一些场景中需要对突发请求进行整形，整形为平均速率请求处理（比如5r/s，则每隔200毫秒处理一个请求，平滑了速率）。这个时候有两种算法满足我们的场景：令牌桶和漏桶算法。Guava框架提供了令牌桶算法实现，可直接拿来使用。
Guava RateLimiter提供了令牌桶算法实现：平滑突发限流(SmoothBursty)和平滑预热限流(SmoothWarmingUp)实现。



## Ref
[http://xiaobaoqiu.github.io/blog/2015/07/02/ratelimiter/](http://xiaobaoqiu.github.io/blog/2015/07/02/ratelimiter/)
[https://www.jianshu.com/p/a3d068f2586d](https://www.jianshu.com/p/a3d068f2586d)
[https://mp.weixin.qq.com/s?__biz=MzI0MTk0NTY5MA==&mid=2247483711&idx=1&sn=28780c8b26f24ac6314ff5c599bb622c&chksm=e9029c0ade75151c353cd6b720ce438b4342afd8ef3a7d03c61712554c6a000ac3646bbc3124&scene=38#wechat_redirect](https://mp.weixin.qq.com/s?__biz=MzI0MTk0NTY5MA==&mid=2247483711&idx=1&sn=28780c8b26f24ac6314ff5c599bb622c&chksm=e9029c0ade75151c353cd6b720ce438b4342afd8ef3a7d03c61712554c6a000ac3646bbc3124&scene=38#wechat_redirect)
[聊聊高并发系统之限流特技-1](http://zhuanlan.51cto.com/art/201611/523072.htm)

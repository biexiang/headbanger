---
title: "Redis分布式限流器"
date: 2023-02-22T22:48:51+08:00
categories: ['Golang','Troubleshooting']
keywords: ['Golang','Go Runtime Throws Gorountine Stack Dumps']
description: "Go Runtime Throws Gorountine Stack Dumps"
---

{{< music id="1814784389" >}}

压测场景下，需要准确控制endpoint的发压QPS，因为压测机器很多，所以需要一个分布式限流器来对压力进行控制。

## 策略上

### 基于令牌桶
参考go-zero中的[tokenlimit](https://github.com/zeromicro/go-zero/blob/master/core/limit/tokenlimit.go)实现，原理即通过eval执行下面的lua脚本，一句话概括就是对于一个key，每次请求获取最近一次变更的时间戳以及剩余的token数，计算时间差按照平均速率能增加多少token，总和token数能不能满足本次的token数需求：

{{< highlight go "linenos=table,linenostart=1" >}}
local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])   -- current timestamp may need redis.call("TIME")
local requested = tonumber(ARGV[4])

local fill_time = capacity/rate
local ttl = math.floor(fill_time*2)

local last_tokens = tonumber(redis.call("get", KEYS[1]))
if last_tokens == nil then
    last_tokens = capacity
end
local last_refreshed = tonumber(redis.call("get", KEYS[2]))
if last_refreshed == nil then
    last_refreshed = 0
end
local delta = math.max(0, now-last_refreshed)
local filled_tokens = math.min(capacity, last_tokens+(delta*rate))
local allowed = filled_tokens >= requested
local new_tokens = filled_tokens
if allowed then
    new_tokens = filled_tokens - requested
end
redis.call("setex", KEYS[1], ttl, new_tokens)
redis.call("setex", KEYS[2], ttl, now)
return allowed
{{< / highlight >}}

在这里`rate`是速率的含义，`burst`是容量的含义，并且如果redis连接异常，会使用`golang.org/x/time/rate`的本地限速器进行兜底，只不过如果限速QPS是集群总QPS，以同样的rate降级为本地限速器最终的限速会升级为rate*集群机器数，不太合理。

### 基于滑动窗口
能实现一样的效果，就是没有tokenlimiter好理解。
{{< highlight go "linenos=table,linenostart=1" >}}
local a = KEYS[1]
local b = tonumber(ARGV[1])                 -- refill duration
local c = tonumber(ARGV[2])                 -- current cost duration
local d = redis.call("TIME")
local e = d[1] + d[2] / 1e6                 -- current timestamp
local f = math.ceil(b) + 1                  
local g = e + b                             -- next window
local h = tonumber(redis.call("GET", a))    -- get key window
if h == nil then                            -- key window has expired
    h = e                                   -- set key window = current timestamp
end
h = math.max(h, e) + c
if h > g then
    return (h - g) * 1e6
end
redis.call("SETEX", a, f, string.format("%.6f", h))
return 0
{{< / highlight >}}


## 工程上
go-redis SDK优先使用EvalSha执行脚本，如果加载不到才使用Eval，带宽能节省就节省：
{{< highlight go "linenos=table,linenostart=1" >}}
// Run optimistically uses EVALSHA to run the script. If script does not exist
// it is retried using EVAL.
func (s *Script) Run(c scripter, keys []string, args ...interface{}) *Cmd {
	r := s.EvalSha(c, keys, args...)
	if err := r.Err(); err != nil && strings.HasPrefix(err.Error(), "NOSCRIPT ") {
		return s.Eval(c, keys, args...)
	}
	return r
}
{{< / highlight >}}

既然是高频执行的脚本，通过Evalsha可以避免每次都从文本解析生产function，假设目标实例上以及有了script的缓存，那如何保证相关的key都命中那一台实例呢？
解决方法就是使用hash tag，我们需要把key中的一部分使用{}包起来，redis将通过{}中间的内容作为计算slot的key，这样保证相关的key都会转到同一个slot中。


## 参考
* [浅析 redis lua 实现](https://mytechshares.com/2022/10/07/dive-redis-lua/)
---
title: "Rate Limiting in a Distributed System"
date: 2023-07-15T13:19:42+03:00
draft: false 
categories: ["distributed system design"]
tags: ["rate limiter"]
---

Assume that we own a suite of API based microservices that implement a discount coupon service. Rate-limiting is needed to ensure that all clients have equal access to our resources and protect our APIs from (malicious or erroneous) intensive usage.

Rate-limiting seems straightforward: we only allow a given client to perform X calls every minute. It’s quite easy to implement on a single server instance, and we can easily find libraries to do that for us. But if our API is hosted in 6 data centers (in Europe, North America, and Asia), with multiple instances in each one. This means we need some kind of distributed rate-limiting system.

## Use your load balancer
Before trying to develop your own system, it’s important to see if existing parts of your infrastructure could provide the feature you want.

So, what’s in front of all the instances on a data center and is already responsible to inspect and route traffic toward them? The load balancer. And most load balancers provide a rate-limiting feature or some kind of abstraction that can be used to implement it. For example, HAProxy has [ stick tables ](https://www.haproxy.com/blog/introduction-to-haproxy-stick-tables) that can be used to set up rate limits. It works well, it handles synchronization between instances for you and it’s already there.

In most scenarios we will have some feature requirements (dynamic limits, token introspection, …) that would mean that we need something more specific.

## Naive approaches
### Sticky sessions
Speaking of load-balancer, we don’t need a distributed rate-limiting system if a given client is not load-balanced and always interacts with a single instance 🤓. Most clients will reach the data center closer to their application (via our geo-DNS), so if we enable “stickiness” on the load balancer, a client should always reach the same instance. And this would allow us to use a simple “local” rate-limiting.

This works in theory, not in practice. The load faced by most business API systems is not constant. For example, lets say that in our demo app serving discount coupon service, the Black Friday / Cyber Week is the biggest part of the year. During this period, our imaginary development team is on alert, and we are prepared to scale our infrastructure to face the increased demand from our clients. But session stickiness and scaling don’t mix well (what’s the use of creating new instances, if all existing clients are stuck to the old ones?).

Using a smarter session stickiness, which reshuffles sessions when scaling, would help. But it means each time we scale up, clients could be switched to another instance, which has no idea how many calls the client performed on the previous instance. In essence, this would make our limit inconsistent each time we scale, allowing clients to make more calls each time our system is under pressure.

### Chatty servers
If a client can reach any of the instances, it means that the “call count” must be shared between the instances. One way to do it would be to have each instance call every other one to ask for their current count for a given client, and sum that up. Or we could do it the other way around, with each server broadcasting a “count update” to the others.

This has two main problems:

* The more instances we have, the more calls need to be made.
* Each instance needs to know the address of every other one, and this has to be updated each time the service is scaled up or down.
While this solution can be done (it’s essentially a peer-to-peer ring and a lot of systems have been implemented to do that well), it’s far from trivial.

### Kafka streams
If we don’t want to have each instance talking to the others, we could use Kafka to synchronize the counters in all the instances.

For example, each time a call would reach an instance, an event would be pushed to a topic. Those events would be aggregated with a sliding window (Kafka Stream does that quite well) and the up-to-date counts for each client for the last minute would be published on another topic. Each instance would then consume this topic to get the shared count of all clients.

The problem is that Kafka is, in essence, asynchronous. While the lag is often quite small, it will increase when the load on the API increase. And if the instances use outdated counters, they could allow calls that should be blocked.

___
___

All those solutions have something in common: they work well when everything is fine, but they degrade under heavy load. That’s how we design most of our system, and it’s perfectly fine usually. But rate-limiting is not a typical component, as its very goal is to protect the rest of the system from this heavy load.

>The goal of a rate-limiting system is to work well when the system is under heavy load. It needs to be built for the worst 1%, not the good 99%

## Distributed algorithms
What we need is a centralized and synchronous storage system and an algorithm that can leverage it to compute the current rate for each client. An in-memory cache (like Memcached or Redis) is ideal. But not all rate-limiting algorithms can be implemented with every caching system. So let’s see what kind of algorithm exists.

For simplification, we will consider that we are always trying to implement a **100 calls per minute** limit.

Let’s look at the tools at our disposal.

### Sliding window via event log
If we want to know how many calls a client did in the past minute, we could store a list of timestamps in the cache for each client. Each time a call is made, the corresponding timestamp is appended to the list. Then we can loop over each item in the list, discarding the ones that are more than a minute old and counting the ones that are not.

👍**Pros**:
* Very accurate
* Simple

👎**Cons**:
* Require strong transactional support (Two instances handling calls for the same client will want to update the same list).
* The size of the stored object (the list) can be quite big depending on the limit and the number of calls.
* Performances are not stable (More calls mean more timestamps to go through)

### Fixed window
Most distributed caching systems have specific, high-performance, abstraction for "counters" (an integer value that can be increased atomically and that is attached to a string key).

It is very easy to maintain a counter for each client, with the key **{clientId}**. But this would simply count the number of calls the client made, **since the counter creation**, not over the last minute. Using the key **{clientId}_{yyyyMMddHHmm}** would allow us to maintain a counter for each client for each calendar minute (in other words: for each fixed window of 1 minute). Looking for the counter corresponding to the current time would then give us the number of calls performed by the client this minute. And if this number is above the limit we can block the call.

>Note that this is not the same thing as **over the last minute**. If a call is made at ***07:10:23 AM***, the fixed window counter will give us the number of calls made between ***07:10:00 AM*** and ***07:10:23 AM***. But what we really want is the number of calls made between ***07:09:23 AM*** and ***07:10:23 AM***.

In a way, the fixed window algorithm ***forgets*** how many calls were made before the minute mark, so a client could theoretically perform 100 calls at ***07:09:59*** and then 100 additional calls at ***07:10:00***.

👍**Pros**:
* Very fast (a single atomic increment+read operation)
* Very basic transactional support is needed (atomic counter)
* Constant performances
* Simple

👎**Cons**:
* Inaccurate (up to x2 calls let through)

### Token bucket
> It’s 1994, and your parents dropped you off at the arcade to play Super Street Fighter II with your friends. They gave you a small bucket filled with $5 in coins and went to the bar across the street. Every hour, one of them comes and drops $5 worth of coin in the bucket. You’ve been essentially rate-limited to $5 an hour (and hopefully you became very good at Street Fighter).

That’s the main idea behind the "token bucket" algorithm: instead of a simple counter, a "bucket" is stored in the cache for each client. A bucket is an object that consists of two attributes:

* the number of remaining "tokens" (or remaining calls that can be made)
* the timestamp of the last call.
When a call is made, the bucket is retrieved. New tokens are added to the bucket depending on the amount of time between the current call and the last call. After that, if there is more than one token, it is decremented and the call can be made.

So, contrary to my "Street fighter" example, there is no "parent" that has to refill the buckets every minute. The bucket is efficiently refilled in the same operation as the token consumption (with the number of tokens corresponding to the time gap between the last call). If the last call was half a minute ago, the 100 calls per minute limit would mean 50 tokens would be added to the bucket. If a bucket is too "old" (the last call is more than 1 minute), the token count is reset to 100.

In fact, you could choose to initialize the bucket with more than 100 tokens (but have it refill at a 100 tokens/minute rate): this would be akin to a "burst" feature, where a client could go above the limit for a short period of time, but would not be able to sustain it.

>**Note:** It’s important to compute a decimal value for the tokens to be added or you risk improperly replenishing the bucket.

This algorithm offers perfect accuracy while working at constant performance. The main problem is the need for transactions (you don't want two instances updating the cached object at the same time). This is because now you have a bucket containing two attributes and you will need a transaction to operate on both of these as a unit of work.

{{< figure src="/1_iaQ44hHf8T81mrJy-Bd9ug.webp" alt="Step-by-step example of a Token bucket for a valid call and a limit of 100 calls per minute" caption="Step-by-step example of a Token bucket for a valid call and a limit of 100 calls per minute" >}}

👍**Pros**:
* Very accurate
* Fast
* Constant performances
* Tuning the initial token number allow client to “burst”

👎**Cons**:

* More complex
* Require strong transactional support
> Leaky bucket: another version of the algorithm (the “leaky bucket”) also exists. In this version, the calls pile up in the bucket and are handled at a constant rate (that matches the rate limit). If the bucket overflow, the incoming calls are refused. This is more complex to implement but allows to smooth the load on your services (which is something that you may want, or not).

### 🏆 The best?
Looking at those three algorithms, the token bucket seems to offer the best compromise of performances and accuracy. But it’s only possible if your system offers good transactional support. That is perfect if you have access to a Redis cluster (you can even implement the algorithm as a Lua script to make it run directly on your Redis cluster, for increased performances), but Memcached only supports atomic counter, not transactions.

>It’s possible to implement an [ optimistic concurrent ](https://en.wikipedia.org/wiki/Optimistic_concurrency_control) version of the token bucket using Memcached, but this would be more complex, and optimistic concurrency’s performance degrades under heavy load.

---
---

## Approximated sliding window from fixed windows
Without strong transactional support, are you condemned to use the inaccurate fixed window algorithm?

Kind of, but it can be improved. Remember that the main problem with the fixed window is that it “forgets” what happened just before the minute mark. But we still have access to this information (in the counter for the previous minute). So we could use it to estimate the number of calls in the previous minute by computing a weighted average.
{{< figure src="/1_Gxe-if0-1gVfoFP3OZ0B3w.webp" alt="60s fixed-windows composition used to approximate a sliding 60s window" caption="60s fixed-windows composition used to approximate a sliding 60s window" >}}

>**Example:** If a call is made at ***00:01:43 AM***, we increment and get the value of the "00:01" counter. As this is the current calendar minute, it will now contain the number of calls between ***00:01:00 AM*** and ***00:01:43 AM*** (the last 17 seconds have not occured yet).
>
> But we want the number of calls in the 60s sliding window, so we are missing the count for the ***00:00:43 AM*** to ***00:01:00 AM*** period. For those, we could use the "00:00" counter, and adjust it by a 17/60 factor to account for the fact that only the last 17 seconds interest us.

Under constant load the approximation is perfect. But it will be overestimated when most of the calls were made at the start of the previous minute. And it will be underestimated when most of the calls were made at the end of the previous minute.

## Let’s compare
To more accurately understand the accuracy difference, the best thing is to simulate both algorithms under the same conditions.

This first graph shows what the “fixed counter” algorithm will return with a random traffic input. The grey line is the output of a “perfect” sliding window, which at any point in time corresponds to the number of calls made in the past 60 seconds. It’s what we aim for. ***The orange-dotted line represents what the fixed window algorithm “counts” for the same traffic.***
{{< figure src="/1_f32OFMmBk1IDf1keuYmJOQ.webp" alt="60s fixed-window vs perfect sliding window" caption="60s fixed-window vs perfect sliding window" >}}
We no longer see drops around the minute marks and we can see that the new version of the algorithm more closely matches the perfect one. It sometimes goes a bit over, sometimes under, but overall a tremendous improvement.

## Diminishing returns
But can we do better?

Our approximation use only the current and previous 60s fixed windows. But instead, we could use several, smaller, sub-windows. An extreme approach would be to use sixty 1s windows to reconstruct the traffic over the last minute. Obviously, this would mean reading 60 counters for each call, which would add a big performance cost. But we could choose any fixed-window duration, and compose an approximation from them. The smaller the windows, the more of them are needed and the more precise the approximation will be.
{{< figure src="/1_ulzcfZrETy0jALTFqbPCfw.webp" alt="" caption="" >}}
Let’s look at what combining five 15s windows would do:
{{< figure src="/1_OONVSVj3p166pkw-9LRt3g.webp" alt="" caption="" >}}
As expected, the accuracy improved but is still not perfect.

We are in a classic **better accuracy = worse performance scenario**. In the end, there is no absolute best, you will have to look into your accuracy and performance requirement to find the solution that suits you the best. A simple fixed window could even be a very viable solution if the only thing you care about is protecting your service from gross overuse, without needing to consistently enforce a limit.
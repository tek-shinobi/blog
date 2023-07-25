---
title: "Golang: Distributed Rate Limiting Using Fixed Window"
date: 2023-07-25T10:41:57+03:00
draft: false 
categories: ["golang", "distributed system design"]
tags: ["golang", "rate limiter"]
---

Fixed Window rate limiter is the simplest form of rate limiting. It means that once the expiration has been set, a client that reaches the limit will be blocked from making further requests until the expiration time arrives. If a client has a limit of 50 requests every minute and makes all 50 requests in the first 5 seconds of the minute, it will have to wait 55 seconds to make another request. This is also the main downside of this implementation, it would still let a client burn through its whole limit quickly (bursty traffic) and that could still overload your service, as it could be expecting this traffic to be spread out throughout the whole limiting period.

A good part with this approach is that since at the heart of it, it involves only one step, that of incrementing the counter, this strategy can be implemented in a distributed-system-safe way by just using an atomic counter.
> Atomic operations, pipelined commands and transactions are three different ways to make a distributed-system-safe rate limiter. Atomic operations are supported by nearly all in-momory cache solutions like Redis, Memcached etc. The latter two are not that widely supported. Redis is one that supports all three. 

> We will be using redis for implementing fixed window rate limiter. 

> Redis transactions and redis pipelined commands are two separate things. They are both designed to solve the same problem: making a bunch of operations run as a unit, without back and forth client-server hops and associated race-conditions. Redis transactions are like database transactions. Like you use SQL to write a database transaction, you use Lua scripts to write code that executes in a transaction on redis side. This code executes in an atomic way, i.e. its safe from data-races. This approach gives you the most flexibility with the downside of adding another language (Lua) to your stack. The other option is to you use Pipelined commands and then execute them in an atomic way using `Exec`. This is more restrictive approach but benefit is that you can implement it using Go without needing Lua. If you have more complex requirements, Lua script based transctions are the only way to go. 

We will use  **Provider Pattern** to implement our rate limiter.

Here is how our `ratelimiter.go` file looks like:

```go
package ratelimiter

import (
	"context"
	"time"
)

// Request defines a request that needs to be checked if it will be rate-limited or not.
// The `Key` is the identifier you're using for the client making calls. This could be a user/account ID if the user is
// signed into your application, the IP of the client making requests (this might not be reliable if you're not behind a
// proxy like Cloudflare, as clients can try to spoof IPs). The `Key` should be the same for multiple calls of the
// same client so we can correctly identify that this is the same app calling anywhere.
// `Limit` is the number of requests the client is allowed to make over the `Duration` period. If you set this to
// 100 and `Duration` to `1m` you'd have at most 100 requests over a minute.
type Request struct {
	Key      string
	Limit    uint64
	Duration time.Duration
}

// State is the result of evaluating the rate limit, either `Deny` or `Allow` a request.
type State int64

const (
	Deny  State = 0
	Allow       = 1
)

// Result represents the response to a check if a client should be rate-limited or not. The `State` will be either
// `Allow` or `Deny`, `TotalRequests` holds the number of requests this specific caller has already made over
// the current period and `ExpiresAt` defines when the rate limit will expire/roll over for clients that
// have gone over the limit.
type Result struct {
	State         State
	TotalRequests uint64
	ExpiresAt     time.Time
}

type Limiter interface {
	Do(ctx context.Context, r *Request) (*Result, error)
}

```

this is how our `fixedwindow.go` looks like:
```go
package ratelimiter

import (
	"context"
	"time"

	"github.com/pkg/errors"
	"github.com/redis/go-redis/v9"
)

const (
	keyWithoutExpire = -1
)

type fixedWindowLimiter struct {
	redisClient *redis.Client
	now         func() time.Time
}

func NewFixedWindowLimiter(cl *redis.Client, now func() time.Time) Limiter {
	return &fixedWindowLimiter{
		redisClient: cl,
		now:         now,
	}
}

// Do ...
// this implementation uses a simple counter with an expiration set to the rate limit duration.
// This implementation is functional but not very effective if you have to deal with bursty traffic as
// it will still allow a client to burn through its full limit quickly once the key expires.
func (sl *fixedWindowLimiter) Do(ctx context.Context, r *Request) (*Result, error) {
	// a pipeline in Redis is a way to send multiple commands that will all be run together.
	// this is not a transaction and there are many ways in which these commands could fail
	// (only the first, only the second) so we have to make sure all errors are handled,
	// this is a network performance optimization.
	p := sl.redisClient.Pipeline()
	incrResult := p.Incr(ctx, r.Key)
	ttlResult := p.TTL(ctx, r.Key)

	if _, err := p.Exec(ctx); err != nil {
		return nil, errors.Wrapf(err, "failed at exec pipeline with key: %s", r.Key)
	}

	totalRequests, err := incrResult.Result()
	if err != nil {
		return nil, errors.Wrapf(err, "failed to increment key: %s", r.Key)
	}

	var ttlDuration time.Duration
	// we want to make sure there is always an expiration set on the key, so on every
	// increment we check again to make sure it has a TTl and if it doesn't we add one.
	// a duration of -1 means that the key has no expiration so we need to make sure there
	// is one set, this should, most of the time, happen when we increment for the
	// first time but there could be cases where we fail at the previous commands so we should
	// check for the TTL on every request.
	dur, err := ttlResult.Result()
	if err != nil || dur == keyWithoutExpire {
		ttlDuration = r.Duration
		if err := sl.redisClient.Expire(ctx, r.Key, r.Duration).Err(); err != nil {
			return nil, errors.Wrapf(err, "failed to set expiration to key: %s", r.Key)
		}
	} else {
		ttlDuration = dur
	}

	expiresAt := sl.now().Add(ttlDuration)
	requestCount := uint64(totalRequests)

	if requestCount > r.Limit {
		return &Result{
			State:         Deny,
			TotalRequests: requestCount,
			ExpiresAt:     expiresAt,
		}, nil
	}

	return &Result{
		State:         Allow,
		TotalRequests: requestCount,
		ExpiresAt:     expiresAt,
	}, nil
}
```

Note: the implementation above is ***write first*** approach. We modify the redis counter first and then decide if this request will be allowed or denied. This makes the counter increment unbounded as any request will increment it even if it is rate limited. In most cases this is OK if redis is run as a cluster and window size is small enough that counter does not overflow (uint64 is really large value).

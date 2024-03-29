---
title: "Golang Rand Concurrency Safe"
date: 2020-11-11T00:38:46+08:00
draft: false
tags: ["golang"]
---

Golang 提供标准随机包 math/rand，随机包默认会创建一个全局的对象，从源码上可以看到使用了 lockedSource 这个源，并且是基于 1 创建的 Source，也就是说每次初始化都是固定的结果，这一点和 C 一样。

```Go
var globalRand = New(&lockedSource{src: NewSource(1).(*rngSource)})

// Type assert that globalRand's source is a lockedSource whose src is a *rngSource.
var _ *rngSource = globalRand.src.(*lockedSource).src
```

在项目中为了增加随机性，比较常见的做法是每次程序启动时都把全局的随机对象重新设置一下，比如用当前时间作为随机种子，这一切看起来都非常美好，直到某个比较有想法的包出现。

```Go
import (
	"math/rand"
	"time"
)

func init() {
	rand.Seed(time.Now().UnixNano())
}
```

现在试想一下需要开发一个独立的包，基于某些原因它需要使用随机数，自然而然的想到 math/rand 包，但是基于解耦和随机安全的考虑，肯定是不能使用系统自带的全局随机对象，因为无法保证它有没有被好好设置，也不太好自己主动去设置。所以会使用系统的包重新创建随机对象，自然而然的有了如下操作。

```Go
r := rand.New(NewSource(time.Now().UnixNano()))
```

然而这里有坑，虽然标准的随机对象是并发安全的，但是标准包的 NewSource 创建的 Source 并不是并发安全的，如果你仔细阅读，就会发现注释以一个不起眼的方式告知着。

```Go
// NewSource returns a new pseudo-random Source seeded with the given value.
// Unlike the default Source used by top-level functions, this source is not
// safe for concurrent use by multiple goroutines.
func NewSource(seed int64) Source {
	var rng rngSource
	rng.Seed(seed)
	return &rng
}
```

再看一下 rngSource 的实现

```Go
type rngSource struct {
	tap  int           // index into vec
	feed int           // index into vec
	vec  [rngLen]int64 // current feedback register
}

// 省略

// Uint64 returns a non-negative pseudo-random 64-bit integer as an uint64.
func (rng *rngSource) Uint64() uint64 {
	rng.tap--
	if rng.tap < 0 {
		rng.tap += rngLen
	}

	rng.feed--
	if rng.feed < 0 {
		rng.feed += rngLen
	}

	x := rng.vec[rng.feed] + rng.vec[rng.tap]
	rng.vec[rng.feed] = x
	return uint64(x)
}
```

可以很明显的看到 rng 内部的数据没有保护起来，也就是说，当并发访问时，里面的数据会乱。乱了之后随机的结果就难说了，我们多次遇到的现象是基于浮点随机的命中率失效了，感兴趣可以并发测试一下。

了解原因之后，解决方法就好说了，可以给 Rand 上锁，或者给 Source 上锁。但是 Golang exp 已经有了支持，建议可以直接使用。

```Go
import (
	"time"

	rx "golang.org/x/exp/rand"
)

// New .
func New(source rx.Source) *rx.Rand {
	return rx.New(source)
}

// NewLockedSource .
func NewLockedSource(seed int64) *rx.LockedSource {
	source := &rx.LockedSource{}
	source.Seed(uint64(seed))
	return source
}

// NewLockedSourceTime
func NewLockedSourceTime() *rx.LockedSource {
	return NewLockedSource(time.Now().UnixNano())
}
```


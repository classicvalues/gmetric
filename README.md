# Application operation performance metric (gmetric)


[![Application operation performance metric.](https://goreportcard.com/badge/github.com/viant/gmetric)](https://goreportcard.com/report/github.com/viant/gmetric)
[![GoDoc](https://godoc.org/github.com/viant/asc?status.svg)](https://godoc.org/github.com/viant/gmetric)

This library is compatible with Go 1.12+

Please refer to [`CHANGELOG.md`](CHANGELOG.md) if you encounter breaking changes.

- [Motivation](#Motivation)
- [Usage](#Usage)
- [License](#License)
- [Credits and Acknowledgements](#Credits-and-Acknowledgements)


<a name="Motivation"></a>
## Motivation:

The goal of this project is to provide metrics counters to measure various aspect of the current and recent application behaviours with minimum overhead.  
All metrics are locking and memory allocation free to provide best possible performance.

<a name="Usage"></a>

## Usage:

##### Single metric counter


```go

package mypackage

import (
	"errors"
	"github.com/viant/gmetric"
	"github.com/viant/gmetric/stat"
	"log"
	"math/rand"
	"net/http"
	"testing"
	"time"
)

func OperationCounterUsage() {

	metrics := gmetric.New()
	handler := gmetric.Handler("/v1/metrics", metrics)
	http.Handle("/v1/metrics", handler)

	//basic single counter
	counter := metrics.OperationCounter("pkg.myapp", "mySingleCounter1", "my description", time.Microsecond, time.Minute, 2)
	go runBasicTasks(counter)

	err := http.ListenAndServe(":8080", http.DefaultServeMux)
	if err != nil {
		log.Fatal(err)
	}
}

func runBasicTasks(counter *gmetric.Operation) {
	for i := 0; i < 1000; i++ {
		runBasicTask(counter)
	}
}

func runBasicTask(counter *gmetric.Operation) {
	onDone := counter.Begin(time.Now())
	defer func() {
		onDone(time.Now())
	}()
	time.Sleep(time.Nanosecond)

}

```

```bash
open http://127.0.0.1:8080/v1/metrics/counters
## or 
open http://127.0.0.1:8080/v1/metrics/counter/mySingleCounter1
```

##### Multi metric counter




```go
package gmetric_test

import (
	"errors"
	"github.com/viant/gmetric"
	"github.com/viant/gmetric/counter/base"
	"github.com/viant/gmetric/stat"
	"log"
	"math/rand"
	"net/http"
	"testing"
	"time"
)



const (
	NoSuchKey       = "noSuchKey"

	MyStatsCacheHit       = "cacheHit"
	MyStatsCacheCollision = "cacheCollision"
)


//MultiStateStatTestProvider represents multi stats value provider
type MultiStateStatTestProvider struct{ *base.Provider }

//Map maps value int slice index
func (p *MultiStateStatTestProvider) Map(key interface{}) int {
	textKey, ok := key.(string)
	if !ok {
		return p.Provider.Map(key)
	}
	switch textKey {
	case NoSuchKey:
		return 1
	case MyStatsCacheHit:
		return 2
	case MyStatsCacheCollision:
		return 3
	}
	return -1
}


func OperationMulriCounterUsage() {
	metrics := gmetric.New()
	handler := gmetric.Handler("/v1/metrics", metrics)
	http.Handle("/v1/metrics", handler)
	counter := metrics.MultiOperationCounter("pkg.myapp", "myMultiCounter1", "my description", time.Microsecond, time.Minute, 2, &MultiStateStatTestProvider{})
	go runMultiStateTasks(counter)

	err := http.ListenAndServe(":8080", http.DefaultServeMux)
	if err != nil {
		log.Fatal(err)
	}
}

func runMultiStateTasks(counter *gmetric.Operation) {
	for i := 0; i < 1000; i++ {
		runMultiStateTask(counter)
	}

}

func runMultiStateTask(counter *gmetric.Operation) {
	stats := stat.New()
	onDone := counter.Begin(time.Now())
	defer func() {
		onDone(time.Now(), stats)
	}()

	time.Sleep(time.Nanosecond)
	//simulate task metrics state
	rnd := rand.NewSource(time.Now().UnixNano())
	state := rnd.Int63() % 3
	switch state {
		case 0:
			stats.Append(NoSuchKey)
		case 1:
			stats.Append(MyStatsCacheHit)
		case 2:
			stats.Append(MyStatsCacheHit)
			stats.Append(MyStatsCacheCollision)
	}
	if rnd.Int63() % 10 == 0 {
		stats.Append(errors.New("test error"))
	}
}
```

```bash

### all operations counters
open http://127.0.0.1:8080/v1/metrics/operations


## indivual operation multi counter
open http://127.0.0.1:8080/v1/metrics/operations/myMultiCounter1


## cumulative indivual operation multi counter metric value

open http://127.0.0.1:8080/v1/metrics/operations/myMultiCounter1/cumulative/counter
open http://127.0.0.1:8080/v1/metrics/operations/myMultiCounter1/cumulative/error
open http://127.0.0.1:8080/v1/metrics/operations/myMultiCounter1/cumulative/noSuchKey
open http://127.0.0.1:8080/v1/metrics/operations/myMultiCounter1/cumulative/cacheCollision



## recent counter 
open http://127.0.0.1:8080/v1/metrics/operations/myMultiCounter1/recent/counter

open http://127.0.0.1:8080/v1/metrics/operations/myMultiCounter1/recent/counter
open http://127.0.0.1:8080/v1/metrics/operations/myMultiCounter1/recent/error
open http://127.0.0.1:8080/v1/metrics/operations/myMultiCounter1/recent/noSuchKey
open http://127.0.0.1:8080/v1/metrics/operations/myMultiCounter1/recent/cacheCollision
```

<a name="License"></a>
## License

The source code is made available under the terms of the Apache License, Version 2, as stated in the file `LICENSE`.

Individual files may be made available under their own specific license,
all compatible with Apache License, Version 2. Please see individual files for details.


<a name="Credits-and-Acknowledgements"></a>

##  Credits and Acknowledgements

**Library Author:** Adrian Witas


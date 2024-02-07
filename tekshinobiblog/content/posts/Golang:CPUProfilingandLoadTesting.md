---
title: "Golang: CPU Profiling And Load Testing"
date: 2018-02-07T08:05:22+02:00
draft: false 
categories: ["golang" ]
tags: ["golang", "testing"]
---

>**Goal**: We want to do a CPU profile when a particular endpoint handler or set of handlers is/are called. 
>
>**Idea**: We have a service and we want to build a profile to have insights where the time is spent the most
>
>**Execution**: If we already have an endpoint that directly invokes the service, its good. Otherwise create a temporary endpoint and call that service from there. Doing this makes it easy to run a load testing tool like vegeta against that endpoint to create some load when building the profile.

Lets start with a service:
```golang
package main

import (
	"net/http"

	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
)

func main() {
	r := chi.NewRouter()
	r.Use(middleware.Recoverer)

	r.Get("/", handlerWelcome)

    fmt.Println("Server listening on :3333")
	http.ListenAndServe(":3333", r)
}

func handlerWelcome(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Write([]byte(`{"message": "welcome"}`))
}
```
Here `handlerWelcome` is the service that we want to build a profile for. It's trivial here, but in real life, instead of it just writing a welcome message, it will call the service layer method that we want to build profile for.

**Step 1**: **CPU Profile**
```golang
func main() {
    cpuprofile := "cpuprofile.prof"
	f, err := os.Create(cpuprofile)
	if err != nil {
		log.Fatal(err)
	}
	err = pprof.StartCPUProfile(f)
	if err != nil {
		log.Fatal(err)
	}
	defer pprof.StopCPUProfile()

	r := chi.NewRouter()
	r.Use(middleware.Recoverer)

	r.Get("/", handlerWelcome)

    fmt.Println("Server listening on :3333")
    go func(router *chi.Mux) {
		err := http.ListenAndServe(":3333", router)
		if err != nil {
			fmt.Println(err)
			return
		}
	}(r)

	sigch := make(chan os.Signal, 1)
	signal.Notify(sigch, os.Interrupt)

	<-sigch
}
```
If you see, I have added this code snippet for building the CPU profile:
```golang
    cpuprofile := "cpuprofile.prof"
	f, err := os.Create(cpuprofile)
	if err != nil {
		log.Fatal(err)
	}
	err = pprof.StartCPUProfile(f)
	if err != nil {
		log.Fatal(err)
	}
	defer pprof.StopCPUProfile()
```

and then I have put the invocation to server behind a go routine. This is done to make it easy to close the app and spit out the profile once the app exits. Note that profile is actually flushed to file only when you close the application.

We are now ready for Step 2.

**Step 2: Load Test**

Let's use this library for load testing: `https://github.com/tsenart/vegeta`

Create a configuration file with all the endpoints that need to be load tested. Let's call this file `load_test.conf`:
Paste the following in that file: 
```shell
GET http://localhost:3000
Content-Type: application/json
```

**Step 3: Makefile**

To make it easy, lets put everything in a makefile:
```Makefile
.PHONY: load-test
load-test:
	@echo "Running load test"
	go get github.com/tsenart/vegeta
	go run github.com/tsenart/vegeta attack -duration=10s -rate=10/s -targets=load_test.conf | tee results.bin | vegeta report

.PHONY: analysis
analysis:
	@echo "Analyzing load test results"
	go build -o main main.go
	go tool pprof main cpuprofile.prof
```

**Step 4: Run**
1. Run the application: `go run main.go`
1. Run the load test and wait till it finishes: `make load-test`
1. **important!** close the application via ctrl+C. This will flush the profile info into file used in next step.
1. `make analysis`
1. type `web` in the pprof prompt 

---

**Note:** To run POST in vegeta load tester along with payload, create the payload json file, `payload.json`:
```json
{
  "name": "Meaow",
  "some field": "I am a cat"
}
``` 

Assume we have a POST endpoint `/hello` that takes above json body as payload.

In the `load_test.conf` file:
```shell
POST http://localhost:3333/hello
Content-Type: application/json
@payload.json
```

here, `@payload.json` is path to `payload.json`. Since the file is in the project root itself, this is enough. Say if it was in `internal/hello` folder, then it would be: `@internal/hello/payload.json`
# cotton

[![Build Status](https://travis-ci.org/morikuni/cotton.svg?branch=master)](https://travis-ci.org/morikuni/cotton)
[![GoDoc](https://godoc.org/github.com/morikuni/cotton?status.svg)](https://godoc.org/github.com/morikuni/cotton)

Simple, Lightweight and Composable HTTP Handler/Middleware

## Install

```sh
go get github.com/morikuni/aec
```

## Design

cotton is designed as Filter/Middleware for `http.HandlerFunc` and work with `net/http`.  
There are 4 important types.

- http.HandlerFunc
- Filter
- Service
- RecoverFunc

Flexible `http.HandlerFunc` can be made by composing these types.

```
Filter  + Filter           => Filter
Filter  + http.HandlerFunc => Service
Filter  + Service          => Service
Service + RecoverFunc      => http.HandlerFunc
```

## Example

```go
package main

import (
	"log"
	"net/http"

	"github.com/morikuni/cotton"
)

func main() {
	// Filter + Filter => Filter
	myFilter := cotton.Filter(cotton.PanicFilter).And(cotton.MethodFilter(cotton.GET))

	// Filter + http.HandlerFunc => Service
	myService := myFilter.For(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Hello\n"))
	})

	// Service + RecoverFunc => http.HandlerFunc
	myHandler := myService.Recover(func(w http.ResponseWriter, r *http.Request, err cotton.Error) {
		switch e := err.(type) {
		case cotton.PanicOccured:
			w.WriteHeader(http.StatusInternalServerError)
			w.Write([]byte("Internal Server Error\n"))
			log.Println(e.Reason)
		case cotton.MethodNotAllowed:
			w.WriteHeader(http.StatusMethodNotAllowed)
			w.Write([]byte("Method Not Allowed\n"))
			log.Printf("expect %v but %s\n", e.Expect, e.Method)
		}
	})

	http.HandleFunc("/hello", myHandler)
	http.ListenAndServe("127.0.0.1:12345", nil)
}
```

```sh
$ go run main/main.go &

$ curl -X "GET" "http://127.0.0.1:12345/hello"
Hello

$ curl -X "PUT" "http://127.0.0.1:12345/hello"
2016/02/28 00:19:30 expect [GET] but PUT # This is from main.go
Method Not Allowed
```

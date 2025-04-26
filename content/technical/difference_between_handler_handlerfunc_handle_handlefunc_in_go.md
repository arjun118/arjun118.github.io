+++
date = '2025-04-26T18:09:29Z'
draft = false
title = 'Difference Between Handler,Handlerfunc,Handle,Handlefunc in Go'
tags=['technical']
+++

Let's see what is the difference between Handler, HandlerFunc, Handle, HandleFunc in the net/http package in go and how to use them

# Handler

![Handler definition](/images/handler.png)

> type: `interface`

as you can see Handler is an interface. this is the core contract for any object that serve or respond to an HTTP request in Go

if a particular type wants to act as an HTTP handler, it must implement the `ServeHTTP` method

## Usage

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

type myHandler struct {
	Text string
}

func (h *myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, h.Text)
}

func main() {
	mux := http.NewServeMux()
	mycustomHandler := &myHandler{Text: "this is your custom handler"}
	mux.Handle("/", mycustomHandler)
	fmt.Println("Starting server on :8081")
	log.Fatal(http.ListenAndServe(":8081", mux))

}
```

here our custom type, `myHandler` can be passed to `Handle` in place of a handler because myHandler satisfies the interface `Handler` by implementing ServeHTTP

but we haven't seen what `Handle` does, right? hold that thought we are getting there, it will all make sense in some time

# HandlerFunc

now if you are coming from a MVC background (like me, for ex: node, express for backend where you have `controllers.js`, `routes.js` etc) you can consider the `HandlerFunc` as parallel to controller functions

![alt text](/images/handlerfunc.png)

> type: `Function`

now the purpose of HandlerFunc is to act as a bridge that allows us to use ordinary Go functions (provided it has the correct signature `func(ResponseWriter, *Request)`)

the `HandlerFunc` itself implements `Handler` (i.e.,it has its own ServeHTTP method)

[refer to this link](https://cs.opensource.google/go/go/+/refs/tags/go1.24.2:src/net/http/server.go;l=2290)

![HandlerFunc Source](/images/handlerfuncsrc.png)

so when `ServeHTTP` is called on the value of the type `HandlerFunc` it simply calls the underlying function that `HandlerFunc` wraps

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func handlerHome(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Welcome to the home page!")
}

func main() {
	mux := http.NewServeMux()
	mux.Handle("/", http.HandlerFunc(handlerHome))
	fmt.Println("Starting server on :8081")
	log.Fatal(http.ListenAndServe(":8081", mux))

}
```

# Handle

> type: `Function`

![Handle signature](/images/handle.png)

this is a primary block,which is used to associate a specific URL pattern to a specific `http.Handler`

When a multiplexer receives a request whose URL path matches the pattern, it will call the `ServeHTTP` method of the registered handler

the second argument has to be a value that satisfies the http.Handler interface

we saw Handle's usage above

# HandleFunc

> type: `Function`

this is also used to register handler's but the way to do it is a little bit different and convenient

![HandleFunc Signature](/images/handlefunc.png)

this does the same job as `http.Handle`, but specifically designed to accept a regular function (with correct signature as http.`HandlerFunc`) directly as handler without requiring to manually convert this function into a` http.HandlerFunc` and pass it to `http.Handle` as an handler, more like a convenience wrapper

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func homePage(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Welcome to the Home Page!")
}

func contactPage(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Contact us at contact@example.com")
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", homePage)
	mux.HandleFunc("/contact", contactPage)
	fmt.Println("Starting server on :8081")
	log.Fatal(http.ListenAndServe(":8081", mux))
}
```

That's the end of it, hope this helps.

---
title: "Middlewares"
date: 2020-05-01T11:01:23+09:00
---
## CORS
Example to run Atreugo server with CORS middleware [![Link](https://img.icons8.com/officexs/16/000000/link.png)](https://github.com/atreugo/examples/tree/master/middlewares/cors) 

*main.go*
```go
package main

import (
	"github.com/atreugo/middlewares/cors"
	"github.com/savsgio/atreugo/v11"
)

func main() {
	config := atreugo.Config{
		Addr: "0.0.0.0:8001",
	}
	server := atreugo.New(config)

	cors := cors.New(cors.Config{
		AllowedOrigins:   []string{"http://localhost:8001", "null"},
		AllowedHeaders:   []string{"Content-Type", "X-Custom"},
		AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE"},
		ExposedHeaders:   []string{"Content-Length", "Authorization"},
		AllowCredentials: true,
		AllowMaxAge:      5600,
	})

	server.UseAfter(cors)

	server.GET("/", func(ctx *atreugo.RequestCtx) error {
		return ctx.JSONResponse(atreugo.JSON{"Method": "GET"})
	})

	server.POST("/", func(ctx *atreugo.RequestCtx) error {
		return ctx.JSONResponse(atreugo.JSON{"Method": "POST"})
	})

	server.PUT("/", func(ctx *atreugo.RequestCtx) error {
		return ctx.JSONResponse(atreugo.JSON{"Method": "PUT"})
	})

	server.DELETE("/", func(ctx *atreugo.RequestCtx) error {
		return ctx.JSONResponse(atreugo.JSON{"Method": "DELETE"})
	})

	if err := server.ListenAndServe(); err != nil {
		panic(err)
	}
}
```

## Custom 
Example to creates an atreugo server with middlewares. [![Link](https://img.icons8.com/officexs/16/000000/link.png)](https://github.com/atreugo/examples/tree/master/middlewares/custom) 

*main.go*
```go
package main

import (
    "github.com/savsgio/atreugo/v11"
)

func main() {
	config := atreugo.Config{
		Addr: "0.0.0.0:8000",
	}
	server := atreugo.New(config)

	// Register before middlewares
	server.UseBefore(beforeGlobal)

	// Register after middlewares
	server.UseAfter(afterGlobal)

	server.GET("/", func(ctx *atreugo.RequestCtx) error {
		return ctx.TextResponse("Middlewares example")
	}).UseBefore(beforeView).UseAfter(afterView)

	// Run
	if err := server.ListenAndServe(); err != nil {
		panic(err)
	}
}
```

*middlewares.go*
```go
package main

import (
	"github.com/savsgio/atreugo/v11"
	"github.com/savsgio/go-logger/v2"
)

func beforeGlobal(ctx *atreugo.RequestCtx) error {
	logger.Info("Middleware executed BEFORE GLOBAL")

	return ctx.Next()
}

func afterGlobal(ctx *atreugo.RequestCtx) error {
	logger.Info("Middleware executed AFTER GLOBAL")

	return ctx.Next()
}

func beforeView(ctx *atreugo.RequestCtx) error {
	logger.Info("Middleware executed BEFORE VIEW")

	return ctx.Next()
}

func afterView(ctx *atreugo.RequestCtx) error {
	logger.Info("Middleware executed AFTER VIEW")

	return ctx.Next()
}
```
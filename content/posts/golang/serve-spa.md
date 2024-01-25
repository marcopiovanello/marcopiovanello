---
author: Marco Piovanello
title: "Embed your single page app in Gin"
date: 2022-09-23T11:39:07+02:00
draft: false
cover: ""
tags: [
    "go",
    "golang",
    "gin",
]
categories: [
    "Software",
]
description: "Embed a single page application in Golang using Gin framework (using a single http server)."
---

## Preface

One of the many features about Go that fascinates me is the ability of embedding resource into a single binary. I tought that shipping a product/project without the necessity of installers, upackers, etc... it's an elegant and efficient way to do. Just download it, put it in a PATH folder and run it.  
Recently I developed an application to show in real-time OTPs messages sent to an android phone. I wanted to build something that have a minimal impact on the machine, cross-platform and able to handle many connections simultaneusly.  
  
I opted to use the following stack: 

-   **Go ([Gin](https://github.com/gin-gonic/gin))** to handle POST request of messages and handle WebSocket connections
-   **Redis** as a db to store messages
-   **Vue.js** to build the frontend app to show messages


## Go Embed

Go has a builtin called **[embed](https://pkg.go.dev/embed)** to, you guess it, embed resources into your binary. It exposes a directive to mark the following variable declaration as an ebedded filesystem and hold the specified resources/directories.

```go
//go:embed image/* template/*
//go:embed html/index.html
var content embed.FS
```
Let's get to code then. Our compiled frontend will be in folder named **public**

```go
package main

import (
	"embed"
    "http"
	"log"
	"os"

	"github.com/gin-gonic/gin"
)

//go:embed public/*
var public embed.FS

func main() {
	r := gin.Default()

	r.GET("/public/*f", func(ctx *gin.Context) {
		staticServer := http.FileServer(http.FS(vue))
		staticServer.ServeHTTP(ctx.Writer, ctx.Request)
	})

	r.Run()
}
```

Ok... this solution seems elegant and pretty straightforward: we initialize the public variable to hold the public folder and its subfolders, set an http handler and serve the content of the public diretory.  
This is awesome, it works, but there's a problem: the route must match the name of the folder oterwise we'll get an HTTP 404.

```go
//go:embed public/*
var public embed.FS

func main() {
	r := gin.Default()

	// THIS WILL NOT WORK
	r.GET("/web/*f", func(ctx *gin.Context) {
		staticServer := http.FileServer(http.FS(vue))
		staticServer.ServeHTTP(ctx.Writer, ctx.Request)
	})

	r.Run()
}
```

## A more generic approach (custom HTTP handler)

If the aim is serving the frontend on the index "/" while still having different endpoints for API, websocket and so on, the solution above is not optimal.  
What can be done is writing a custom handler for static assets and index.html files.

Gin lets write you your handler, so let's just to this.

```go
//go:embed public
var app embed.FS
var appFS fs.FS  //basic hierarchical file system

func main(){
	//...
	appFS, err = fs.Sub(vue, "public")
	if err != nil {
		panic("Something gone terribly wrong.")
	}
	//...

```
Then our handler
```go
r.GET("/", func(ctx *gin.Context) {
	path := filepath.Clean(ctx.Request.URL.Path)
	// if the path matches "/" then we will return the index.html file
	path = "index.html"
	path = strings.TrimPrefix(path, "/")

	// FS exposes basic filesystem functionality, Open opens the named file
	file, err := appFS.Open(path)
	if err != nil {
		ctx.AbortWithStatus(http.StatusNotFound)
		return
	}

	// Infer the content type by file's extension
	contentType := mime.TypeByExtension(filepath.Ext(path))
	ctx.Writer.Header().Set("Content-Type", contentType)

	// Assets might be cached
	if strings.HasPrefix(path, "assets/") {
		ctx.Writer.Header().Set("Cache-Control", "public, max-age=2592000")
	}

	stat, err := file.Stat()
	if err == nil && stat.Size() > 0 {
		ctx.Writer.Header().Set("Content-Length", fmt.Sprintf("%d", stat.Size()))
	}

	// fianlly write the file in the http stream
	io.Copy(ctx.Writer, file)
})
```

This is pretty bulky so it can be extracted to a func and made more generic.  

## TLDR, final solution

### handlers.go
```go
package main

func spaHandler(fs *fs.FS, index bool) func(ctx *gin.Context) {
	return func(ctx *gin.Context) {
		path := filepath.Clean(ctx.Request.URL.Path)
		if index {
			if path == "/" {
				path = "index.html"
			}
		}
		path = strings.TrimPrefix(path, "/")

		file, err := (*fs).Open(path)
		if err != nil {
			ctx.AbortWithStatus(http.StatusNotFound)
			return
		}

		contentType := mime.TypeByExtension(filepath.Ext(path))
		ctx.Writer.Header().Set("Content-Type", contentType)

		if strings.HasPrefix(path, "assets/") {
			ctx.Writer.Header().Set("Cache-Control", "public, max-age=2592000")
		}

		stat, err := file.Stat()
		if err == nil && stat.Size() > 0 {
			ctx.Writer.Header().Set("Content-Length", fmt.Sprintf("%d", stat.Size()))
		}

		io.Copy(ctx.Writer, file)
	}
}
```
### main.go
```go
import (
	"embed"
	"io/fs"
	"http"

	"github.com/gin-gonic/gin"
)

//go:embed public
var app embed.FS
var appFS fs.FS  //basic hierarchical file system

func main(){
	appFS, err = fs.Sub(vue, "public")
	if err != nil {
		panic("Something gone terribly wrong.")
	}
	// ...
	r.GET("/", spaHandler(&vueFS, true))
	r.GET("/assets", spaHandler(&vueFS, false))
	// ...
}
```
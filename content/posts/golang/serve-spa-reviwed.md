+++
title = "Embed your single page app in Golang"
date = "2023-01-02T23:16:08+01:00"
author = "Marco Piovanello"
authorTwitter = "" #do not include @
cover = ""
tags = ["go", "golang", "react", "embed"]
keywords = ["golang", "embed"]
description = "A reworked approach to embed single page applications in Golang."
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

*Image by [Tim Zänkert](https://unsplash.com/it/@znkrt)*

**Continuum of [this article](../serve-spa/).**

## Preface

As mentioned in the previous version of this article, I love the idea of having a nice
modern ui embedded in a single binary. It's clean, elegant and saves a lot of time when
deploying the applcation.

This time i built a simple gallery viewe named `Fuu`.

## Application Structure

FS <--> Golang Fileserver <-- Browsing API  <--> React App 

- Fileserver exposes static resources (photos)
- Background process that generates thumbnails
- HTTP Methos to retrieve the directory structure
- React App for viewing photos grouped by directory

## Embed React App

Golang http package exposes the `http.FileServer` method that takes a FS as parameter

```go
//go:embed frontend/dist
var app embed.FS

// creates a hierarchical FS starting from the specified subfolder
appBuild, _ := fs.Sub(*app, "frontend/dist")

// Serve a FS
http.Handle("/", http.FileServer(http.FS(appBuild)))
```

This actually works pretty well!

But there is a catch: you **must** neither serve resources from the index route.

```go
//go:embed frontend/dist
var app embed.FS

// creates a hierarchical FS starting from the specified subfolder
appBuild, _ := fs.Sub(*app, "frontend/dist")

// Serving a directory
http.Handle("/res/", http.StripPrefix("/res", http.FileServer(http.Dir("./res"))))
http.Handle("/app/", http.StripPrefix("/app", http.FileServer(http.FS(appBuild))))
```

But i want to serve my react app from the index route and the static resources from the /static route. This is a job for a http middleware.

## A job for a HTTP middleware

Essentially the middleware need's to:

1. detect if we're asking for `index.html`, if so return the document
2. if we're asking for an asset open the embedded FS to the correct folder
3. apply the correct MIME
4. return the buffer and also might set a cache-control header

## TLDR;

`react_handler.go` 

```go
package pkg

func reactHandler(fs *fs.FS) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodGet {
			http.Error(w,
			  http.StatusText(http.StatusMethodNotAllowed), 
			  http.StatusMethodNotAllowed,
			)
			return
		}

		path := filepath.Clean(r.URL.Path)

		// Frontend routes must be known.
		// Add as many routes as frontend has.
		// The frontend router will take care of redirecting to the correct view/component
		if path == "/" || strings.HasPrefix(path, "/someroute"){
			path = "index.html"
		}

		path = strings.TrimPrefix(path, "/")

		file, err := (*fs).Open(path)

		if err != nil {
			if os.IsNotExist(err) {
				log.Println("file", path, "not found:", err)
				http.NotFound(w, r)
				return
			}
			log.Println("file", path, "cannot be read:", err)
			http.Error(w,
			  http.StatusText(http.StatusInternalServerError), 
				http.StatusInternalServerError,
			)
			return
		}

		contentType := mime.TypeByExtension(filepath.Ext(path))
		w.Header().Set("Content-Type", contentType)

		if strings.HasPrefix(path, "assets/") {
			w.Header().Set("Cache-Control", "public, max-age=2592000")
		}

		stat, err := file.Stat()
		if err == nil && stat.Size() > 0 {
			w.Header().Set("Content-Length", fmt.Sprintf("%d", stat.Size()))
		}

		io.Copy(w, file)
	})
}
```

`server.go`

```go
//go:embed frontend/dist
var app embed.FS

// creates a hierarchical FS starting from the specified subfolder
appBuild, _ := fs.Sub(*app, "frontend/dist")

// Serving the react app build
http.Handle("/", reactHandler(&appBuild))
// Now i can serve a different folder while keeping the react app on "/"
http.Handle("/media/", http.StripPrefix("/media", http.FileServer(http.Dir("./media"))))
```

Happy Coding!

And happy new year!
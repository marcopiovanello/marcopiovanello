+++
title = "Concurrent thumbnail generator"
date = "2023-01-08T22:01:28+01:00"
author = "Marco Piovanello"
authorTwitter = "" #do not include @
cover = ""
tags = ["go", "golang", "concurrency"]
keywords = ["concurrency"]
description = "Create a thumbnail generator easily with golang concurrency structures."
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

*Image by [Markus Spiske](https://unsplash.com/it/@markusspiske)*

## Preface

This article is part of the series regarding `Fuu` a simple self-hosted photo viewer.

The aim of this module is design a structure that from a list of folders generates a thumbnail (of the first file) for each folder.
By leveraging the concurrent nature of Go it's easy to efficiently use all the CPU power of our machine. By efficiently use I mean using each core for a thread of execution.

## Overwiew
I opted to use imagemagick for the image thumbnail generator and ffmpeg for the video ones.

At the first glance I though to spawn a `goroutine` for each job and leave the management of spawned process to the operating system. Doing so easily hogs the CPU and spawns a terrible amount of processes that also takes up memory space.

Thankfully Go's standard library offers many concurrency structures.

## Goroutines, channels and building a pipeline

Go's channel can be either **buffered** or unbuffered.
Buffering a channel allows you to specify how many items can be sent at one before the channel blocks.

With an `unbuffered` channel the channel will not block and len(items) goroutines will be spawned at once.

```go
items := []int{2,4,5,6,5,3,1,7,2,4,5,6,5,3,1,9}

ch := make(chan struct{})

for _, n := range items {
    ch <- 1struct{}{}
    go func(i int){
        fmt.Println("doubled", i*2)
        <-ch
    }(n)
}
```

In this example a `buffered` channel of 8 items is used to double 8 ints a time.

```go
items := []int{2,4,5,6,5,3,1,7,2,4,5,6,5,3,1,9}

ch := make(chan struct{}, 8)

for _, n := range items {
    ch <- struct{}{}
    go func(i int){
        fmt.Println("doubled", i*2)
        <-ch
    }(n)
}
```

**Essentially this is a 8 stage pipeline.**

The result is the same but in the second example we're in a sense synchronizing how many goroutines will be executed, which is just what we need.

The goal is indeed make use of *n* cores to launch *n* instances of imagemagick or ffmpeg.

## The actual implemeantation

In the following code I will use [this library](https://github.com/marcopeocchi/fazzoletti) I coded which ports some Javascript's array operation in Go.

Feel free to remove the library and convert the code in plain Go as might be considered non *idiomatic*.

```go
package workers

import (
	"fmt"
	"io/fs"
	"log"
	"os"
	"os/exec"
	"runtime"
)

type Generator struct {
	path   string
	ext    string
	images []Job
}

type Job struct {
	InputFile  string
	OutputFile string
	IsImage    bool
}

func (g *Generator) convert(j *Job) error {
	if j.IsImage {
		cmd = exec.Command(
			"convert", j.InputFile,
			"-geometry", fmt.Sprintf("x%d", t.ImgHeight),
			"-format", "webp",
			"-quality", "75",
			j.OutputFile,
		)
	} else {
		cmd = exec.Command(
			"ffmpeg",
			"-i", j.InputFile,
			"-ss", "00:00:01.000",
			"-vframes", "1",
			j.OutputFile,
		)
	}

	// free
	if err := cmd.Wait(); err != nil {
		return err
	}
	
	log.Println("Generated thumbnail for", j.InputFile)
	return nil
}

func (g *Generator) process() {
	var (
		wg       sync.WaitGroup
		pipeline = make(chan struct{}, runtime.NumCPU())
	)

	// A WaitGruop block the current gorouting until its internal 
	// counter reaches 0
	wg.Add(len(g.images))

	for i, image := range g.images {
		go func(job *Job) {
			pipeline <- struct{}{}
			g.convert(job)
			<-pipeline
			wg.Done()
		}(&image)
	}

	// Everything's processed
	wg.Wait()
}
```

## Conclusions

Now the program scales linearly with the number of cores of the machine.

I reccomend this [slides about Go Concurrency Patterns](https://go.dev/talks/2012/concurrency.slide#1) by Rob Pike which deep dives on more 
patterns.
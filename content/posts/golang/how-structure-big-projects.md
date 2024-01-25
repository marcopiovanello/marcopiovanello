+++
title = "How I structure big Go web projects"
date = "2024-01-24T11:41:06+01:00"
author = "Marco Piovanello"
cover = ""
tags = ["go", "golang"]
keywords = ["project", "structure", "architecture", "fx", "dependency injection"]
description = "How i use dependency injection and domain driven architecture to modularize big projects"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

It's the 1 million dollars question: "what's the best way to structure your Go web project?".  
There's none.  
But you can adapt one to your needs. One way I found working for me it's the following: a domain-ish driven architecture with a modular approach.

The main star here is [fx](https://github.com/uber-go/fx) by Uber.

## What's fx?
Fx is *A dependency injection based application framework for Go.* according to fx's GitHub page.  
It can helps you divide your project in **modules** and dependecies in **providers** and manage theirs entire **life cylcles**.

What's really nice about fx is that there's no build time and `//go:generate` / `//go:build` / code genaration shenanigans unlike [wire](https://github.com/google/wire) and most importantly no **reflection**.

## Defining a module structure

A module in follows this structure:
```
📦internal
 ┣ 📂user           ->  module package
 ┃ ┣ 📂db           ->  contains db models and queries
 ┃ ┃ ┣ 📜db.go
 ┃ ┃ ┣ 📜models.go
 ┃ ┃ ┗ 📜query.sql.go
 ┃ ┣ 📂repository   ->  repository handles db accesing logic
 ┃ ┃ ┗ 📜user.go
 ┃ ┣ 📂rest
 ┃ ┃ ┗ 📜handler.go ->  rest handler functions
 ┃ ┣ 📂service      ->  service handles domain level logic
 ┃ ┃ ┗ 📜user.go
 ┃ ┣ 📜module.go    ->  defines and exports a module
 ┃ ┣ 📜query.sql    ->  sqlc queries
 ┃ ┣ 📜schema.sql   ->  sqlc db models
 ┃ ┗ 📜sqlc.yaml    ->  generates db package
 ┣ 📜user.go        ->  domain level structs and validate func
 ┗ 📜validation.go  ->  module agnostic validation utils (regexes, ...)

```
It ensures isolated and self-containing modules for each domain.

Let's look at `module.go` file and we'll top-down from there.
```go
package user

import (
  "github.com/marcopeocchi/fx-sample-app/internal/user/repository"
  "github.com/marcopeocchi/fx-sample-app/internal/user/rest"
  "github.com/marcopeocchi/fx-sample-app/internal/user/service"

  "github.com/gin-gonic/gin"
  "go.uber.org/fx"
)

var Module = fx.Module(
  "user",
  fx.Provide(repository.NewRepository),
  fx.Provide(service.NewService),
  fx.Provide(rest.NewHandler),
  fx.Invoke(func(h *rest.Handler, g *gin.RouterGroup) {
    h.RegisterRoutes(g)
  }),
)
```

`user.go`

This file contains all domain level data and validation functions.
```go
package internal

import (
  "time"

  validation "github.com/go-ozzo/ozzo-validation/v4"
  "github.com/go-ozzo/ozzo-validation/v4/is"
)

type User struct {
  ID        int64     `json:"id"`
  Admin     bool      `json:"admin"`
  Email     string    `json:"email"`
  Username  string    `json:"username"`
  Password  string    `json:"password"`
  CreatedAt time.Time `json:"createdAt"`
}

func (u User) Validate() error {
  return validation.ValidateStruct(&u,
    validation.Field(&u.Email, validation.Required, is.Email),
    validation.Field(&u.Username, validation.Length(4, 32), validation.Required),
    validation.Field(&u.Password, validation.Length(8, 0), validation.Required),
  )
}
```

`repository/user.go`

A repository handles all db/data access layer functionalites.  
Theoretically if you need a caching layer it should be implemented here.
```go
type Repository struct {
  conn *pgxpool.Pool
  q    *db.Queries
}

func NewRepository(conn *pgxpool.Pool) *Repository {
  return &Repository{
    conn: conn
    q:    db.New(conn)
  }
}

func (r *Repository) GetUserById(ctx context.Context, id int64) (*db.User, error) {
  model, err := r.q.GetUserById(id)
  if err != nil {
    return nil, err
  }

  return &model, err
}
``` 

`service/user.go`

A service needs to convert objects to the domain layer and call repository or other repositories
in order to access data.
```go
type Service struct {
  repo *repository.Repository
}

func NewService(r *repository.Repository) *Service {
  return &Service{
    repo: r
  }
}

func (s *Service) GetUserById(ctx context.Context, id int64) (*internal.User, error) {
  model, err := s.repo.GetUserById(id)
  if err != nil {
    return nil, err
  }
  // convert it to a domain level user struct
  return &internal.User{
    Id:        model.Id
    Admin:     model.Admin
    Username:  model.Username,
    Password:  model.Password,
    CreatedAt: model.CreatedAt.Time
    Email:     model.Email,
  }, nil
}
``` 

`rest/handler.go`

A REST handler it quite self-explanatory provides a REST interface to access data. It is 
framework agnostic, in this example I'll use Gin, but you can use straight `net/http` which is what
I will also normally do.
```go
type Handler struct {
  service *service.Service
}

func NewHandler(s *service.Service) *Handler {
  return &Handler{
    service: s
  }
}

func (s *Service) getUserById(router *gin.RouterGroup) {
  router.GET("/user/:id", func(ctx *gin.Context) {
    id, err := strconv.ParseInt(ctx.Param("id"), 10, 64)
    if err != nil {
      ctx.AbortWithStatusJSON(http.StatusBadRequest, err.Error())
      return
    }

    user, err := h.service.GetUserById(ctx, id)
    if err != nil {
      ctx.AbortWithStatusJSON(http.StatusInternalServerError, err.Error())
      return
    }

    ctx.JSON(http.StatusOK, users)
  })
}

func (h *Handler) RegisterRoutes(router *gin.RouterGroup) {
  h.getUserById(router)
}
``` 

Let's define an app and it's dependencies to provide/inject and wrap it up.

`main.go`

```go
func main() {
  fx.New(
    fx.Provide(newLogger),
    fx.Provide(
      newGin,
      newApiV1Group,
      newPostgresPool,
    ),
    user.Module,
    fx.Invoke(run),
  ).Run()
}

func newPostgresPool(lc fx.Lifecycle, logger *zap.Logger) *pgxpool.Pool {
  dsn := os.Getenv("POSTGRES_DSN")

  ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
  defer cancel()

  pool, err := pgxpool.New(ctx, dsn)

  if err != nil {
    logger.Sugar().Fatalln(err)
  }

  lc.Append(fx.Hook{
    OnStop: func(ctx context.Context) error {
      cancel()
      pool.Close()
      return nil
    },
  })
  return pool
}

func newGin() *gin.Engine {
  return gin.Default()
}

func run(r *gin.Engine) {
  r.Run()
}

func newApiV1Group(r *gin.Engine) *gin.RouterGroup {
  r.Use(gzip.Gzip(gzip.DefaultCompression))
  r.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"*"},
    AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
    AllowHeaders:     []string{"Origin", "Content-Type"},
    ExposeHeaders:    []string{"Content-Length", "Content-Type"},
    AllowCredentials: true,
  }))
  return r.Group("/api/v1")
}

func newLogger(lc fx.Lifecycle) *zap.Logger {
  logger, err := zap.NewProduction()
  if err != nil {
    os.Exit(1)
  }
  lc.Append(fx.Hook{
    OnStop: func(ctx context.Context) error {
      return logger.Sync()
    },
  })
  return logger
}
```

## Conclusions
This structure really shines with many modules 4+ I'll say.  
Adding or removing dependencies is quite easy and straightforward, let's say i wanted to add a **redis** client:
what I need is creating a new constructor function and pass it to the **fx.Provide** as an additional argument, then
the repository constructor will also be modified to accept a the redis client and voilà.
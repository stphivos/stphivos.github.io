---
layout:      post
image:       ''
description: 'Example layered microservice in Go, using gRPC, MongoDB and Docker.'
title:       "A Go microservice using gRPC And MongoDB"
categories:  blog
tags:        microservices go grpc mongodb
date:        2018-01-22 17:00:00 +0200
---

<img src="{{ "/assets/img/2018-01-22/microservices.png"}}" alt="" style="display:block; margin: 0 auto;">

<p style="text-align:center; font-size:small;">Image from elastic.io <a href="https://www.elastic.io/breaking-down-monolith-microservices-and-self-contained-systems/" target="_blank">article.</a></p>

In a microservice architecture, a single application is composed from a set of small modular services. Each one is independently deployable, runs as a separate process and communicates through lightweight mechanisms within the ecosystem.

[gRPC](https://grpc.io/docs/quickstart/) is a high performance open-source RPC framework which enables such an interoperability. It relies on [protocol buffers](https://developers.google.com/protocol-buffers/) for interface definition (IDL) and payload serialization into an efficient binary format.

Containers are a great way to deploy microservices because they provide several benefits such as portability, automation, state management, security via image registry vulnerability scanning etc.

[MongoDB](https://www.mongodb.com/) is a general purpose document-based database system. It offers low-latency high data throughput by using replica sets to scale reads and sharding to scale writes. It is suited for a broad range of use cases, some of which are: recording frequently generated machine/sensor or other schema-less data and building reports on large volumes of data which updates in real time.

This blog post is my attempt to go over the implementation of such a service using Google's [Go](https://tour.golang.org/) programming language which is rapidly gaining [popularity](https://opensource.com/article/17/11/why-go-grows) in the last few years.

## Prerequisites

It is assumed that you already have some familiarity with `Go`. It's also helpful to have some understanding of the `gRPC` framework and containers - particularly [Docker](https://docs.docker.com/get-started/).

## Objectives

- [ ] Write service main entry point
- [ ] Add a server layer that uses gRPC
- [ ] Add a database layer that uses MongoDB
- [ ] Package service container image using Docker
- [ ] Test service using [grpcc](https://github.com/njpatel/grpcc) cli interface

## Project files

<img src="{{ "/assets/img/2018-01-22/project.png"}}" alt="" height="300">

The complete project source code can be downloaded from this [GitHub repo](https://github.com/stphivos/todo-api-go-grpc).

## Entry point

First thing we need to do is to define the entry point of our server in package `main` so that when the executable is run the server starts listening for client requests.

{% highlight go linenos %}
//  main.go

package main

func main() {
    config := getConfig()
    srv := getServer(config)

    err := srv.Start()
    if err != nil {
        log.Fatal(err)
    }
}

func getConfig() *models.Config {
    config := new(models.Config)
    err := configor.Load(config, "config.yml")
    if err != nil {
        log.Fatal(err)
    }
    return config
}

func getServer(config *models.Config) server.Runner {
    srv, err := server.Create(config)
    if err != nil {
        log.Fatal(err)
    }
    return srv
}
{% endhighlight %}

**line 6:** The `config` object contains the variables we need to initialize the application - such as host names, ip addresses, etc. In this case, we use the [configor](https://github.com/jinzhu/configor) library which combines variables we defined in a `config.yml` with environment variables we set in our `Config` struct:

**line 7:** The `srv` object is of interface type `Runner` defined in our `server` package. It contains only one method: `Start()` which returns a result of type `error`.

**line 9:** We call the `start` method which we expect to block for the lifecycle of the program.

Finally we call `log.Fatal` if there was an error to log and exit with status code `1`.

- [x] **Write service main entry point**

## Server layer

In this package the first things we need to define are the interface `Runner` mentioned above, and a factory function which creates the appropriate concrete implementation based on the `config` parameter:

{% highlight go linenos %}
//  runner.go

package server

type Runner interface {
    Start() error
}

func Create(config *models.Config) (Runner, error) {
    var srv Runner
    var err error

    switch config.Server.Type {
    case "grpc":
        srv, err = grpc.NewRunner(config)
    default:
        err = fmt.Errorf("Server type %v is not supported", config.Server.Type)
    }

    return srv, err
}
{% endhighlight %}

**line 13:** In our case `config.Server.Type` will be set to `grpc`, so we will call the `NewRunner` constructor defined in our `grpc` sub-package as shown below:

{% highlight go linenos %}
//  grpc.go

package grpc

type Runner struct {
    Config   *models.Config
    Database database.Handler
}

func NewRunner(config *models.Config) (*Runner, error) {
    db, err := database.Create(config)
    runner := &Runner{
        Config:   config,
        Database: db,
    }
    return runner, err
}
{% endhighlight %}

**line 5:** The `grpc.Runner` struct requires fields `Config` and `Database` which are used later when we start the server and query the database, respectively.

**line 10:** Notice the `NewRunner` constructor creates a `Handler` object defined in the `database` package by also following the factory pattern.

Before actually adding any functionality to our grpc `Runner`, let's first create a file named `todos.proto` to define our service's interface to the outside world.

The contents of the `proto` file are shown below:

{% highlight protobuf %}
syntax = "proto3";

package grpc;

service Todos {
    rpc GetTodos(Request) returns (Response) {}
}

message Request {
    string token = 1;
}

message Response {
    message Todo {
        string id = 1;
        string title = 2;
        string tag = 3;
        int32 priority = 4;
    }

    repeated Todo todos = 1;
}
{% endhighlight %}

**Syntax:** We are using [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview) version 3.

**Todos:** We define a service named `Todos`, with a single method `GetTodos` accepting an input parameter of type `Request` and returning a result of type `Response`.

**Request:** The `Request` only contains one field which we won't be using in this tutorial. In a production setting we would use that as a form of user authentication.

**Response:** The `Response` contains a list of `Todo` messages with the fields in the order declared above. Notice how we can define nested messages.

Now that we have finished writing the `proto` file, we are ready to generate our Go interface and stubs using the protocol buffer compiler [protoc](https://github.com/google/protobuf) and the Go gRPC [plugin](https://godoc.org/github.com/golang/protobuf/protoc-gen-go):

```sh
protoc -I ./server/grpc/ ./server/grpc/todos.proto --go_out=plugins=grpc:.
```

Next, let's continue with the grpc `Runner` implementation:

{% highlight go linenos %}
//  grpc.go

package grpc

// func NewRunner ...

func (srv *Runner) Start() error {
    listener, err := net.Listen("tcp", srv.Config.Server.Host + ":" + srv.Config.Server.Port)
    if err != nil {
        return err
    }

    grpcServer := grpc.NewServer()
    RegisterTodosServer(grpcServer, srv)

    return grpcServer.Serve(listener)
}

func (srv *Runner) GetTodos(ctx context.Context, req *Request) (*Response, error) {
    todos, err := srv.Database.GetTodos()
    if err != nil {
        log.Println(err)
        return nil, err
    }

    res := &Response{
        Todos: srv.mapTodos(todos...),
    }

    return res, err
}
{% endhighlight %}

**line 7** Method `Start` has a pointer receiver to the `Runner` struct above, so that it implicitly implements the `Runner` interface defined in the `server` package earlier.

**line 8** We call `net.Listen` to receive an object of type `Listener` configured to use the `tcp` communication protocol and a host name/port combination such as `:8000`. This means it will listen on all available unicast and anycast IP addresses of the local system on port `8000`.

**line 13** We call the `NewServer` constructor defined in the `google.golang.org/grpc` package - not to be confused with our `grpc` package above, to obtain an object of type `Server`. That represents an RPC service's specification and contains methods/fields which abstract its implementation.

**line 14** We need to register `grpcServer` with our server implementation `srv` - defined in the method receiver, before invoking the service. It is essential because the provided grpc server needs to map the service's rpc methods (from the `.proto` file) to our handlers that implement them. In this case we only need one handler, method `GetTodos` in line 17.

**line 16** Finally we call the grpc server's `Serve` method with our listener above and we are ready to accept requests.

Method `GetTodos` will be called by the grpc server every time a client makes a request to our service. We are then calling the runner's database handler (which we will look at its implementation in the next section) to retrieve a slice of `Todo` objects defined in our `models` package. We need to convert those objects to type `Response_Todo` which is a struct generated by the protocol buffer compiler in the previous step. Finally we construct the response with the mapped todos attached.

- [x] **Add a server layer that uses gRPC**

## Database layer

Before looking at our `database` package, let's first have a quick look at the `models` package where we define the `Todo` struct type:

```go
//  models.go

package models

type Todo struct {
    ID       bson.ObjectId `bson:"_id"`
    Title    string        `bson:"title"`
    Tag      string        `bson:"tag"`
    Priority int32         `bson:"priority"`
}
```

Apart from the field declarations, it also contains struct tags to map to the target MongoDB collection's fields.

`BSON` is the binary serialization format used by MongoDB to store document field data. The full reference of available types can be found in the [manual](https://docs.mongodb.com/manual/reference/bson-types/).

In the `database` package, similarly to the `server` package, we will be defining a generic interface which we will call `Handler` and a factory function `Create` to determine which type of database handler needs to be initialized based on the `config` parameter:

{% highlight go linenos %}
//  handler.go

package database

type Handler interface {
    GetTodos() ([]models.Todo, error)
}

func Create(config *models.Config) (Handler, error) {
    var db Handler
    var err error

    switch config.Database.Type {
    case "mongo":
        db, err = mongo.NewHandler(config)
    default:
        err = fmt.Errorf("Database type %v is not supported", config.Database.Type)
    }

    return db, err
}
{% endhighlight %}

**line 13:** In our case `config.Server.Type` will be set to `mongo`, so we will call the `NewHandler` constructor defined in our `mongo` sub-package as shown below:

{% highlight go linenos %}
//  mongo.go

package mongo

type Handler struct {
    *mgo.Session
}

func NewHandler(config *models.Config) (*Handler, error) {
    session, err := mgo.Dial("mongodb://" + config.Database.Host + ":" + config.Database.Port)
    handler := &Handler{
        Session: session,
    }
    return handler, err
}
{% endhighlight %}

**line 5:** The `mongo.Handler` struct only requires the field `Session` from the `gopkg.in/mgo.v2` package which is used later to connect to the MongoDB.

**line 10:** The `mgo.Dial` method returns a new session to the mongo cluster. We typically only need to call this method once for a given cluster.

Let's continue with the mongo `Handler` implementation where we actually use the `Session` field to make queries to the database:

{% highlight go linenos %}
//  mongo.go

package mongo

// func NewHandler ...

func (db *Handler) GetTodos() ([]models.Todo, error) {
    session := db.getSession()
    defer session.Close()

    todos := []models.Todo{}
    err := session.DB("TodosDB").C("todos").Find(nil).All(&todos)

    return todos, err
}

func (db *Handler) getSession() *mgo.Session {
    return db.Session.Copy()
}
{% endhighlight %}

**line 8:** The `getSession` method of our database handler calls session's method `Copy` to get a new copy of the session, and we call it every time before using it. This is to ensure the pool of connections to the cluster is managed appropriately.

**line 12:** This is where we connect to the database named `TodosDB` and using collection `todos` with no query arguments to get all the todo records. Finally we pass the memory address of our empty slice of todos to be populated by method `All`, which is what actually triggers the call to the database.

- [x] **Add a database layer that uses MongoDB**

## Container image

It is time to package our microservice into a container to be deployed to a cluster somewhere like Kubernetes or Mesos.

We will be writing a `Dockerfile` which contains successive instructions on top of a pre-packaged base box to assemble the image:

```dockerfile
# Stage 0
FROM golang:1.9.2-alpine AS build

ARG PROJECT
ENV PROJECT_SRC=/go/src/${PROJECT}

RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

COPY Gopkg.lock Gopkg.toml ${PROJECT_SRC}/
WORKDIR ${PROJECT_SRC}

RUN dep ensure -vendor-only

COPY . ${PROJECT_SRC}/
COPY ./config.yml /project/
RUN go build -o /project/server

# Stage 1
FROM alpine:latest
COPY --from=build /project /project
WORKDIR /project
ENTRYPOINT ["./server"]
```

We used `multi-stage builds` to significantly reduce the size of the final image, since all we need is a lightweight box to run our executable.

**Stage 0** We build the project output from the source and dependencies (vendor packages) and copy the output binary `server` and `config.yml` to `/project`.

**Stage 1** We copy the `/project` directory from `stage 0` to a new smaller base image (only 17.7MB!) in this last stage and define the entry-point to our service.

For more info about best practices for writing Dockerfiles check out the [user guide](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#use-multi-stage-builds).

To build the image we need to supply the `PROJECT` build argument that represents the base package of our service, in this case `github.com/stphivos/todo-api-go-grpc`. Change that to match your own project's package.

We also tag the image with repository name `todo-api-go-grpc`. If pushed to a cloud provider, the tag needs to be prefixed with a public container registry such as `quay.io/<username>/todo-api-go-grpc`.

```sh
docker build --build-arg PROJECT=github.com/stphivos/todo-api-go-grpc -t todo-api-go-grpc .
```

- [x] **Package service container image using Docker**

## Test service

The final step is to test our service using a gRPC client.

But first, we start a `MongoDB` container using the [image](https://hub.docker.com/_/mongo/) from docker hub.

```sh
docker run -p 27017:27017 --name mongo -d mongo
```

Find the running container and start a `Bash` session with it:

```sh
docker exec -it `docker ps -f name=mongo -q` bash
```

Add some records by connecting to the MongoDB Shell:

```sh
root@c699564f0238:/# mongo

> use TodosDB
switched to db TodosDB

> db.todos.insert({ "title": "Walk the dog", "tag": "daily", "priority": 1 })
WriteResult({ "nInserted" : 1 })

> db.todos.insert({ "title": "Buy groceries", "tag": "weekly", "priority": 1 })
WriteResult({ "nInserted" : 1 })
```

Exit both the MongoDB Shell and bash session with the container.

To run our image and make it accessible through the host machine we need to expose the `host:container` ports the service will be listening on. Optionally, we can supply the set of `environment variables` supported by our `Config` struct as shown below:

```sh
docker run -it -p 8000:8001 --env DB_HOST=192.168.99.100 --env SERVER_PORT=8001 todo-api-go-grpc
```

If everything worked as expected we should be able to see output similar to:

```sh
2018/01/22 17:00:00 Starting Todos service...
2018/01/22 17:00:00 Configuration: {% raw %}{{grpc  8001} {mongo 192.168.99.100 27017}}{% endraw %}
2018/01/22 17:00:00 Accepting requests:
```

Now let's use the [grpcc](https://github.com/njpatel/grpcc) cli interface to get the todos from our gRPC service:

```sh
grpcc --proto ./server/grpc/todos.proto --address 192.168.99.100:8000 -i
```

We should be now presented with the grpcc interface:

<img src="{{ "/assets/img/2018-01-22/grpcc.png"}}" alt="">

Invoke the client's `getTodos` method with a sample request object and `printReply` convenience callback for printing the response:

```json
Todos@192.168.99.100:8000> client.getTodos({ token: 'xyz' }, printReply)
EventEmitter {}
Todos@192.168.99.100:8000>
{
  "todos": [
    {
      "id": "5a67c5312990d8cba2301512",
      "title": "Walk the dog",
      "tag": "daily",
      "priority": 1
    },
    {
      "id": "5a67c5512990d8cba2301513",
      "title": "Buy groceries",
      "tag": "weekly",
      "priority": 1
    }
  ]
}
```

- [x] **Test service using grpcc cli interface**

## Wrap-up

If you have any problems following this tutorial, please leave a comment bellow or use the [Github Issue Tracker](https://github.com/stphivos/todo-api-go-grpc/issues) of this repo.

In a future post I will go through deploying this service on Kubernetes, hope you follow along!

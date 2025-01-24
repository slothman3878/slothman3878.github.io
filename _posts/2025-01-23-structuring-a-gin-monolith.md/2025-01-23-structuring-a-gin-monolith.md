---
layout: post
title:  "Structuring a Gin Monolith"
date:   2025-01-23 22:00:00 -0700
categories: go gin backend
math: true
---

I just bombed an amazon interview and trying to clear my head. Here is something I've spent way to much time at work thinking about because I couldn't get a clear answer from anywhere. So, I figured I might as well write it down in case someone happens upon this.

Standard way to structure a gin server, or any web-server project for that matter, is as follows.

```cmd
src/
  .env
  go.mod
  go.sum
  main.go
  utils/
  static/
  routes/
  models/
  repositories/
  modules/
```

Something along these lines. All of this is pretty self explanatory. The focus of this article is how to deal with whatever goes into the `/modules` directory.

When it comes to individual modules or services, I personally like to use the singleton design pattern. For one thing I don't have to worry about memory overhead. It also encourages the idea that these modules are separate from the rest of the monolith, and if ever I wanted to detach services to achieve horizontal scaling, in theory it shouldn't be too hard to do so. Or maybe I'm just used to javascript. In any case, I like singletons, and I've yet to see a reason why I shouldn't.

```go

var moduleAInstance *moduleA
var once sync.Once

type moduleA struct {
  repository *repository.Repository // the repository interface
  // and various clients that moduleA requires access to
}

func GetInstance (repository *repository.Repository) {
  once.Do(func() {
    moduleAInstance = &moduleA{repository: repository}
  })
  return moduleAInstance
}

func (m *moduleA) HelloFeature(ctx context.Context) error {
  return fmt.Errorf("hello")
}
```

The tricky part is when we start considering the possibility of different features interacting with each other. Go enforces an acyclic dependency structure; an understandable restriction as this is one of the reasons why the go compiler is so fast. Also, I am told that loose coupling is never a bad thing.

Assume we have features A and B that need to invoke each other at some point. The obvious solution is to use forward declaration. Define interfaces for each module in maybe a sibling package.

```go
// src/modules/commons/moduleAInterface.go
package commons

type ModuleAInterface interface {
  HelloFeature(context.Context) error
}

// src/modules/commons/moduleBInterface.go
type ModuleBInterface interface {
  WorldFeature(context.Context) error
}

// src/modules/commons/interfaces.go
type ModuleInterfaces struct {
  ModuleA ModuleAInterface
  ModuleB ModuleBInterface
}
```

In this case, all modules should import all interface

```go
// src/modules/moduleA/moduleA.go
type moduleA struct {
  repository *repository.Repository
  modules *commons.ModuleInterfaces
}
```

Everything can be tied together in main as such.

```go
// src/main.go
modules := &commons.ModuleInterfaces {}

moduleA := moduleA.GetInstance(repository, modules)
moduleB := moduleB.GetInstance(repository, modules)
```

If we don't want all modules to have access to each other, we can simply specify the accessible modules per module.

```go
type moduleA struct {
  repository *repository.Repository
  //
  moduleB commons.ModuleBInterface
  moduleC commons.ModuleCInterface
}
```

But what if we don't want the modules to directly invoke each other at all. Maybe we want to simulate a distributed system as much as possible by implementing an event based interaction.

Arguably this adds an unncessary layer of complexity. It will help when you want a service to handle invokations sequentially.

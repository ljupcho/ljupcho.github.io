---
layout: post
title: "Go Generics"
date: 2023-05-24 16:08:10 +0200
tags: golang, performance
comments: true
---

The main idea about the generics is type safety so you won't be getting runtime errors that might happen in case of not using them. Performance wise they have not improved and my testing will show that there is no difference. I will do the tests on golang 1.20 in two cases, once I will use type to interface and second scenario will be the same functionality using generics.

I wanted to check what is the performance using generics in scenarios where I have two models, product and shop and I have a service that should upload an image for both. The functionality is the same and the methods would insert and fetch images for the model. I would usually create a New function that will return me a service that will have the model and possible other objects as well.

The logic using interface.

```go
type (
    MediaFiles interface {
		GetID() uint
	}

    media struct {
    	model    models.MediaFiles
    	repo     repositories.MediaRepository
    }

)

// Build service to handle images for a model.
func NewMedia(gmodel models.MediaFiles) Media {
	return &media{
		model:    gmodel,
		repo:     repositories.GetMediaRepo(),
	}
}

// GetAllImages fetches all images for a model and encodes the images.
func (m *media) GetAllImages() (*[]ResponseImage, error) {
	// Skip, not relevent to this blog post.
}

// Get service for media.
svc := media.NewMedia(product)
// Or for the other model.
svc := media.NewMedia(shop)
// Call the method to fetch images.
output, err = svc.GetAllImages()
```

<br/>
The same logic written with generics, I have:

```go
type (
    MediaModels interface {
        models.Product | models.Shop
        GetID() uint
    }

    media[T MediaModels] struct {
    	model    T
    	repo     repositories.MediaRepository
    }

)

// Build service to handle images for a model.
func NewMedia[T MediaModels](model T) Media {
	return &media[T]{
		model:    model,
		repo:     repositories.GetMediaRepo(model),
	}
}

// GetAllImages fetches all images for a model and encodes the images.
func (m *media[T]) GetAllImages() (*[]ResponseImage, error) {
	// Skip, not relevent to this blog post.
}

// Get service for media.
svc := media.NewMedia[models.Product](product)
// Or for the other model.
svc := media.NewMedia[models.Shop](shop)
// Call the method to fetch images.
output, err = svc.GetAllImages()
```

<br/>
The results. I run this command first for both cases:

```
ab -n 1000 -c 100 /api/images/product/46a5dff0-d639-4c9d-b621-03e1af666537

with generics:
Time taken for tests:   4.871 seconds
Requests per second:    205.29 [#/sec] (mean)

with interface:
Time taken for tests:   4.722 seconds
Requests per second:    211.79 [#/sec] (mean)
```

<br/>
I run the benchmark from go:

```
with generics:
root@c64d1b2c6088:/go/src/app/tests# go test -bench=.
goos: linux
goarch: amd64
pkg: app/tests
cpu: VirtualApple @ 2.50GHz
BenchmarkInterface-4   	     240	   5097498 ns/op
PASS
ok  	app/tests	1.515s

with interface:
root@c64d1b2c6088:/go/src/app/tests# go test -bench=.
goos: linux
goarch: amd64
pkg: app/tests
cpu: VirtualApple @ 2.50GHz
BenchmarkInterface-4   	     240	   5045698 ns/op
PASS
ok  	app/tests	1.521s
```

<br/>
I run the tests a couple of times, sometimes the generics are slightly better, sometimes the interface, so these times are averages of my test runs.<br/>
Conclusion from the tests:<br/>
Not a clear winner in terms of performance. Generics were introduced in go1.18 and were worse than using an interface, but at least now in go 1.20 that is fixed and hopefully can run even faster in newer versions.

Some thoughts about how I feel writing the code either with interfaces or generics.

- you can use an interface that specific structs will implement and type hint it into the service function. The difference is that this can accept nil, but does it really matter? I will probably query a model and err handle it if it is nil or not before passing it to the service function. So not really a strong argument in favor of generics.

- elimination of the interface{} and casting it to a specific type in a switch of if statements. so generics make the code look good in this case.

- one thing that generics improved and makes sense to use them is improving reusability. For example, if I have a repository service and use orm for db interactions, it seems a lot of duplication, boilerplate code if I have to do the CRUD methods for each repository service for each model. With generics I can just create one repository service that will handle each model's crud actions.

- you would need to have at least one method for the interface so you can type hint it in the function, which is not the case with generics you can also have the interface without methods types only.

- generics forces you to think about what is happening on compile vs run time. Obviously we don't want to have runtime errors or panics so the more stuff we figure out and make concrete on compile time the better.

- slice, map, reduce, filter will be using generics from golang 1.21, so it seems that might be even improved upon and we'll see if they bring some performance benefit in the future.

So, writing the code I would prefer using generics and now that the performance is fixed and hopefully stays that way I am inclined to use them, otherwise I would not accept code syntax win over performance win.

---
layout: post
title: "Path to golang microservices"
date: 2023-09-13 18:27:10 +0200
tags: golang, kubernetes, docker, performance
comments: true
---

I have one application that has a lot of things like modules, internal packages/libs, migrations, cli commands, controllers/handlers, services, repositories, utils functions etc. I wanted to extract some of those into services that will make more sense so they will be handled in a better way both in terms of working with them and deployments. I am still not sold on the microservices bandwagon, but I for one know that carrying on with a monolith is not the way. Kubernetes offers a lot of flexibility for building service, more than I need really, all I want is to have a couple of services split in a way that are independent regardless of data share so k8s will load balance between multiple instances/pods. Regarding the data of-course most companies deviate towards having a managed service from aws, google cloud or elsewhere will proper backups, data-warehouse solutions etc. So, I began splitting some of the folders/code from my monolith into modules and making services from some of them. I would have entities, libs, consumers, migrations, seeds and app (what's left of the monolith) as modules and will make services out of them except for the entities and libs. The reason for that is because I want to share the code as much as possible, but I'd rather extract them as modules or even duplicate some of the functions instead of running grps calls between the services. Out of these services migrations and seeds will be tasks instead of pods, so once they are called and run they will free up space and memory.

Once I have finished with refactoring the service, I push the image to the github registry. I will be using minikube to run the k8s cluster on my local machine.

```
docker build -t v01-api:latest . -f ./docker/api/Dockerfile
docker login ghcr.io -u ljupcho -p {github-token}
docker tag v01-api:latest ghcr.io/ljupcho/v01-api:latest
docker push ghcr.io/ljupcho/v01-api:latest
# Create a secret for k8s to pull images from the github registry.
kubectl create secret docker-registry gh-regcred --docker-server=ghcr.io --docker-username=ljupcho --docker-password={github-token}
kustomize build overlays/dev > customized-manifest.yaml
kubectl apply -f customized-manifest.yaml
kubectl get all
# List of the services built on the cluster.
minikube service list
# Run the api service.
minikube service api
minikube addons enable ingress
# Logs from the ingress-nginx controller that sits in from the 2 api pods.
kubectl logs pod/ingress-nginx-controller-7799c6795f-hb874 -n ingress-nginx
```

First, I wanted to have migrations as a task in kubernetes and in order to do that I need to decouple the migrations part from my monolith and make it a `kind: Job`, then build an image out of it, push it to registry and make a task in k8s, then run the `kustomize build overlays/prod > customized-manifest.yaml` that will create the pod. I kind of took the rust approach, as I wanted to have modules/entities, basically the tables into a separate service/folder, have another for migrations folder and then have an src folder that will have all the code for the service. At this moment I will move the migrate and models folder outside and make them with their own go.mod file and leave all the rest logic under the src folder with its own go.mod file.

I have put the api as a service and it generated a pod, but the thing is that as I had it as a monolith I needed to have everything installed and have a huge docker image. So now, the docker image for api is just compiling it and having it exposed on a port, no need to have anything installed inside that container and run commands in it, but have created other services/pods. You create a new pod/task/job/cronjob, that's how it goes. I don't really need to move the migration folder outside and I can keep the existing monolith codebase as is, I can just run and build services. For example, for the migrations I can run the go build command pointing to the migrate.go file with all the code first copied in the Dockerfile. Basically it will be the same as the api build pointing to another folder all the rest is the same. But the image in size is totally different, as to run the migrations all I need to import is gorm library and the modules. So, if I keep it as a monolith the image size shows something like 78MB and if I move the folder outside and import only what I need to the migrations then the image size is 23MB. Probably not a big deal right, but might make a big difference in lots of services and lots of tasks that need to be run and you probably need to import just what you need anyway.

So, we will break it down into different folder structures. Besides having modules extracted that the services will import there can be some of those module as libs, for example if you need a user repository to fetch the user by id most likely you will need that function into different services, so duplicating the code is kinda messy and I don't want to do it, rather I have created a crud repository that works with Generics, put it under libs modules and use it in different services. The crud with generics will allow me to use the same create, update, list, fetch methods for any module as they are all Gorm modules, so I don't need different repositories for each, except in special queries. Someone might actually run grpc calls to the main app service to fetch the user by id, instead I have two services, consumers and app, that will use the crud repository from the libs module and both of them connected to the same database. Some of the utils functions that will most likely never change I will duplicate in the service instead of running grpc calls again.

The Dockerfile for the services work in the same way, they would first compile and build a binary executable and then use alpine image to just copy the binary and expose it on a port.
I have created consumers, frontend and what remains from the main app as services, and have migrations, seeds as tasks that will be run once when triggered.
One of the reasons for moving into or creating a service is the database design. I would prefer to have the FKs in the database between entities, rather than let's say have a products service and a user services and have the user_id in the products table without constraint. In most cases this might lead to inconsistency in the database, especially as there might be a cron job on top and moving that data into another database or reporting tools and showing some bad data. I can say I do not prefer small services in a sense that they will require a lot more grpc calls than their own calls/calculations.

Docker images before the refactor:

```
➜  ~ docker images
REPOSITORY                    TAG                       IMAGE ID       CREATED          SIZE
v01-monolith-fe               latest                    5c409beafa8c   25 minutes ago   872MB
v01-monolith-api              latest                    87207ce39772   27 minutes ago   749MB
v01-monolith-nginx            latest                    ea2ca47ad851   29 minutes ago   46.9MB
v01-monolith-redis            latest                    54d599255aa9   30 minutes ago   138MB
postgres                      latest                    69e765e8cdbe   4 days ago       412MB
rabbitmq                      3.8.3-management-alpine   e7503e14fb69   3 years ago      140MB
```

Docker images after the refactor:

```
➜  ~ docker images
REPOSITORY                       TAG                       IMAGE ID       CREATED          SIZE
ghcr.io/ljupcho/v01-api          latest                    dda9ba0bc640   42 minutes ago   34.2MB
ghcr.io/ljupcho/v01-fe           latest                    5a233f7ad654   25 hours ago     79.2MB
ghcr.io/ljupcho/v01-consumers    latest                    c71e328216a4   10 days ago      39.4MB
ghcr.io/ljupcho/v01-seeds        latest                    12fadc6fb992   12 days ago      29MB
ghcr.io/ljupcho/v01-migrations   latest                    edf581b1bfad   12 days ago      23.1MB
postgres                         latest                    69e765e8cdbe   4 days ago       412MB
alpine                           latest                    7e01a0d0a1dc   4 weeks ago      7.33MB
gcr.io/k8s-minikube/kicbase      v0.0.40                   c6cc01e60919   8 weeks ago      1.19GB
golang                           1.20.4-alpine3.18         98045bb148f1   4 months ago     254MB
rabbitmq                         3.8.3-management-alpine   e7503e14fb69   3 years ago      140MB
```

When splitting the images in order to have them smaller you end up using builders and just copy the executable to the new alpine image. That means you won't necessarily log in into the container and do stuff inside, but you would work normally in dev mode on your local machine/host so we are back to where we started before the docker era just pointing to the docker images/containers you don't really change. This should be especially easier for development for the frontend as having the watch loaded and coding inside the container can be very painful and have to reload/recompile after every change, rather than just install all you need locally.

The argument that we can use different technologies or different versions of same technologies in different services is not really a strong argument for microservices as we finish in most cases with the same technology and version even. However, i would still prefer to have a service or some services in some technology rather than the others, like if memory is a big concert i would make those services in rust and have the api still in golang. eventually you would say let's build everything in rust, but where you gonna find the rust developer to help you out, once it gains popularity we can switch the tech as well! The same argument might be made for golang as well even though it has gained in popularity recently to be fair.

I did a comparison of running the same endpoint when using docker vs using the kubernetes cluster I have built with minikube.

Using k8s and ingress-nginx:

```
wrk http://marketplaza.test/api/test/1
Running 10s test @ http://marketplaza.test/api/test/1
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     5.15ms    4.79ms  46.94ms   81.56%
    Req/Sec     1.18k   177.64     1.55k    73.50%
  23588 requests in 10.00s, 24.29MB read
Requests/sec:   2357.68
Transfer/sec:      2.43MB
```

Using docker with nginx image:

```
wrk http://marketplaza.test/api/test/1
Running 10s test @ http://marketplaza.test/api/test/1
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     5.73ms    4.82ms  35.61ms   74.38%
    Req/Sec     0.97k   135.22     1.24k    82.50%
  19226 requests in 10.00s, 15.18MB read
Requests/sec:   1921.74
Transfer/sec:      1.52MB
```

So, it looks like the k8s even shows some better performance which might be because I have 2 api pods, which is what i wanted or it just might be because of nginx configurations. Now, all I have to do is buy a k8s cluster on some cloud provider and buy 2 bigger servers and run the cluster deploy.

Some of the errors I was getting along the way:

Error from server (BadRequest): container "task-run-migrations" in pod "task-run-migrations-7zgf9" is waiting to start: image can't be pulled
-> fix it by adding the `imagePullSecrets` in the prod/args.yaml file where my image path is as well.

[error] failed to initialize database, got error failed to connect to `host=db user=shop database=shop`: hostname resolving error (lookup db on 10.96.0.10:53: no such host)
-> fix it by changing the service name of the database that has a `kind: ClusterIP` with name to `db` instead of `db-service` so it can be used in the db connector in the api service.

0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling..
-> this is because some of the names of the services and deployments when overriding did not match. As i am using kustomize I have a base folder that is common for every environment and than have dev, prod which have different configMap and different values for some of the tags (the replicas number for ex), so some of the sections' names need to match in order to override the base ones.

-> ➜ kustomize git:(main) ✗ kubectl logs api-shop-deployment-5585f688d4-xxkjp
panic: open .env: no such file or directory
-> basically have a configMap for the k8s solution and have a .env for the docker containers only.

Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: "/go/bin/consumers": permission denied: unknown
-> the reason for this was because my services in the main.go they were not defined with package main, but the folder name!

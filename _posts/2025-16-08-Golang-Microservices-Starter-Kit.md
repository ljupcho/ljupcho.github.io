---
layout: post
title: "Golang Microservices Starter Kit"
date: 2025-16-08 15:02:10 +0200
tags: golang, microservices, kubernetes
comments: true
---
Golang microservices starter kit is my idea on how to architecturalize services in order to find the optimal way to a scalable solution and at same time abstractions to allow code reusability and easier development. This would be something to get me started on the right foot and give me a way to break down services.
In microservices terminology we can distinguish three entities: api, worker and consumer. Sometimes I spend quite some time making a decision where some logic belongs and how to move the data. Those decisions lead up to a more robust system and more scalable solution and if not done in the right way might lead up to unnecessary processing and requests, services hard to work with and as the times goes on even harder to break down. Starter kit offers monitoring with prometheus and grafana. Deployment is done with helm charts.

In this starter kit as an example I have the business logic to sync data from strava and store that data in a database. These are the following repositories:
- github.com/ljupcho/stravasync-api
- github.com/ljupcho/stravasync-worker
- github.com/ljupcho/stravasync-consumer
- github.com/ljupcho/stravasync-deployment
- github.com/ljupcho/go-common-lib
- github.com/ljupcho/go-contracts-v2

# API story: How does it work
The api supports endpoints both http and grpc. When the user registers and decides to sync its data, he will need to approve the sync from strava. Once that is done a request from strava is sent to this api which will start the process. The means dispatch a job to the queue which the  consumer will pick up and start fetching data.

The GRPC server supports endpoints for the other services and returns the needed data. The contracts are proto generated files between the services.

The api is built with domain driver design in mind so each package has its service and repository layer supported by sql boiler ORM. The ORM generates entities from the migrations so it is type safe instead of handing those on runtime, so there is some more generated code as a trade off for performance.

Mock files are generated for the purpose of unit tests with mocking the lower layers.

# Consumer story: How does it work
The idea of the consumer is to pull data from the message broker and process the job. But, there are different ways to do it and all of them affect scalability. The main idea of golang microservices is that they are scalable so they can handle high loads of traffic seamlessly.

There are push and pull consumers, usually the push used for instantaneously receiving the message, like real-time scenarios, whereas the pull is more used for pulling more at once and processing a lot of data. Also the pull consumers scale better when we set up more consumers for a topic.

In this case we are building a pull consumer, as we are collecting a higher volume of data from external API and would like to process it in a more scalable fashion. I am also using NATs as a message broker. Ofcourse, there are options such as RabbitMQ, Kafka, Google PubSub commonly known, but NATs should give me the throughput of Kafka and the flexibility of RabbitMQ.

I have address mainly two areas which i usually found problematic with consumers:

1) We would like to have the message broker behave as a fan-out or load-balancer. When defining the consumer:
 - use the same durable name that are bound to the same stream, in this case it will load-balance the jobs among the consumers,
which will be pods under kubernetes.
 - use different durable name or the same topic, in this case it will fan-out and each of the consumers will get a copy of the message from NATs.

The construction of the durable name of the consumer will be generated based on the message, for which I have decided to use proto generated messages concatenated with the service name. So, if it is a different service with the same message, it gets a copy, if it is the same service with the same message only one consumer gets the message. Proto files are like contracts that both sides, the publishing and consuming side agree on, it simplifies things instead of hard-coding naming around different microservices.

2) How should the consumer behave when processing the data?
- the consumer can connect to the intended database and manage the data or
- the consumer can send grpc calls to the api which manages the data.

The down side of the consumer having a db connection and do the job on its own, is that we cannot accomplish a scalability in the right way, because while we can still have a concurrency while processing the data and have a configuration in kubernetes that we get additional pod if cpu/memory limit is reached, but in that case the new pod/consumer will just pick a new job from the queue and our current consumer is still over burden with load, so we have not really make a scalability, we have just created another instance of the consumer. While this still can work and the consumer can dispatch itself another load of jobs to the message broker and we can have another queue/topic defined or we can send grpc calls to the api. 

Sending grpc call to the api has some cost, but the idea here is that we send all these requests
to the api and the api will auto-scale so we have kind of achieved organic scalability. The plus side of this is that in that case only the api has a db connection and manages the data. The grpc calls are using proto generated files between the two services. I have decided to go with the grcp approach in my case, with defining the number of consumers that will pull jobs from the queue in a load-balancing manner and send grcp calls to the api which will have their own limits and predefined number of pods.

## Useful commands to interact with NATs broker
```
nats stream ls --server localhost:4222
nats stream info athlete
nats consumer info
```

# Worker story: How does it work
I have decided to use temporal to schedule jobs and set up workflows. This service's only functionality is to fetch the athletes and dispatch a job for each into the queue.

The way we are fetching the athletes is by chunks and passing the data into the publishing library. I have made an abstraction for the publishing in a way to just pass in a feeder function, which in this case would query for the data per page. The data is then passed into a go channel upon which we can spin out multiple goroutines to publish the data to NATs. This makes the publishing a lot quicker in case we have lots of data to publish and is especially important when moving the data between services.

In this case I am building the publisher object with 10 goroutines used for publishing the data.
You may use the Publish method as well straight forwardly. 

For fetching the data, I have chosen to go with grpc calls to the api and fetch the athletes instead of having the worker maintain a db connection. This way I make sure that I have scalability with sending the requests to the api instead of the worker doing the querying.

# Deployment story: How does it work
For building services under the kubernetes cluster I am using helm charts. 
The platform folder has the helm charts which are downloaded with slight modifications to meet my needs for this specific project.
The backend folder has the values.yaml for overriding and mapping the values in the helm chart's template. Each of the services has per environment overrides.

The overall architecture is run with minikube locally and runs some tests and main logic to verify.
External services such as servers: database/postgres, NATs, vault, prometheus, grafana, loki, tempo are all docker containers built locally and the configuration just uses the host IP address to access them.

No secrets or passwords are hardcoded, rather they are accessed by key using external-secrets helm chart.

## Creating the vault secrets
```
docker exec -it vault sh
export VAULT_TOKEN=myroot
vault kv put -address=http://localhost:8200 secret/strava-credentials STRAVA_CLIENT_ID= STRAVA_CLIENT_SECRET=
```

## Creating the k8s secrets
```
kubectl create secret generic vault-token \
  --from-literal=TOKEN=myroot \
  -n stravasync

kubectl create secret generic dev-stravasync-db \
  --from-literal=LOGIN=ljupcho \
  --from-literal=PASSWORD=ljupcho \
  --from-literal=DATABASE_NAME=stravasync \
  -n stravasync

kubectl create secret docker-registry dockerhub-secret \
  --docker-username= \
  --docker-password= \
  --docker-email= \
  -n stravasync

kubectl get secrets -n stravasync
```
The main apps helm charts has in deployment.yaml the env variables set for database and vault token, in overrides under values.yaml I would just enable or disable them.
As the services build and push the binaries to dockerhub in order for this deployment to fetch them dockerhub-secrets need to be provided. It is defined
in values.yaml by setting the `imagePullSecrets`.

## Start minikube
```
minikube start --addons=ingress --cpus=4 --cni=flannel --install-addons=true --kubernetes-version=stable --memory=6g --driver=docker
```

## Creating the namespaces
```
kubectl create namespace stravasync
kubectl create namespace platform
kubectl create namespace monitoring
```

## Setting up external-secrets
```
helm uninstall external-secrets -n platform

helm install external-secrets platform/helm-charts/external-secrets -f backend/npd/platform/external-secrets/values.yaml --namespace platform
helm upgrade --install external-secrets platform/helm-charts/external-secrets -f backend/npd/platform/external-secrets/environments/dev.yaml --namespace platform

kubectl get external-secrets -n stravasync
```
external-secrets is used mainly by the consumer for accessing the strava client_id and client_secret by putting them in the vault for that client.
When the user/customer accepts to sync his data, the webhook hits the stravasync-api and stores the strava-credentials in the vault.

## Setting up temporal
```
helm uninstall temporal-dev -n platform

helm install temporal-dev platform/helm-charts/temporal -f backend/npd/platform/temporal/values.yaml -f backend/npd/platform/temporal/environments/dev.yaml --namespace platform --create-namespace

kubectl get all -n platform
```
Temporal is built using a helm chart and only its data is being kept in postgres database hosted locally. It is overridden by values.yaml and per env overrides. Putting it under k8s cluster manages scaling better.

## Setting up k8s-monitoring (should be for all env)
```
helm uninstall k8s-monitoring -n monitoring

helm install k8s-monitoring platform/helm-charts/k8s-monitoring -f backend/npd/platform/k8s-monitoring/values.yaml -f backend/npd/platform/k8s-monitoring/environments/dev.yaml --namespace monitoring --create-namespace

kubectl get all -n monitoring
```
I have used `k8s-monitoring` helm chart and overrides the paths for the servers from localhost, so they keep the metrics, logs and traces. After setting this up
I am able to see the logs in grafana, filter them, as well as metrics and traces that come out of the box with open telemetry. 

## Setting up stravasync-api service
```
--configure verticalpoduatoscaler
https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/installation.md
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
git checkout origin/vpa-release-1.0
REGISTRY=registry.k8s.io/autoscaling TAG=1.0.0 ./hack/vpa-process-yamls.sh apply

helm install stravasync-api-dev platform/helm-charts/apps -f backend/npd/apps/stravasync-api/values.yaml -f backend/npd/apps/stravasync-api/environments/dev.yaml --namespace stravasync --create-namespace
```
For each of the services there is values.yaml that overrides the apps helm chart templates. All configuration I need for a service is in values.yaml.
The api has the db/postgres, vault, ingress enabled with the container ports. The api connects to the db host as env variable, and gets the db user and db name
from k8s secrets defined by the `dev-stravasync-db` key.

## Setting up the worker
```
helm install stravasync-worker-dev platform/helm-charts/apps -f backend/npd/apps/stravasync-worker/values.yaml -f backend/npd/apps/stravasync-worker/environments/dev.yaml --namespace stravasync --create-namespace
```
The worker has just the metrics port exposed, but access it the temporal via the k8s `temporal-dev-frontend.platform.svc.cluster.local:7233`.

## Setting up the consumer
```
helm install stravasync-consumer-dev platform/helm-charts/apps -f backend/npd/apps/stravasync-consumer/values.yaml -f backend/npd/apps/stravasync-consumer/environments/dev.yaml --namespace stravasync --create-namespace
```
The consumer by default will have 2 pods and the NATs message broker is configured to load-balance for the same consumer. By same consumer means the consumers that are coming from the same service and listen on the same subject. By testing it I can verify that only one of the consumers will pick up the job for processing.
As the worker and the consumer access the data via the api using grpc calls, the grcp url is `stravasync-api-dev.stravasync.svc.cluster.local:9233`.

## Workflows
I am using github actions for the microservices. They are triggered when a new tag is created, and the flow will run the tests, compile the binary and push it to dockerhub. The deployment files for the services include the section for the image and its tag with dockerhub credentials which are stored as k8s secrets.

## Debugging
```
kubectl get all -n stravasync
kubectl logs pod/stravasync-api-dev-76f59c9cbb-ngqf4 -n stravasync
kubectl describe pod stravasync-api-dev-549d94f4c7-p6sgm -n stravasync
-- for exposing the api directly
kubectl port-forward service/stravasync-api-dev 8080:8080 -n stravasync
```


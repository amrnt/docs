---
title: Kubernetes Deployment
keywords: kubernetes
tags: [kubernetes]
sidebar: home_sidebar
permalink: deploy-kubernetes.html
summary: 
---


This doc provides a guide to running micro on kubernetes.

## Getting Started

- [Install Micro](#install-micro)
- [Writing a Service](#writing-a-service)
- [Deploying a Service](#deploying-a-service)
- [Healthchecking Sidecar](#healthchecking-sidecar)
  - [Install healthchecker](#install-healthchecker)
  - [Run healtchecker](#run-healtchecker)
  - [K8s Deployment](#k8s-deployment)
- [K8s Load Balancing](#k8s-load-balancing)
  - [Usage](#usage)
  - [K8s Deployment](#k8s-deployment-1)
  - [K8s Service](#k8s-service)

## Install Micro


```
go get github.com/micro/kubernetes/cmd/micro
```

or

```
docker pull micro/micro:kubernetes
```

For go-micro

```
import "github.com/micro/kubernetes/go/micro"
```

## Writing a Service

Write a service as you would any other [go-micro](https://github.com/micro/go-micro) service.

```go
import (
	"github.com/micro/go-micro"
	k8s "github.com/micro/kubernetes/go/micro"
)

func main() {
	service := k8s.NewService(
		micro.Name("greeter")
	)
	service.Init()
	service.Run()
}
```

## Deploying a Service

Here's an example k8s deployment for a micro service

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: default
  name: greeter
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: greeter-srv
    spec:
      containers:
        - name: greeter
          command: [
		"/greeter-srv",
		"--server_address=0.0.0.0:8080",
		"--broker_address=0.0.0.0:10001"
	  ]
          image: microhq/greeter-srv:kubernetes
          imagePullPolicy: Always
          ports:
          - containerPort: 8080
            name: greeter-port
```

Deploy with kubectl

```
kubectl create -f greeter.yaml
```

## Healthchecking Sidecar

The healthchecking sidecar exposes `/health` as a http endpoint and calls the rpc endpoint `Debug.Health` on a service. 
Every go-micro service has a built in Debug.Health endpoint.

### Install healthchecker

```
go get github.com/micro/kubernetes/cmd/health
```

or

```
docker pull microhq/health:kubernetes
```

### Run healtchecker

Run e.g healthcheck greeter service with address localhost:9091

```
health --server_name=greeter --server_address=localhost:9091
```

Call the healthchecker on localhost:8080

```
curl http://localhost:8080/health
```

### K8s Deployment

Add to a kubernetes deployment

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: default
  name: greeter
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: greeter-srv
    spec:
      containers:
        - name: greeter
          command: [
		"/greeter-srv",
		"--server_address=0.0.0.0:8080",
		"--broker_address=0.0.0.0:10001"
	  ]
          image: microhq/greeter-srv:kubernetes
          imagePullPolicy: Always
          ports:
          - containerPort: 8080
            name: greeter-port
        - name: health
          command: [
		"/health",
                "--health_address=0.0.0.0:8081",
		"--server_name=greeter",
		"--server_address=0.0.0.0:8080"
	  ]
          image: microhq/health:kubernetes
          livenessProbe:
            httpGet:
              path: /health
              port: 8081
            initialDelaySeconds: 3
            periodSeconds: 3
```

## K8s Load Balancing

Micro includes client side load balancing by default but kubernetes also provides Service load balancing strategies. 
We can offload load balancing to k8s by using the [static selector](https://github.com/micro/go-plugins/tree/master/selector/static) 
and k8s services.

Rather than doing address resolution, the static selector returns the service name plus a fixed port e.g greeter returns greeter:8080. 
Read about the [static selector](https://github.com/micro/go-plugins/tree/master/selector/static).

### Usage

To use the static selector when running your service specify the flag or env var 

```
MICRO_SELECTOR=static ./service
```

or

```
./service --selector=static
```

### K8s Deployment

An example deployment

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: default
  name: greeter
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: greeter-srv
    spec:
      containers:
        - name: greeter
          command: [
		"/greeter-srv",
		"--selector=static",
		"--server_address=0.0.0.0:8080",
		"--broker_address=0.0.0.0:10001"
	  ]
          image: microhq/greeter-srv:kubernetes
          imagePullPolicy: Always
          ports:
          - containerPort: 8080
            name: greeter-port
```

### K8s Service

The static selector offloads load balancing to k8s services. So ensure you create a k8s Service for each micro service. 

Here's a sample service

```
apiVersion: v1
kind: Service
metadata:
  name: greeter
  labels:
    app: greeter
spec:
  ports:
  - port: 8080
    protocol: TCP
  selector:
    app: greeter
```

Deploy with kubectl

```
kubectl create -f service.yaml
```

Calling micro service "greeter" from your service will route to the k8s service greeter:8080.


{% include links.html %}

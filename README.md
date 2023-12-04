
![Ingress_Kubernetes](https://github.com/MdAhosanHabib/Ingress_LB_Kubernetes/assets/43145662/296644ba-d950-4bc5-ada6-e96b5d3ec926)

# Conatainer image push then Ingress & Load Balancing in Kubernetes with FastAPI
This repository provides a guide on setting up a FastAPI application in a Kubernetes cluster with Ingress and load balancing using HAProxy.

## Prerequisites
Before you begin, make sure you have the following:

1. A Kubernetes cluster up and running.
2. Docker installed on your local machine.

## FastAPI docker image create and push to docker hub
```bash
[root@master1 k8s]# cd app
[root@master1 app]# pwd
/k8s/app

[root@master1 app]# cat main.py
```

```bash
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}
@app.get("/ahosan")
def read_root():
    return {"Hello": "Ahosan"}

@app.get("/itcl")
def read_root():
    return {"Hello": "ITCL"}
```

```bash
[root@master1 app]# cat requirements.txt
fastapi
uvicorn

[root@master1 app]# cat Dockerfile
```

```bash
#Dockerfile
FROM python:3.8

WORKDIR /k8s/app

COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "80"]
```

```bash
[root@master1 app]# cd /k8s/app
[root@master1 app]# docker build -t ahosan/ahosantest1:FastAPIv1 .
[root@master1 app]# docker images
REPOSITORY           TAG         IMAGE ID       CREATED          SIZE
ahosan/ahosantest1   FastAPIv1   034eac64f59e   50 seconds ago   1.02GB
[root@master1 app]# docker login
Username: ahosan
Password:
[root@master1 app]# docker push ahosan/ahosantest1:FastAPIv1
```
<img width="960" alt="Docker_Hub" src="https://github.com/MdAhosanHabib/Ingress_LB_Kubernetes/assets/43145662/eba74b00-d048-4c76-a518-df713f1cfad2">


## Deployment, Service and Ingress Roles
```bash
[root@master1 app]# dnf update -y
[root@master1 app]# curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

[root@master1 app]# helm version
[root@master1 app]# helm repo add stable https://charts.helm.sh/stable
[root@master1 app]# helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
[root@master1 app]# helm repo update
[root@master1 app]# helm install ingress-nginx ingress-nginx/ingress-nginx

--yaml creation for K8s
[root@master1 app]# cd /k8s/app

[root@master1 app]# cat fastapi-deployment.yaml
```
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
      - name: fastapi
        image: ahosan/ahosantest1:FastAPIv1
        ports:
        - containerPort: 80
```
```bash
[root@master1 app]# kubectl apply -f fastapi-deployment.yaml

--"targetPort" is for fastapi-port, "port" is k8s cluster's using.
[root@master1 app]# vi fastapi-service2.yaml
```
```bash
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  selector:
    app: fastapi
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
```bash
[root@master1 app]# kubectl apply -f fastapi-service2.yaml

[root@master1 app]# vi nginx-ingress-nodeport.yaml
```
```bash
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-controller
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: http
      nodePort: 30080
    - name: https
      port: 443
      targetPort: https
      nodePort: 30443
  selector:
    app.kubernetes.io/name: ingress-nginx
```
```bash
[root@master1 app]# kubectl apply -f nginx-ingress-nodeport.yaml

[root@master1 app]# vi fastapi-ingress2.yaml
```
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fastapi-ingress
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /root
        pathType: Prefix
        backend:
          service:
            name: fastapi-service
            port:
              number: 80
      - path: /ahosan
        pathType: Prefix
        backend:
          service:
            name: fastapi-service
            port:
              number: 80
      - path: /itcl
        pathType: Prefix
        backend:
          service:
            name: fastapi-service
            port:
              number: 80
```
```bash
[root@master1 app]# kubectl apply -f fastapi-ingress2.yaml

[root@master1 app]# cat deployment-nginx.yaml
```
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
spec:
  replicas: 3  # Number of desired replicas (adjust as needed)
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest  # You can replace this with your test application image
        ports:
        - containerPort: 80  # The port on which the container listens
```
```bash
[root@master1 app]# cat service-nginx.yaml
```
```bash
apiVersion: v1
kind: Service
metadata:
  name: test-service
spec:
  selector:
    app: test-app  # Match the label used in the Deployment
  ports:
    - protocol: TCP
      port: 80       # Port exposed by the Service
  type: ClusterIP    # Type of Service (ClusterIP is the default)
```
```bash
[root@master1 app]# cat ingress-nginx.yaml
```
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-nginx-ingress
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-service
            port:
              number: 80
```
```bash
[root@master1 app]# kaubectl apply -f deployment-nginx.yaml
[root@master1 app]# kaubectl apply -f service-nginx.yaml
[root@master1 app]# kaubectl apply -f ingress-nginx.yaml
[root@master1 app]#
[root@master1 app]# pwd
/k8s/app
[root@master1 app]#
```

## External load banalancer with HA proxy
```bash
[root@master1 app]# dnf install haproxy -y
--add to file end
[root@master1 app]# cat /etc/haproxy/haproxy.cfg
```
```bash
###for Ahosan's FastAPI###
frontend http_front
  bind *:80
  stats uri /haproxy?stats
  default_backend http_back

backend http_back
  balance roundrobin
  server fastapi1 192.168.141.129:30080 check
  server fastapi2 192.168.141.130:30080 check
  server fastapi3 192.168.141.131:30080 check

[root@master1 app]# systemctl start haproxy
[root@master1 app]# systemctl enable haproxy
[root@master1 app]# systemctl status haproxy
```

<img width="522" alt="API_end_Point" src="https://github.com/MdAhosanHabib/Ingress_LB_Kubernetes/assets/43145662/d53312bb-b41a-4490-924e-6052c2ab8943">

Now access by the LoadBalancer IP
```bash
http://192.168.141.128:80/
http://192.168.141.128:80/ahosan
http://192.168.141.128:80/itcl
```

Thanks from Ahosan.

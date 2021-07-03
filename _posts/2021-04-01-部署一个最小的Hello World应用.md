---
title: 部署一个最小的hello World应用
layout: article
tags: k8s
mode: immersive
lang: zh-Hans
outhor: Boom Young
pageview: true
article_header:
  type: cover
  image:
    src: /screenshot.jpg
---

> 本文以一个Flask应用为例，讲解k8s部署步骤



先看一下项目结构

```shell
[root@master flask-app]# tree
.
├── app
│   └── app.py
├── deploy.yaml
├── Dockerfile
├── hello-kubernetes.yaml
└── ingress-kubernetes.yaml

1 directory, 5 files
```



### 一、编写一个最小的APP-hello world

#### 1.1 app.py

Flask的用法不在本文讲解的范围内，有兴趣的可以查阅[官方文档](https://flask.palletsprojects.com/en/1.1.x/)

```shell
[root@master flask-app]# mkdir app
[root@master flask-app]# more ./app/app.py 
#!/usr/bin/env python3
# coding: utf-8

from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World!'


if __name__ == '__main__':
    app.run('0.0.0.0', 5000, debug=True)
```

#### 1.2 测试代码
启动
```shell
[root@master flask-app]# python3 ./app/app.py 
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 225-207-156
```

验证
```shell
[root@master flask-app]# curl "http://0.0.0.0:5000/"
Hello World!
```


### 二、构建docker镜像

#### 2.1 编写Dockerfile

使用python3.7基础镜像，pip安装flask模块

```shell
[root@master flask-app]# more Dockerfile
FROM python:3.7

RUN pip install flask

COPY ./app /app

CMD ["python3.7","/app/app.py"]
```

#### 2.2构建镜像

```shell
[root@master flask-app]# docker build -t hello-world:v1 .
// 稍作等待后
[root@master flask-app]# docker images
REPOSITORY            TAG         IMAGE ID            CREATED             SIZE
hello-world           v1          196574233df1        5 hours ago         885MB
```

### 三、使用K8S部署引用

#### 3.1  hello-kubernetes.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hell-world
spec:
  type: ClusterIP
  ports:
  - port: 5000
    targetPort: 5000
  selector:
    app: hello-world
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: helloworld-kubernetes
        image: hello-world:v1
        imagePullPolicy: Never
        ports:
        - containerPort: 5000
```


hello-kubernetes.yaml 分为两个部分

1. 第一部分创建Service，使用ClusterIP模式
2. 第二部分创建Deployment， spec.replicas指定副本数为1，spec.template.spec.image指定镜像，imagePullPolicy设为Never表示只从本地拉去镜像


#### 3.2 部署

```shell
[root@master flask-app]# kubectl apply -f ./hello-kubernetes.yaml
[root@master flask-app]# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-7bb68bc94f-25g22   1/1     Running   0          4h46m
[root@master flask-app]# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
hell-world   ClusterIP   10.110.198.96   <none>        5000/TCP   4h55m
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    9h
```


#### 3.3 验证

```shell
[root@master flask-app]# curl "http://10.110.198.96:5000/"
Hello World!
```



### 四、Ingress

对外暴漏服务

使用[NGINX Ingress 控制器](https://www.nginx.com/products/nginx-ingress-controller/)

```shell
# kubectl apply -f https://raw.githubusercontent.com/StudyXX/google-containers/v1.16.4/install/ingress-nginx/ingress-nginx-controller.yaml
```

*未完待续。。。*
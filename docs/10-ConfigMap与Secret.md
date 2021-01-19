# ConfigMap与Secret



ConfigMap与Secret是Kubernetes两种常用配置方式，挂载配置文件到容器内。

与配置中心（程序运行过程获取配置，不同于我们组的配置中心）的对比，ConfigMap/Secret有如下特点：

||配置中心|k8s ConfigMap|
|--|----|----|
|优|1）本地开发环境的代码可与生产环境保持一致<br />2）配置信息可控制访问|1）无需引入客户端，不用改代码<br />2）技术无关，任何编程语言的程序均可使用<br />3）运行时不引入额外的依赖|
|缺|1）需要加入客户端，要改代码<br />2）程序运行依赖于配置中心<br /><br />|1）本地开发环境不能使用ConfigMap|

ConfigMap用于保存一般配置信息。

Secret用于保存敏感信息（证书、密码），会被base64编码，且Secret信息不会出现在log中。



### 1. ConfigMap

**ConfigMap最广的应用是挂载配置文件到Pod内**。借助ConfigMap，可以为不同环境设置不同的配置文件。

**创建ConfigMap**

编辑配置文件，名为``nginx.conf``：

```
#user       www www;  ## Default: nobody
worker_processes  5;  ## Default: 1
#error_log  /log/error.log;
#pid        /app/nginx.pid;
worker_rlimit_nofile 8192;

events {
  worker_connections  4096;  
}

http {
#  include       mime.types;
  default_type  application/octet-stream;
  log_format   main '$remote_addr - $remote_user [$time_local]  $status '
    '"$request" $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
  sendfile     on;
  tcp_nopush   on;
  server_names_hash_bucket_size 128; 

  gzip on;
	gzip_min_length 1k;
	gzip_buffers 4 16k;
	#gzip_http_version 1.0;
	gzip_comp_level 2;
	gzip_types text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
	gzip_vary off;
	gzip_disable "MSIE [1-6]\.";

  client_max_body_size 100m;
  upstream jenkins {
    server 172.30.10.138:8080;
  }

  server {
    listen       80;
#    access_log  /log/web.access.log  main;

    location / {
      proxy_set_header  X-Forwarded-For $remote_addr;
      proxy_pass http://jenkins;
    }
  }
}
```

创建ConfigMap：

```
# kubectl -n pg-allen create configmap nginx-conf --from-file=nginx.conf=nginx.conf
configmap/nginx-conf created
```



**查看ConfigMap**

```
# kubectl get cm nginx-conf -n pg-allen -o yaml
... ...
# kubectl describe cm nginx-conf -n pg-allen
... ...
```



**挂载ConfigMap**

修改``super-front``的yaml，将``nginx-conf``这个ConfigMap挂载到``/etc/nginx``下：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: super-front
    codename: k8s-playground
    project: k8s-playground
  name: super-front
  namespace: 【你的环境名】
spec:
  minReadySeconds: 30
  replicas: 1  
  selector:
    matchLabels:
      app: super-front
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: super-front
        project: k8s-playground
    spec:
      containers:
      - env:   
        - name: LANG
          value: en_US.UTF-8
        - name: LANGUAGE
          value: en_US:en
        - name: LC_ALL
          value: en_US.UTF-8
        image: nginx:latest
        imagePullPolicy: Always
        livenessProbe:   
          failureThreshold: 3
          initialDelaySeconds: 168   
          periodSeconds: 20   
          successThreshold: 1
          tcpSocket:
            port: 80
          timeoutSeconds: 1
        name: super-front
        readinessProbe:   
          failureThreshold: 3
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 11
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 10m    
            memory: 64Mi    
          requests:
            cpu: 10m    
            memory: 64Mi   
        volumeMounts:      # volume挂载声明
        - mountPath: /etc/nginx
          name: volume0
      dnsPolicy: Default
      restartPolicy: Always
      volumes:          # ConfigMap类型的volume声明
      - configMap:
          defaultMode: 420
          name: nginx-conf
        name: volume0
```



更新``super-front``配置：

```
# kubectl apply -f ./super-front.yaml
```

在Gears上启动``super-front``的K8s Service，访问暴露IP:端口。看看能否显示Jenkins界面？



### 2. Secret

Secret可以认为是一种Kubernetes特殊对待的ConfigMap：

* Secret常用于存储机密信息，因此Kubernetes会对Secret数据进行Base64编码。
* 为了避免Secret泄漏，Kubernetes不会在操作日志中记录Secret的值。

在我司，目前有两处地方用到Secret：

* 为Ingress提供TLS证书。
* 镜像仓库的认证信息（imagePullSecrets）。



**查看Secret**

镜像仓库的认证信息（imagePullSecrets）也是以Secret的形式保存。我们尝试获取这个Secret：

```
# kubectl get secret reg-key -n pg-allen -o yaml
apiVersion: v1
data:
  .dockerconfigjson: 【Base64编码信息】
kind: Secret
metadata:
  creationTimestamp: "2020-04-08T02:53:40Z"
  name: reg-key
  namespace: pg-allen
  resourceVersion: "354090167"
  selfLink: /api/v1/namespaces/pg-allen/secrets/reg-key
  uid: 26df7407-7944-11ea-b4d6-005056810481
type: kubernetes.io/dockerconfigjson
```

对``.dockerconfigjson``进行Base64解码，查看镜像仓库认证信息。

```
# echo '【Base64编码信息】' | base64 --decode
```



**使用Secret**

Secret用法与ConfigMap类似，这里只展示作为``imagePullSecrets``的用法。

然后，查看deploy信息：

```
# kubectl get deploy super-front -n pg-allen -o yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: super-front
  namespace: pg-allen
  resourceVersion: "356413730"
  selfLink: /apis/extensions/v1beta1/namespaces/pg-allen/deployments/super-front
  uid: 0c906dc6-7adb-11ea-b4d6-005056810481
spec:
  minReadySeconds: 30
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: super-front
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
... ...
    spec:
      containers:
      - env:
        - name: LANG
          value: en_US.UTF-8
        - name: LANGUAGE
          value: en_US:en
        - name: LC_ALL
          value: en_US.UTF-8
        image: nginx:latest
        imagePullPolicy: Always
... ...
      dnsPolicy: Default
      imagePullSecrets:    # 配置imagePullSecrets
      - name: reg-key
... ...
```


扩展阅读：

* Configure a Pod to Use a ConfigMap： https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/
* Secrets：https://kubernetes.io/docs/concepts/configuration/secret/



Next：[11-Schedule](11-Schedule.md)


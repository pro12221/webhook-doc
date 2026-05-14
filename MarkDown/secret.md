# ConfigMap
kubectl create configmap -h

kind: ConfigMap
apiVersion: v1
metadata:
  name: cm-demo
  namespace: default
data:
  data.1: hello
  data.2: world
  config: |
    property.1=value-1
    property.2=value-2
    property.3=value-3








你可以使用四种方式来使用 ConfigMap 配置 Pod 中的容器：
在容器命令和参数内
容器的环境变量
在只读卷里面添加一个文件，让应用来读取
编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap

## 环境变量形式
apiVersion: v1
kind: Pod
metadata:
  name: testcm1-pod
spec:
  containers:
    - name: testcm1
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: cm-demo3
              key: db.host
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: cm-demo3
              key: db.port
      envFrom:
        - configMapRef:
            name: cm-demo1

## 卷形式
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    configMap:
      name: myconfigmap


# Secret
Secret用来保存敏感信息，例如密码、OAuth 令牌和 ssh key 等等，将这些信息放在 Secret 中比放在 Pod 的定义中或者 Docker 镜像中要更加安全和灵活。

内置类型	用法
Opaque	用户定义的任意数据
kubernetes.io/service-account-token	服务账号令牌
kubernetes.io/dockercfg	~/.dockercfg 文件的序列化形式
kubernetes.io/dockerconfigjson	~/.docker/config.json 文件的序列化形式
kubernetes.io/basic-auth	用于基本身份认证的凭据
kubernetes.io/ssh-auth	用于 SSH 身份认证的凭据
kubernetes.io/tls	用于 TLS 客户端或者服务器端的数据
bootstrap.kubernetes.io/token	启动引导令牌数据

创建好 Secret对象后，有两种方式来使用它：
以环境变量的形式
apiVersion: v1
kind: Pod
metadata:
  name: secret1-pod
spec:
  containers:
  - name: secret1
    image: busybox
    command: [ "/bin/sh", "-c", "env" ]
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: password
以Volume的形式挂载
apiVersion: v1
kind: Pod
metadata:
  name: secret2-pod
spec:
  containers:
  - name: secret2
    image: busybox
    command: ["/bin/sh", "-c", "ls /etc/secrets"]
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
  volumes:
  - name: secrets
    secret:
     secretName: mysecret
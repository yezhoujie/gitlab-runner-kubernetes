# 一个跑在k8s里的gitlab-runner 部署文件例子

## 前提

安装了k8s 集群， 安装了gitLab

## 安装runner

- 在git上创建项目
- 点到 settings -> CI CD -> Runners -> expand -> install Runner on Kubernetes
- 点击add Kubernetes cluster -> tab add existing cluster
- 填入数据，其中认证那个 PEM 可以 在 k8s 结点上查看 ./kube/config 文件获取
- 跳转到页面，要求安装 helm tiller, 但是k8s中已经安装了，不然那先安装，结果，果然失败了，未知原因
```
Something went wrong while installing Helm Tiller

Can't start installation process
```
- 放弃，还是在k8s中直接装吧

## 在k8s中安装runner

官方地址：

```
https://docs.gitlab.com/runner/install/kubernetes.html
```

yaml文件

```
#该yaml创建 git runner 
---
#创建namespace
apiVersion: v1
kind: Namespace
metadata:
  name: gitlab
  labels:
    name: gitlab
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-runner
  namespace: gitlab
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec", "secrets"]
  verbs: ["get", "list", "watch", "create", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-runner
  namespace: gitlab
subjects:
  - kind: ServiceAccount
    name: gitlab-runner
    namespace: gitlab
roleRef:
  kind: Role
  name: gitlab-runner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-runner
  namespace: gitlab
---
#创建configMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitlab-runner
  namespace: gitlab
data:
  config.toml: |
    concurrent = 1

    [[runners]]
      name = "Kubernetes Runner"
      url = "http://10.66.0.18:8088/"
      token = "7da090751a123c7584fba78b94a603"
      executor = "kubernetes"
      [runners.kubernetes]
        namespace = "gitlab"
        image = "busybox" 
         helper_image = "gitlab/gitlab-runner-helper:x86_64-d954cadc"
        [[runners.kubernetes.volumes.host_path]]
          name = "m2"
          mount_path = "/root/.m2"
          read_only = false
          host_path = "/root/.m2"
        [[runners.kubernetes.volumes.host_path]]
           name = "docker"
           mount_path = "/var/run/docker.sock"
           host_path = "/var/run/docker.sock" 
        [[runners.kubernetes.volumes.host_path]]
           name = "cache"
           mount_path = "/root/cache"
           host_path = "/root/cache" 
---
#创建deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: gitlab-runner
  namespace: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      name: gitlab-runner
  template:
    metadata:
      labels:
        name: gitlab-runner
    spec:
      serviceAccount: gitlab-runner
      containers:
      - args:
        - run
        image: gitlab/gitlab-runner:latest
        imagePullPolicy: Always
        name: gitlab-runner
        volumeMounts:
        - mountPath: /etc/gitlab-runner
          name: config
        - mountPath: /etc/ssl/certs
          name: cacerts
          readOnly: true
        - mountPath: /root/.m2
          name: m2
      restartPolicy: Always
      volumes:
      - configMap:
          name: gitlab-runner
        name: config
      - hostPath:
          path: /usr/share/ca-certificates/mozilla
        name: cacerts
        
```

```
kubectl apply -f <yaml file>
```

- 执行后查看pod
```
kubecel get pod -n gitlab
```

- 查看pod日志，发现报错了，这个是正常的，因为上面configMap中的token,是要runner向gielab注册后，由gitlab生成的，现在都还没有注册，哪里来的token
- 进入pod容器重新注册
```
kubectl exec -it gitlab-runner-fbc875786-4tsqp  -n gitlab /bin/bash
```
```
gitlab-runner register
```
- 叫你输入gitlab地址: http://10.66.0.18:8088/
- 叫你输入token, 这个token是 http://10.66.0.18:8088/admin/runners 里面全局的token，或者项目里 CI/CD ->runner 里面的token
- 输入describe
- 输入tags,多个已逗号隔开(==注意，这里的tag不是随便加的，以后你的.gitlab-ci.yml文件里job下如果没有设置和runner匹配的tag,这个runner就不会执行==)
- 输入执行环境，选择kubenetes
- 显示注册成功

现在刷新gitlab runner 页面，会出现一个runner,但是看到是不正常的，因为上面==configMap== 的token是不对的

点击对应runner的 [Runner token] 连接，就能看到页面上有这个runner对应的token，这个token才是一开始yaml文件里==configMap==里==token==的正确值，把configMap里的token值换成这个，重新 apply,过个几秒钟，这个runner就正常了

至此为止，runner就在k8s上安装完成了

==注意==
gitlab runner 不是执行ci主体，注册的gitlab runner只是负责再起几个 runner concurrent pod 来执行 ci 任务，所以，要注意 configMap 里的 runner 配置选项

```
[[runners]]
      name = "Kubernetes Runner"
      url = "http://10.66.0.18:8088/"
      token = "7da090751a123c7584fba78b94a603"
      executor = "kubernetes"
      [runners.kubernetes]
        namespace = "gitlab"
        image = "busybox" 
        --这个玩意比较坑，他是负责从gitlab 下载 代码，和 stages 里生成的 artifacts的，这里的声明只是覆盖，不写，他还是会起一个默认的来执行的， 
        --如果你在base image 里 root 下有文件夹没有映射到宿主机，会被他删掉
        --[考代码的时候，请把这个注释删掉]
         helper_image = "gitlab/gitlab-runner-helper:x86_64-d954cadc" 
         --声明一个volumes使用宿主机上的文件夹，这里映射.m2文件，为了mvn package 的时候可以用国内的镜像
        [[runners.kubernetes.volumes.host_path]]
          name = "m2"
          mount_path = "/root/.m2"
          read_only = false
          host_path = "/root/.m2"
        -- 这里有个坑，因为我们的runner是在k8s 里面以pod 跑的，也就是说他是在docker 里的跑的
        -- 但是我们的pod里还要装一个docker，来跑docker build，docker 里要是装了docker 的，/var/run/docker.sock 这个文件是不会被创建的，容器里的docker是跑不起来的
        -- 所以我们要映射宿主机的这个文件，给容器里的docker用
        --[考代码的时候，请把这个注释删掉]
        [[runners.kubernetes.volumes.host_path]]
           name = "docker"
           mount_path = "/var/run/docker.sock"
           host_path = "/var/run/docker.sock" 
        -- 这个文件映射是因为，我们后面的ci 要分两个stage跑，每个stage的环境是隔离的，也就是说上个stage 打的jar 包，在下个stage中是看不到的，这里用了宿主机的文件来存文件，让每个stage共享， 也可以用 artifacts 来实现，后面ci.yml 文件里会讲
        --[考代码的时候，请把这个注释删掉]
        [[runners.kubernetes.volumes.host_path]]
           name = "cache"
           mount_path = "/root/cache"
           host_path = "/root/cache" 
```

## 可以优化的地方
现在gitrunner 是运行在k8s里，所以pod会在各个宿主机上漂移，我们上面一些runner的配置主要是通过共享宿主机上的文件，这里其实可以用k8s 的pv,和pvc 来做，避免要在宿主机上手动添加配置文件

## 参考
[baseCi镜像：https://github.com/yezhoujie/java-docker-helm-baseci](https://github.com/yezhoujie/java-docker-helm-baseci)
[java-ci-Demo:https://github.com/yezhoujie/hello-world-ci-final](https://github.com/yezhoujie/hello-world-ci-final)
# 更多优质内容请参考[任小牛的博客](https://albumylog.cpolar.cn/#/renxiaoniu)  
# 1. 简介
Pod运行的前置条件就是镜像，镜像在拉取时总是会报拉取失败的错误，这点在访问Docker Hub, k8s.io等国外网站时尤其明显。因此，设置代理是必要的。本节使用clash实现集群代理，Yacd作为dashboard显示流量信息和切换链路。
# 2. 部署Clash
clash在执行的时候需要两个文件，ountry.mmdb和config.yaml。Country.mmdb为全球IP库，可以实现各个国家的IP信息解析和地理定位，自动下载的这个文件clash是无法使用的；config.yaml文件则是配置信息，类似于token的存在，个人推荐[SoCloud](https://socloud.me/auth/register?code=ATUO)，价格实惠，链路稳定。
由于clash需要这两个文件才可以运行，因此需要把这它们挂载到容器。
clash的官方镜像为[dreamacro/clash](https://hub.docker.com/r/dreamacro/clash)，为了不影响官方镜像，同时能确保Country.mmdb能正确挂载，这里把Country.mmdb打包成一个镜像，然后采用initContainers的方式，把Country.mmdb拷贝到clash容器，Config.yaml则采用ConfigMap挂载到容器中。
:::warning
Country.mmdb的挂载和config.yaml的挂载不能在同一目录，在 Deployment 的容器规范中，如果你使用相同的 mountPath 来定义多个 VolumeMounts（卷挂载点），则后面的定义将会覆盖前面的定义。
Country.mmdb不能使用Secret，因为它的大小是5.4MB，而Secret的最大尺寸限制是1MB。
:::
## 2.1 构建Country mmdb镜像
Dockerfile如下：
```Dockerfile
FROM busybox:1.36
WORKDIR /web
COPY Country.mmdb /web
```
Dockerfile的作用就是把Country.mmdb打包为镜像，如果集群分为不同的架构，还需要使用docker buildx构建多平台的镜像。我已经提前打包好了x86和arm架构的镜像，并推送到了Docker hub：[Country mmdb image](https://hub.docker.com/repository/docker/sky94520/country-mmdb/general)，如果想构建其他平台的镜像，可以fork这个[github-clash](https://github.com/sky94520/clash)，并在Setting中添加Docker的账号密码到Repository secrets：
![image.png](/api/v1/file/download/blog/images/62/screenshot-2023-08-20T02:51:03.png)
![image.png](/api/v1/file/download/blog/images/62/screenshot-2023-08-20T02:51:32.png)
然后执行Github Action，它会构建镜像，然后推送到账号密码所对应的Docker hub。
![image.png](/api/v1/file/download/blog/images/62/screenshot-2023-08-20T03:35:54.png)
## 2.2 编写YAML
创建文件clash.yaml，先输入ConfigMap，它保存的就是合法的config.yaml：
```yaml
apiVersion: v1
data:
  config.yaml: |
    #---------------------------------------------------#
    ## 更新：2023-08-18 08:16:29
    ## 感谢：https://github.com/Hackl0us/SS-Rule-Snippet
    ## 链接：https://doata.net/link/gg9v6HHtVi2LdRC1?clash=1
    #---------------------------------------------------#
    please input you config.yaml
kind: ConfigMap
metadata:
  name: clash-config
  namespace: default
```
然后是Deployment，使用initContainers把Country.mmdb挂载到/data/Country.mmdb；把clash-config保存的config.yaml挂载到/data/config/config.yaml，接着输入command覆盖Dockerfile中的命令，更改下clash执行时配置文件的路径，确保clash能够找到它们：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clash
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: clash
  template:
    metadata:
      labels:
        app: clash
    spec:
      initContainers:
        - image: sky94520/country-mmdb:1
          name: cpmmdb
          command: ["cp", "-r", "/web/Country.mmdb", "/app"]
          volumeMounts:
            - mountPath: /app
              name: mmdb-volume
      containers:
        - name: clash
          image: dreamacro/clash:v1.18.0
          command: ["/clash", "-d", "/data", "-f", "/data/config/config.yaml"]
          ports:
            - containerPort: 7890
            - containerPort: 9090
          volumeMounts:
            - mountPath: /data
              name: mmdb-volume
            - mountPath: /data/config
              name: config
      volumes:
        - name: mmdb-volume
          emptyDir: {}
        - name: config
          configMap:
            name: clash-config
            items:
              - key: config.yaml
                path: config.yaml
```
最后使用类型NodePort的Service，把服务暴露出去, 其中端口7890是HTTP代理，9090是clash提供的restful API：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: clash-service
  namespace: default
spec:
  type: NodePort
  ports:
    - port: 7890
      targetPort: 7890
      protocol: TCP
      name: tcp
    - port: 9090
      targetPort: 9090
      protocol: TCP
      name: restful
  selector:
    app: clash
```
执行```kubectl apply -f clash.yaml```部署到集群，查看clash-service对外的端口号：
```bash
kubectl get service clash-service
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
clash-service   NodePort   10.43.33.102   <none>        7890:30270/TCP,9090:30234/TCP   17h
```
其中30270为HTTP代理端口，30234为clash restful API端口。
:::tip
NodePort类型的Service会在30000~32678中随机选择一个端口，一经确定之后，除非删除重建，否则不会改变。
:::
## 2.3 更新集群代理
[k3s官网](https://docs.k3s.io/zh/advanced#%E9%85%8D%E7%BD%AE-http-%E4%BB%A3%E7%90%86)提到，配置HTTP代理，需要修改的文件是
```/etc/systemd/system/k3s.service.env```和```/etc/systemd/system/k3s-agent.service.env```，本文仅仅对拉取镜像做代理。那么只需要在这两个文件添加变量```CONTAINERD_HTTP_PROXY```和```CONTAINERD_HTTPS_PROXY```
```bash
CONTAINERD_HTTP_PROXY="http://127.0.0.1:30270"
CONTAINERD_HTTPS_PROXY="http://127.0.0.1:30270"
```
:::tip
如果仅一个节点，那么可以选择127.0.0.1，否则务必输入局域网地址，确保代理生效。
:::
接着重启服务：
```bash
# 重启master节点的k3s
systemctl restart k3s.service
# 重启agent节点的k3s
systemctl restart k3s-agent.service
```
# 3. 部署Yacd
:::warning
dashboard的作用仅仅是为了能够显示代理所消耗的流量以及更换链路，如果没有这方面的需求，可以不部署。
:::
yacd只需要部署Deployment和Service：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clash-dashboard
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: clash-dashboard
  template:
    metadata:
      labels:
        app: clash-dashboard
    spec:
      containers:
        - name: clash-dashboard
          image: haishanh/yacd:v0.3.8
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: clash-dashboard-service
  namespace: default
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    app: clash-dashboard
```
查看下clash-dashboard-service的端口
```bash
root@g410:/var/lib/rancher/k3s/server/manifests# kubectl get service clash-dashboard-service
NAME                      TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
clash-dashboard-service   NodePort   10.43.7.246   <none>        80:32641/TCP   17h
```
然后在浏览器输入<集群IP地址>:32641：
![image.png](/api/v1/file/download/blog/images/62/screenshot-2023-08-20T03:21:50.png)
然后在输入框输入<集群的IP地址>:30234，即可进入详情。
:::warning
请不要使用ingress绑定域名，否则在添加代理时可能会失败
:::
**测试**
接下来测试下代理是否生效，创建test-pod.yaml
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: python:latest
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /data/SUCCESS && exit 0 || exit 1"   #创建一个SUCCESS文件后退出
  restartPolicy: "Never"
```
这里为了测试，特地选择了一个python image，存储在362MB：
![image.png](/api/v1/file/download/blog/images/62/screenshot-2023-08-20T03:25:11.png)
执行这个yaml文件，可以看到运行速度贼快：
![image.png](/api/v1/file/download/blog/images/62/screenshot-2023-08-20T03:25:55.png)
# 4. 参考链接
1. [Ubuntu Server利用Clash实现git代理](https://wuuconix.link/2021/08/14/clash-dashboard/)
2. [linux环境使用clash实现网络代理访问外网](https://www.codenong.com/cs110925767/)

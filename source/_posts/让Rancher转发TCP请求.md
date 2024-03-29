---
title: 让Rancher支持上的k8s支持TCP转发
date: 2024-03-29 09:14:07 +0800
categories: [运维]
tags: [运维]
author: 
  name: ShoJinto
  link: https://shojinto.github.io
---

# 让Rancher支持上的k8s支持TCP转发

k8s默认只支持http协议转发，最近公司有个项目是基于TCP协议的，为了上其在k8s上顺利的对外提供服务因此这里需要对k8s进行一些设置。

网上找到一些理论介绍：

- https://www.cnblogs.com/liugp/p/16972366.html

- https://lokie.wang/article/126
- https://github.com/rancher/rancher/issues/14744

由于我这里是采用rancher部署的k8s所以文章中提到的一些操作不太一样，但是理论知识都在上面了。具体操作如下：

- 查看ingress-nginx-controller是否开启了 `tcp-server`和`udp-server`

```bash
kubectl -n ingress-nginx edit daemonsets.apps nginx-ingress-controller
```

![image-20240329094149050](D:\Users\ShoJinto\Documents\GitHub\shojinto.github.io\source\images\image-20240329094149050.png)

如果没有上图标记的配置则需要自行加上！！好在rancher默认已经有这两个配置项，我们只需要确认一下即可。

```bash
        - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
        - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
```

- 定义configmap

```shell
# 生成yaml文件，由于我这里是用Deployment进行的项目管理，所有实际上这个tcp-services.yaml文件会被嵌入到Deployment.yml文件中去。
cat >tcp-services.yaml<<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  5195: "tfxing-test/tfxing-socket:5195"
  # 格式为<ingress-controller-svc-port>:"<namespace>/<service-name>:<port>"。
  # 配置的意思是tfxing-test命名空间下的5195端口映射到ingress-controller service的5195端口，即可通过ingress-controller的service ip加5195端口访问到实际的服务
EOF

# 查看集群中的内容
kubectl -n ingress-nginx edit cm tcp-services
```

![image-20240329094915224](D:\Users\ShoJinto\Documents\GitHub\shojinto.github.io\source\images\image-20240329094915224.png)

- 暴露端口

```shell
kubectl -n ingress-nginx edit daemonsets.apps nginx-ingress-controller
```

![image-20240329100339809](D:\Users\ShoJinto\Documents\GitHub\shojinto.github.io\source\images\image-20240329100339809.png)

- 设置流量转发规则

```shell
kubectl -n ingress-nginx edit service ingress-nginx-controller-admission
```

![image-20240329100637253](D:\Users\ShoJinto\Documents\GitHub\shojinto.github.io\source\images\image-20240329100637253.png)

经过以上操作等待`ingress-nginx-controller` 配置完成既可以通过集群内的任意宿主机IP地址就可以访问`5195`端口了！

上面的配置如果是纯K8S操作的话还是比较繁琐负载的，稍不注意可能会导致整个集群出问题。好在我们是用rancher对k8s进行管理，所以以上操作都可以在图像界面中完成。

---

接下来将以截图的形式进行展示：

- 确认是否开启tcp、udp支持

![image-20240329101201327](D:\Users\ShoJinto\Documents\GitHub\shojinto.github.io\source\images\image-20240329101201327.png)

- 配置端口暴露

![image-20240329102005557](D:\Users\ShoJinto\Documents\GitHub\shojinto.github.io\source\images\image-20240329102005557.png)

- 配置流量转发

![image-20240329101621080](D:\Users\ShoJinto\Documents\GitHub\shojinto.github.io\source\images\image-20240329101621080.png)

图像界面的操作也搞定！
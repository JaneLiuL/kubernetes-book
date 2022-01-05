## Overview

这篇文档主要是讲解如何在Kubernetes 集群中签证书。（赶工，还不是因为要交接）

Cert-manager:

Cert-manager是一个在Kubernetes 集群中提供证书的控制器。然后它将这些证书请求发送到`Let'sEncrypt` API服务器进行签名。这将生成一个Kubernetes `Secret`形式的签名证书。

External DNS

`ExternalDNS`运行在集群中，在AWS DNS中创建DNS记录。根据云提供商的不同例如AWS Route53。它可以监视`Ingress `资源以及带注释的LoadBalancer Service和 自动为相应的endpoint 创建DNS记录的入口控制器通过Layer 7路由，所以它是可能的在本地计算机上配置主机名并将其指向入口控制器服务，这是一个繁琐和复杂的的过程。ExternalDNS自动化此过程。

Nginx-ingress

运行在Kubernets集群内部的容器可以通过Pod to Service通信相互通信。Kubernetes中使用一个入口控制器，以一种简单的方式将集群内的服务公开给集群外的客户端。nginx-ingress是我们使用的入口控制器。每个入口控制器需要一个LoadBalancer服务，该服务将给它一个集群外的ip。流量通过负载均衡器流向集群内部的入口控制器实例，然后流量被第7层路由到正确的pod。



(这篇文档是以AWS 平台为主来讲解)



## 组件

| 名字                    | URL                                             | Download link                                            |
| ----------------------- | ----------------------------------------------- | -------------------------------------------------------- |
| Cert-manager            | https://github.com/jetstack/cert-manager        | https://github.com/jetstack/cert-manager/releases/v1.6.0 |
| cert-manager-helm-chart | https://github.com/jetstack/cert-manager        | https://github.com/jetstack/cert-manager/                |
| External-dns            | https://github.com/kubernetes-sigs/external-dns | https://github.com/kubernetes-sigs/external-dns          |
| Nginx-ingress           | https://github.com/nginxinc/kubernetes-ingress  | https://github.com/nginxinc/kubernetes-ingress           |



## 架构图

待补



## 部署





## 组件流程

1. External-dns 在AWS DNS zone创建DNS record
2. Cluster-issuer-route53 在ingress 资源里面告诉cert-manager去创建certificate
3. cert-manager  去Let's encrypt 创建一个签名的request 并且接受ACME的返回
4. cert-manager 在DNS 里面添加ACME token
5. let's Encrypt 验证token 
6. Lets encrypt 返回签名的证书certificate给cert-manager
7. cert-manager 保存cert跟key 在K8S secret里面(这个secret就是ingress tls那里指定的文件名)

## 用户排查流程

用户看到的排查图

![](./images/用户排查图.png)
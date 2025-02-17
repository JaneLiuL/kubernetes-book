# background
https://github.com/HopopOps/k8s-ldap-auth 不支持添加rootca
但是公司内部的ldap必须要rootca,不然报错，代码已改
这个文档是说明这个k8s-ldap-auth的一些工作原理

# 原理说明
Kubernetes 支持多种现成的身份验证方法，例如 X.509 客户端证书、static HTTP bearer tokens, 和 OpenID Connect. 
LDAP 身份验证意味着用户将能够使用其来自轻量级目录访问协议 (LDAP) 目录的现有凭据向 Kubernetes 集群进行身份验证：

<img width="704" alt="image" src="https://github.com/user-attachments/assets/1892814f-a99d-4071-a158-de8a2677d59c" />

Every request to the Kubernetes API passes through three stages in the API server:
 authentication, authorisation, and admission control

 <img width="803" alt="image" src="https://github.com/user-attachments/assets/1ec26fbe-3f68-4ed9-9e0f-2ec3ead20f93" />
<img width="732" alt="image" src="https://github.com/user-attachments/assets/11d86926-b1b3-4e7a-9a7c-ede57efa0cf0" />
每个身份验证插件都实现一种特定的身份验证方法。
传入请求按顺序呈现给每个身份验证插件。
如果任何身份验证插件可以成功验证请求中的凭据，则身份验证完成，请求将进入授权阶段（其他身份验证插件的执行将被短路）。
如果没有一个身份验证插件可以验证请求，则请求将被拒绝，并显示 401 Unauthorized HTTP 状态代码。

# 搭建webhook-token 说明
因为这篇文档是针对我司，所以忽略了ldap server的搭建， 下图说明我们这个webhook 插件的工作原理
<img width="859" alt="image" src="https://github.com/user-attachments/assets/5494643d-3126-458a-b71c-91eeb8df27c9" />

Webhook Token 插件要求请求包含 HTTP bearer token 作为身份验证凭据。

HTTP 承载令牌作为 Authorization: Bearer <TOKEN> 包含在请求的 Authorization 标头中。
When the Webhook Token authentication plugin receives a request, it extracts the HTTP bearer token and submits it to an external webhook token authentication service for verification.

The Webhook Token authentication plugin invokes the webhook token authentication service with an HTTP POST request carrying a TokenReview object in JSON format that includes the token to verify.

TokenReview的格式如下
```
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "spec": {
    "token": "<TOKEN>"
  }
}
```

# solution

<img width="897" alt="image" src="https://github.com/user-attachments/assets/e1163a56-182a-4f3e-baca-f357fad76728" />

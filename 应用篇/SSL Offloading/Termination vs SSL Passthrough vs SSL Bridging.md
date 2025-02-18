# 概念
SSL Bridging：负载均衡器/代理解密传入的 HTTPS 流量，然后重新加密，再将其转发到后端服务器。
SSL Offloading（也称为 SSL Termination）：负载均衡器/代理解密传入的 HTTPS 流量，并将其不加密地发送到后端服务器。
SSL Passthrough：负载均衡器/代理不解密传入的 HTTPS 流量，而是将其原封不动地转发到后端服务器。


# SSL Termination (offloading)

<img width="1061" alt="image" src="https://github.com/user-attachments/assets/8ed29af7-febf-4701-832d-040e0a4e5a47" />
SSL Offloading（也称为 SSL Termination）允许用户借助负载均衡器前端的 SSL 证书与负载均衡器建立安全连接。
负载均衡器会解密传入的 HTTPS 流量。因此，在此阶段可以对流量应用第 7 层操作。
与 SSL 桥接不同，流量在从负载均衡器到后端服务器的途中不会重新加密。
经过卸载过程的流量会标有一个名为 X-Forwarded-Proto 的新header，该header会通知后端服务器客户端使用 HTTPS 联系负载均衡器。

# SSL Bridging
<img width="1043" alt="image" src="https://github.com/user-attachments/assets/2c8749a5-d185-4efc-9916-815305f3a73b" />

SSL Bridging 用户能够使用负载均衡器前端的 SSL 证书与负载均衡器建立安全加密的连接。
负载均衡器解密传入的 HTTPS 流量，从而允许其对收到的流量执行第 7 层操作。
随后，***负载均衡器的后端建立新的加密连接***，以重新加密负载均衡器和后端服务器之间的流量，这次使用后端服务器的证书。
例如，在微服务架构中，当需要解决诸如横切关注点之类的其他功能时，建议采用这种方法。

# SSL Passthrough
<img width="1000" alt="image" src="https://github.com/user-attachments/assets/f6d22906-ae29-41ea-a1f7-bc8d716b7624" />
Passthrough管理负载均衡器上加密流量的最直接方法。
正如其名称所示，此方法涉及通过负载均衡器路由流量，而无需在负载均衡器本身上进行解密。
虽然此选项可以显著降低开销，但它有局限性，因为无法执行任何第 7 层操作。
因此，基于 cookie-based sticky sessions等功能无法通过此方法实现。
此外，在应用程序不在服务器之间共享会话的情况下，用户可能会因重定向到组内的不同服务器而丢失会话。

SSL 直通可确保整个传输过程中的连接安全，解密只发生在目的地，从而最大限度地降低针对负载均衡器和服务器之间流量的中间人攻击风险。
此外，由于负载均衡器不会解密客户端和服务器之间的流量，因此它们的开销相对较低，从而可以更精确地引导流量。
然而，SSL 直通需要更多的中央处理单元 (CPU) 周期，从而导致更高的运营成本。
此外，它缺乏检查请求或对网络流量执行操作的能力，从而无法使用访问规则、重定向和基于 cookie 的粘性会话。
由于这些限制，SSL 直通最适合较小的部署。对于更大、更苛刻的使用要求，可能需要考虑替代方法

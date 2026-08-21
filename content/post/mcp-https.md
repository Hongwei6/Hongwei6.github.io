---
title: "将 MCP 改造为 HTTPS"
date: 2026-08-21T14:20:00+08:00
draft: false
description: "MCP Server 升级 HTTPS 的完整方案，包括证书生成、服务端配置、客户端适配（信任所有证书 / 生产环境 CA 校验）"
tags: ["Spring AI", "Java", "MCP", "HTTPS", "安全"]
categories: ["后端"]
cover: "/hero/mcp-https/03-client-success.png"
---

MCP 作为连接大模型与企业数据、业务系统的关键协议，其安全性至关重要。HTTP 属于明文传输协议，无论内网还是跨服务调用，都容易遭受中间人攻击（MITM）。

在金融、运营商、政府等对安全性要求严格的企业环境中，**HTTP 版 MCP Server 通常被判定为不合规**，必须升级为 HTTPS/TLS 加密传输。

## 改造 MCP Server

升级 HTTPS 流程与普通 Web 服务一致：准备证书、配置 TLS、通过反向代理暴露 HTTPS 端点。

### 生成自签名证书

```bash
# 如有重复生成，先删除旧证书
keytool -delete -alias local-ssl -keystore keystore.p12 -storepass 123456

# 生成 p12 服务器证书（包含公私钥）
keytool -genkeypair -alias local-ssl -keyalg RSA -keysize 2048 -storetype PKCS12 \
  -keystore keystore.p12 -validity 3650 -storepass 123456 -keypass 123456 \
  -dname "CN=localhost, OU=Dev, O=Demo, L=Local, ST=Local, C=CN" \
  -ext "SAN=IP:127.0.0.1,DNS:localhost" \
  -ext "BasicConstraints=ca:true"
```

将生成的 `keystore.p12` 放入项目 `resources` 目录：

![证书文件](/hero/mcp-https/01-keystore.png)

### 配置 HTTPS

```yaml
server:
  port: 8443
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: 123456
    key-store-type: PKCS12
    key-alias: local-ssl
    enabled: true
```

> **注意：** 生产环境应使用合法域名证书，并通过 Nginx 进行反向代理。

访问 8443 端口验证 HTTPS 是否生效：

![HTTPS 测试](/hero/mcp-https/02-https-test.png)

## 改造 MCP Client

### 方案一：信任所有证书（开发/测试环境）

适用于内网环境或合规性检查场景：

```java
public static McpSyncClient createInsecureHttpsClient(String baseUrl, String endpoint) {
    try {
        // 1. 创建信任所有证书的 TrustManager
        TrustManager[] trustAllCerts = new TrustManager[]{
                new X509TrustManager() {
                    public X509Certificate[] getAcceptedIssuers() {
                        return new X509Certificate[0];
                    }
                    public void checkClientTrusted(X509Certificate[] certs, String authType) {}
                    public void checkServerTrusted(X509Certificate[] certs, String authType) {}
                }
        };

        // 2. 初始化 SSL 上下文，绕过校验
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, trustAllCerts, new SecureRandom());

        // 3. 禁用主机名验证
        SSLParameters sslParameters = new SSLParameters();
        sslParameters.setEndpointIdentificationAlgorithm(null);

        // 4. 构建 httpClient
        HttpClient.Builder httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(30))
                .sslContext(sslContext)
                .sslParameters(sslParameters);

        HttpClientSseClientTransport transport = HttpClientSseClientTransport
                .builder(baseUrl)
                .sseEndpoint(endpoint)
                .clientBuilder(httpClient)
                .build();

        McpSyncClient mcp = McpClient.sync(transport).build();
        mcp.initialize();
        return mcp;
    } catch (Exception e) {
        throw new RuntimeException("创建 Insecure MCP Client 失败", e);
    }
}
```

### 方案二：CA 证书校验（生产环境）

```java
public static McpSyncClient createSecureHttpsClient(String baseUrl, String endpoint, String caCertPath) {
    try {
        // 1. 加载 CA 证书
        CertificateFactory cf = CertificateFactory.getInstance("X.509");
        FileInputStream fis = new FileInputStream(caCertPath);
        Certificate caCert = cf.generateCertificate(fis);
        fis.close();

        // 2. 创建 KeyStore 并导入 CA
        KeyStore ks = KeyStore.getInstance(KeyStore.getDefaultType());
        ks.load(null, null);
        ks.setCertificateEntry("caCert", caCert);

        // 3. 构建 TrustManagerFactory
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(
                TrustManagerFactory.getDefaultAlgorithm());
        tmf.init(ks);

        // 4. 创建 SSLContext
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, tmf.getTrustManagers(), new SecureRandom());

        // 5. 构建 httpClient
        HttpClient.Builder httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(30))
                .sslContext(sslContext);

        // 6. 构建 SSE Transport
        HttpClientSseClientTransport transport = HttpClientSseClientTransport
                .builder(baseUrl)
                .sseEndpoint(endpoint)
                .clientBuilder(httpClient)
                .build();

        // 7. 初始化 MCP Client
        McpSyncClient mcp = McpClient.sync(transport).build();
        mcp.initialize();
        return mcp;
    } catch (Exception e) {
        throw new RuntimeException("创建 Secure MCP Client 失败", e);
    }
}
```

### 导出公钥证书

从 p12 服务器证书导出 crt 公钥证书：

```bash
keytool -exportcert -alias local-ssl -keystore keystore.p12 \
  -storetype PKCS12 -storepass 123456 -rfc -file mcp-server.crt
```

验证连接成功：

![Client 连接成功](/hero/mcp-https/03-client-success.png)

### 本地测试注意事项

如果证书未配置 SAN，需要添加 JVM 参数忽略主机名校验：

```
-Djdk.internal.httpclient.disableHostnameVerification=true
```

![JVM 参数配置](/hero/mcp-https/04-jvm-args.png)

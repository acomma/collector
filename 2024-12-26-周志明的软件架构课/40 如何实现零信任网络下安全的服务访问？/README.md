你好，我是周志明。

在上节课“零信任网络安全”当中，我们探讨了与微服务运作特点相适应的零信任安全模型。今天这节课，我们会从实践和编码的角度出发，一起来了解在前微服务时代（[以 Spring Cloud 为例](https://github.com/fenixsoft/microservice_arch_springcloud)）和云原生时代（[以 Kubernetes with Istio 为例](https://github.com/fenixsoft/servicemesh_arch_istio)），零信任网络分别是如何实现安全传输、认证和授权的。

这里我要说明的是，由于这节课是面向实践的，必然会涉及到具体代码，为了便于讲解，在课程中我只贴出了少量的核心代码片段，所以我建议你在开始学习这节课之前，先去浏览一下这两个样例工程的代码，以便获得更好的学习效果。

## 建立信任

首先我们要知道，零信任网络里不存在默认的信任关系，一切服务调用、资源访问成功与否，都需要以调用者与提供者间已建立的信任关系为前提。

之前我们在[第 23 讲](https://time.geekbang.org/column/article/329954)也讨论过，真实世界里，能够达成信任的基本途径不外乎基于共同私密信息的信任和基于权威公证人的信任两种；而在网络世界里，因为客户端和服务端之间一般没有什么共同私密信息，所以真正能采用的就只能是基于权威公证人的信任，它有个标准的名字：[公开密钥基础设施](https://en.wikipedia.org/wiki/Public_key_infrastructure)（Public Key Infrastructure，PKI）。

这里你可以先记住一个要点，PKI 是构建[传输安全层](https://en.wikipedia.org/wiki/Transport_Layer_Security)（Transport Layer Security，TLS）的必要基础。

在任何网络设施都不可信任的假设前提下，无论是 DNS 服务器、代理服务器、负载均衡器还是路由器，传输路径上的每一个节点都有可能监听或者篡改通讯双方传输的信息。那么要保证通讯过程不受到中间人攻击的威胁，**唯一具备可行性的方案是启用 TLS 对传输通道本身进行加密，让发送者发出的内容只有接受者可以解密**。

建立 TLS 传输，说起来好像并不复杂，只要在部署服务器时预置好[CA 根证书](https://icyfenix.cn/architect-perspective/general-architecture/system-security/transport-security.html#%E6%95%B0%E5%AD%97%E8%AF%81%E4%B9%A6)，以后用该 CA 为部署的服务签发 TLS 证书就行了。

但落到实际操作上，这个事情就属于典型的“必须集中在基础设施中自动进行的安全策略实施点”，毕竟面对数量庞大且能够自动扩缩的服务节点，依赖运维人员手工去部署和轮换根证书，肯定是很难持续做好的。

而除了随服务节点动态扩缩而来的运维压力外，微服务中 **TLS 认证的频次**也很明显地高于传统的应用。比起公众互联网中主流单向的 TLS 认证，在零信任网络中，往往要启用[双向 TLS 认证](https://en.wikipedia.org/wiki/Mutual_authentication)（Mutual TLS Authentication，常简写为 mTLS），也就是不仅要确认服务端的身份，还需要确认调用者的身份。

* **单向 TLS 认证**：只需要服务端提供证书，客户端通过服务端证书验证服务器的身份，但服务器并不验证客户端的身份。**单向 TLS 用于公开的服务**，即任何客户端都被允许连接到服务进行访问，它保护的重点是客户端免遭冒牌服务器的欺骗。

* **双向 TLS 认证**：客户端、服务端双方都要提供证书，双方各自通过对方提供的证书来验证对方的身份。**双向 TLS 用于私密的服务**，即服务只允许特定身份的客户端访问，它除了保护客户端不连接到冒牌服务器外，也保护服务端不遭到非法用户的越权访问。

另外，对于前面提到的围绕 TLS 而展开的密钥生成、证书分发、[签名请求](https://en.wikipedia.org/wiki/Certificate_signing_request)（Certificate Signing Request，CSR）、更新轮换，等等，这其实是一套操作起来非常繁琐的流程，稍有疏忽就会产生安全漏洞。所以尽管它在理论上可行，但实践中如果没有自动化的基础设施的支持，仅靠应用程序和运维人员的努力，是很难成功实施零信任安全模型的。

那么接下来，我们就结合 Fenix's Bookstore 的代码，聚焦于“认证”和“授权”这两个最基本的安全需求，来看看它们在微服务架构下，有或者没有基础设施支持的时候，各自都是如何实现的。

我们先来看看认证。

## 认证

根据认证的目标对象，我们可以把认证分为两种类型，**一种是以机器作为认证对象**，即访问服务的流量来源是另外一个服务，这被叫做服务认证（Peer Authentication，直译过来是“节点认证”）；**另一种是以人类作为认证对象**，即访问服务的流量来自于最终用户，这被叫做请求认证（Request Authentication）。

当然，无论是哪一种认证，无论有没有基础设施的支持，它们都要有可行的方案来确定服务调用者的身份，只有建立起信任关系才能调用服务。

好，下面我们来了解下服务认证的相关实现机制。

### 服务认证

Istio 版本的 Fenix's Bookstore 采用了双向 TLS 认证，作为服务调用双方的身份认证手段。得益于 Istio 提供的基础设施的支持，我们不需要 Google Front End、Application Layer Transport Security 这些安全组件，也不需要部署 PKI 和 CA，甚至无需改动任何代码，就可以启用 mTLS 认证。

不过，Istio 毕竟是新生事物，如果你要在自己的生产系统中准备启用 mTLS，还是要先想一下，是否整个服务集群的全部节点都受 Istio 管理？如果每一个服务提供者、调用者都会受到 Istio 的管理，那 mTLS 就是最理想的认证方案。你只需要参考以下简单的 PeerAuthentication [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)配置，就可以对某个[Kubernetes 名称空间](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)范围内的所有流量启用 mTLS：

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: authentication-mtls
  namespace: bookstore-servicemesh
spec:
  mtls:
    mode: STRICT
```

不过，如果你的分布式系统还没有达到完全云原生的程度，其中还存在部分不受 Istio 管理（即未注入 Sidecar）的服务端或者客户端（这是很常见的），你也可以将 mTLS 传输声明为“**宽容模式**”（Permissive Mode）。

宽容模式的含义是受 Istio 管理的服务，会允许同时接受纯文本和 mTLS 两种流量。纯文本流量只用来和那些不受 Istio 管理的节点进行交互，你需要自行想办法解决纯文本流量的认证问题；而对于服务网格内部的流量，就可以使用 mTLS 认证。

这里你要知道的是，宽容模式为普通微服务向服务网格迁移提供了良好的灵活性，让运维人员能够逐个给服务进行 mTLS 升级。甚至在原本没有启用 mTLS 的服务中启用 mTLS 时，可以不中断现存已经建立的纯文本传输连接，完全不会被最终用户感知到。

这样，一旦所有服务都完成迁移，就可以把整个系统设置为严格 TLS 模式，即前面代码中的 mode: STRICT。

在 Spring Cloud 版本的 Fenix's Bookstore 里，因为没有基础设施的支持，一切认证工作就不得不在应用层面去实现。我选择的方案是借用[OAtuh 2.0 协议](https://time.geekbang.org/column/article/331411)的客户端模式来进行认证的，其大体思路有如下两步。

**第一步**，每一个要调用服务的客户端，都与认证服务器约定好一组只有自己知道的密钥（Client Secret），这个约定过程应该是由运维人员在线下自行完成，通过参数传给服务，而不是由开发人员在源码或配置文件中直接设定。我在演示工程的代码注释中，也专门强调了这点，以免你被示例代码中包含密钥的做法所误导。

这个密钥其实就是客户端的身份证明，客户端在调用服务时，会先使用该密钥向认证服务器申请到 JWT 令牌，然后通过令牌证明自己的身份，最后访问服务。

你可以看看下面给出的代码示例，它定义了五个客户端，其中四个是集群内部的微服务，均使用客户端模式，并且注明了授权范围是 SERVICE（授权范围在下面介绍授权中会用到），示例中的第一个是前端代码的微服务，它使用密码模式，授权范围是 BROWSER。

```java
/**
 * 客户端列表
 */
private static final List<Client> clients = Arrays.asList(
    new Client("bookstore_frontend", "bookstore_secret", new String[]{GrantType.PASSWORD, GrantType.REFRESH_TOKEN}, new String[]{Scope.BROWSER}),
    // 微服务一共有Security微服务、Account微服务、Warehouse微服务、Payment微服务四个客户端
    // 如果正式使用，这部分信息应该做成可以配置的，以便快速增加微服务的类型。clientSecret也不应该出现在源码中，应由外部配置传入
    new Client("account", "account_secret", new String[]{GrantType.CLIENT_CREDENTIALS}, new String[]{Scope.SERVICE}),
    new Client("warehouse", "warehouse_secret", new String[]{GrantType.CLIENT_CREDENTIALS}, new String[]{Scope.SERVICE}),
    new Client("payment", "payment_secret", new String[]{GrantType.CLIENT_CREDENTIALS}, new String[]{Scope.SERVICE}),
    new Client("security", "security_secret", new String[]{GrantType.CLIENT_CREDENTIALS}, new String[]{Scope.SERVICE})
);
```

**第二步**，每一个对外提供服务的服务端，都扮演着 OAuth 2.0 中的资源服务器的角色，它们都声明为要求提供客户端模式的凭证，如以下代码所示。客户端要调用受保护的服务，就必须先出示能证明调用者身份的 JWT 令牌，否则就会遭到拒绝。这个操作本质上是授权的过程，但它在授权过程中其实已经实现了服务的身份认证。

```java
public ClientCredentialsResourceDetails clientCredentialsResourceDetails() {
    return new ClientCredentialsResourceDetails();
}
```

而且，由于每一个微服务都同时具有服务端和客户端两种身份，它们既消费其他服务，也提供服务供别人消费，所以在每个微服务中，都应该要包含以上这些代码（放在公共 infrastructure 工程里）。

另外，Spring Security 提供的过滤器会自动拦截请求，驱动认证、授权检查的执行，以及申请和验证 JWT 令牌等操作，无论是在开发期对程序员，还是在运行期对用户，都能做到相对透明。

不过尽管如此，这样的做法仍然是一种应用层面的、不加密传输的解决方案。为什么呢？

前面我提到，在零信任网络中面对可能的中间人攻击，TLS 是唯一可行的办法。其实我的言下之意是，即使应用层的认证能在一定程度上，保护服务不被身份不明的客户端越权调用，但是如果内容在传输途中被监听、篡改，或者被攻击者拿到了 JWT 令牌之后，冒认调用者的身份去调用其他服务，应用层的认证就无法防御了。

所以简而言之，这种方案并不适用于零信任安全模型，只有在默认内网节点间具备信任关系的边界安全模型上，才能良好工作。

好，我们再来说说请求认证。

### 请求认证

对于来自最终用户的请求认证，Istio 版本的 Fenix's Bookstore 仍然能做到单纯依靠基础设施解决问题，整个认证过程不需要应用程序参与（JWT 令牌还是在应用中生成的，因为 Fenix's Bookstore 并没有使用独立的用户认证服务器，只有应用本身才拥有用户信息）。

当来自最终用户的请求进入服务网格时，Istio 会自动根据配置中的[JWKS](https://datatracker.ietf.org/doc/html/rfc7517)（JSON Web Key Set）来验证令牌的合法性，如果令牌没有被篡改过且在有效期内，就信任 Payload 中的用户身份，并从令牌的 Iss 字段中获得 Principal。关于 Iss、Principals 等概念，我在安全架构这个小章节中都介绍过了，你可以去回顾复习一下第 23 到 30 讲。而 JWKS 倒是之前从没有提到过，它代表了一个**密钥仓库**。

我们知道在分布式系统中，JWT 要采用非对称的签名算法（RSA SHA256、ECDSA SHA256 等，默认的 HMAC SHA256 属于对称加密），认证服务器使用私钥对 Payload 进行签名，资源服务器使用公钥对签名进行验证。

而常与 JWT 配合使用的 JWK（JSON Web Key）就是一种存储密钥的纯文本格式，在功能上，它和[JKS](https://en.wikipedia.org/wiki/Java_KeyStore)（Java Key Storage）、[P12](https://en.wikipedia.org/wiki/PKCS_12)（Predecessor of PKCS#12）、[PEM](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail)（Privacy Enhanced Mail）这些常见的密钥格式并没有什么本质上的差别。

所以顾名思义，JWKS 就是一组 JWK 的集合。支持 JWKS 的系统，能通过 JWT 令牌 Header 中的 KID（Key ID）自动匹配出应该使用哪个 JWK 来验证签名。

以下是 Istio 版本的 Fenix's Bookstore 中的用户认证配置。其中，jwks 字段配的就是 JWKS 全文（实际生产中并不推荐这样做，应该使用 jwkUri 来配置一个 JWKS 地址，以方便密钥轮换），根据这里配置的密钥信息，Istio 就能够验证请求中附带的 JWT 是否合法。

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: authentication-jwt-token
  namespace: bookstore-servicemesh
spec:
  jwtRules:
    - issuer: "icyfenix@gmail.com"
      # Envoy默认只认“Bearer”作为JWT前缀，之前其他地方用的都是小写，这里专门兼容一下
      fromHeaders:
        - name: Authorization
          prefix: "bearer "
      # 在rsa-key目录下放了用来生成这个JWKS的证书，最初是用java keytool生成的jks格式，一般转jwks都是用pkcs12或者pem格式，为方便使用也一起附带了
      jwks: |
        {
            "keys": [
                {
                    "e": "AQAB",
                    "kid": "bookstore-jwt-kid",
                    "kty": "RSA",
                    "n": "i-htQPOTvNMccJjOkCAzd3YlqBElURzkaeRLDoJYskyU59JdGO-p_q4JEH0DZOM2BbonGI4lIHFkiZLO4IBBZ5j2P7U6QYURt6-AyjS6RGw9v_wFdIRlyBI9D3EO7u8rCA4RktBLPavfEc5BwYX2Vb9wX6N63tV48cP1CoGU0GtIq9HTqbEQs5KVmme5n4XOuzxQ6B2AGaPBJgdq_K0ZWDkXiqPz6921X3oiNYPCQ22bvFxb4yFX8ZfbxeYc-1rN7PaUsK009qOx-qRenHpWgPVfagMbNYkm0TOHNOWXqukxE-soCDI_Nc--1khWCmQ9E2B82ap7IXsVBAnBIaV9WQ"
                }
            ]
        }
      forwardOriginalToken: true
```

而 Spring Cloud 版本的 Fenix's Bookstore 就要稍微麻烦一些，它依然是采用 JWT 令牌作为用户身份凭证的载体，认证过程依然在 Spring Security 的过滤器里中自动完成。不过因为这节课我们讨论的重点不在 Spring Security 的过滤器工作原理，所以它的详细过程就不展开了，只简单说说其主要路径：过滤器→令牌服务→令牌实现。

既然如此，Spring Security 已经做好了认证所需的绝大部分工作，那么真正要开发者去编写的代码就是令牌的具体实现，即代码中名为“RSA256PublicJWTAccessToken”的实现类。

它的作用是加载 Resource 目录下的公钥证书 public.cert（实在是怕“抄作业不改名字”的行为，我再一次强调不要将密码、密钥、证书这类敏感信息打包到程序中，示例代码只是为了演示，实际生产应该由运维人员管理密钥），验证请求中的 JWT 令牌是否合法。

```java
@Named
public class RSA256PublicJWTAccessToken extends JWTAccessToken {
    RSA256PublicJWTAccessToken(UserDetailsService userDetailsService) throws IOException {
        super(userDetailsService);
        Resource resource = new ClassPathResource("public.cert");
        String publicKey = new String(FileCopyUtils.copyToByteArray(resource.getInputStream()));
        setVerifierKey(publicKey);
    }
}
```

如果 JWT 令牌合法，Spring Security 的过滤器就会放行调用请求，并从令牌中提取出 Principals，放到自己的安全上下文中（即“SecurityContextHolder.getContext()”）。

在开发实际项目的时候，你可以根据需要自行决定 Principals 的具体形式，比如既可以像 Istio 中那样，直接从令牌中取出来，以字符串的形式原样存放，节省一些数据库或者缓存的查询开销；也可以统一做些额外的转换处理，以方便后续业务使用，比如将 Principals 转换为系统中的用户对象。

Fenix's Bookstore 的转换操作是在 JWT 令牌的父类 JWTAccessToken 中完成的。所以可见，尽管由应用自己来做请求验证，会有一定的代码量和侵入性，但同时自由度确实也会更高一些。

这里为了方便不同版本实现之间的对比，在 Istio 版本中，我保留了 Spring Security 自动从令牌转换 Principals 为用户对象的逻辑，因此就必须在 YAML 中包含 forwardOriginalToken: true 的配置，告诉 Istio 验证完 JWT 令牌后，不要丢弃掉请求中的 Authorization Header，而是要原样转发给后面的服务处理。

## 授权

那么，经过认证之后，合法的调用者就有了可信任的身份，此时就不再需要区分调用者到底是机器（服务）还是人类（最终用户）了，只需要根据其身份角色来进行权限访问控制就行，即我们常说的 RBAC。

不过为了更便于理解，Fenix's Bookstore 提供的示例代码仍然沿用此前的思路，分别针对来自“服务”和“用户”的流量来控制权限和访问范围。

举个具体例子。如果我们准备把一部分微服务看作是私有服务，限制它只接受来自集群内部其他服务的请求，把另外一部分微服务看作是公共服务，允许它可以接受来自集群外部的最终用户发出的请求；又或者，我们想要控制一部分服务只允许移动应用调用，另外一部分服务只允许浏览器调用。

那么，一种可行的方案就是**为不同的调用场景设立角色，进行授权控制**（另一种常用的方案是做 BFF 网关）。

我们还是以 Istio 和 Spring Cloud 版本的 Fenix's Bookstore 为例。

**在 Istio 版本的 Fenix's Bookstore 中**，通过以下文稿这里给出的配置，就限制了来自 bookstore-servicemesh 名空间的内部流量，只允许访问 accounts、products、pay 和 settlements 四个端点的 GET、POST、PUT、PATCH 方法，而对于来自 istio-system 名空间（Istio Ingress Gateway 所在的名空间）的外部流量就不作限制，直接放行。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: authorization-peer
  namespace: bookstore-servicemesh
spec:
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces: ["bookstore-servicemesh"]
      to:
        - operation:
            paths:
              - /restful/accounts/*
              - /restful/products*
              - /restful/pay/*
              - /restful/settlements*
            methods: ["GET","POST","PUT","PATCH"]
    - from:
        - source:
            namespaces: ["istio-system"]
```

但针对外部的请求（不来自 bookstore-servicemesh 名空间的流量），又进行了另外一层控制，如果请求中没有包含有效的登录信息，就限制不允许访问 accounts、pay 和 settlements 三个端点，如以下配置所示：

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: authorization-request
  namespace: bookstore-servicemesh
spec:
  action: DENY
  rules:
    - from:
        - source:
            notRequestPrincipals: ["*"]
            notNamespaces: ["bookstore-servicemesh"]
      to:
        - operation:
            paths:
              - /restful/accounts/*
              - /restful/pay/*
              - /restful/settlements*
```

由此可见，Istio 已经提供了比较完善的目标匹配工具，比如前面配置中用到的源 from、目标 to，以及没有用到的条件匹配 when，还有其他像是通配符、IP、端口、名空间、JWT 字段，等等。

当然了，要说灵活和功能强大，它肯定还是不可能跟在应用中由代码实现的授权相媲美，但对绝大多数场景来说已经够用了。在便捷性、安全性、无侵入、统一管理等方面，Istio 这种在基础设施上实现授权的方案，显然要更具优势。

**而在 Spring Cloud 版本的 Fenix's Bookstore 中**，授权控制自然还是使用 Spring Security、通过应用程序代码来实现的。

常见的 Spring Security 授权方法有两种。

第一种是使用它的 ExpressionUrlAuthorizationConfigurer，也就是类似下面编码所示的写法来进行集中配置。这个写法跟前面在 Istio 的 AuthorizationPolicy CRD 中的写法，在体验上是比较相似的，也是几乎所有 Spring Security 资料中都会介绍的最主流的方式，比较适合对批量端点进行控制，不过在 Fenix's Bookstore 的示例代码中并没有采用（没有什么特别理由，就是我的个人习惯而已）。

```java
http.authorizeRequests()
    .antMatchers("/restful/accounts/**").hasScope(Scope.BROWSER)
    .antMatchers("/restful/pay/**").hasScope(Scope.SERVICE)
```

第二种写法就是下面的示例代码中采用的方法了。它是通过 Spring 的[全局方法级安全](https://docs.spring.io/spring-security/site/docs/3.0.x/reference/el-access.html)（Global Method Security）以及[JSR 250](https://jcp.org/en/jsr/detail?id=250)的 @RolesAllowed 注解来做授权控制。

这种写法对代码的侵入性更强，需要以注解的形式分散写到每个服务甚至是每个方法中，但好处是能以更方便的形式做出更加精细的控制效果。比如，要控制服务中某个方法，只允许来自服务或者来自浏览器的调用，那直接在该方法上标注 @PreAuthorize 注解即可，而且它还支持[SpEL 表达式](https://docs.spring.io/spring-framework/docs/4.2.x/spring-framework-reference/html/expressions.html)来做条件。

表达式中用到的 SERVICE、BROWSER 代表的是授权范围，就是在声明客户端列表时传入的，具体你可以参考这节课开头声明客户端列表的代码清单。

```java
/**
 * 根据用户名称获取用户详情
 */
@GET
@Path("/{username}")
@Cacheable(key = "#username")
@PreAuthorize("#oauth2.hasAnyScope('SERVICE','BROWSER')")
public Account getUser(@PathParam("username") String username) {
    return service.findAccountByUsername(username);
}

/**
 * 创建新的用户
 */
@POST
@CacheEvict(key = "#user.username")
@PreAuthorize("#oauth2.hasAnyScope('BROWSER')")
public Response createUser(@Valid @UniqueAccount Account user) {
    return CommonResponse.op(() -> service.createAccount(user));
}
```

## 小结

这节课里，我们尝试以**程序代码和基础设施**两种方式，去实现功能类似的认证与授权，通过这两者的对比，探讨了在微服务架构下，应该如何把业界的安全技术标准引入并实际落地，实现零信任网络下安全的服务访问。

由此我们也得出了一个基本的结论：在以应用代码为主，去实现安全需求的微服务系统中，是很难真正落地零信任安全的，这不仅仅是由于安全需求所带来的庞大开发、管理（如密钥轮换）和建设（如 PKI、CA）的工作量，更是因为这种方式很难符合上节课所提到的零信任安全中“集中、共享的安全策略实施点”“自动化、标准化的变更管理”等基本特征。

但另一方面，我们也必须看到，现在以代码去解决微服务非功能性需求的方案是很主流的，像 Spring Cloud 这些方案，在未来的很长一段时间里，都会是信息系统重点考虑的微服务框架。因此，去学习、了解如何通过代码，尽最大可能地去保证服务之间的安全通讯，仍然非常有必要。

## 一课一思

有人说在未来，零信任安全模型很可能会取代边界安全模型，成为微服务间通讯的标准安全观念，你认为这个判断是否会实现呢？或者你是否觉得这只是存在于理论上的美好期望？

欢迎在留言区分享你的答案。如果你觉得有收获，欢迎你把今天的内容分享给更多的朋友。

好，感谢你的阅读，我们下一讲再见。
# gateway网关指定路由响应超时时间

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        responseTimeout: 10000
```

这个配置用于设置HttpClient的响应超时时间，单位是毫秒。具体来说，这个配置表示当Gateway向后端服务发出请求后，如果在10秒内没有收到后端服务的响应，就会触发超时处理。

这个设置是全局的，也就是说对Gateway的所有请求都生效，除非针对特定路由进行了覆盖或更改。

通过设置响应超时时间，可以控制Gateway与后端服务之间的通讯时间，确保系统能够及时处理超时情况，从而提高系统的可靠性和稳定性。

## 背景

由于有的查询或者导出接口，耗时非常长，可能超过上面的设置的响应超时时间（10S），可以改这个配置来延长，但是这个配置会影响到全部接口，所以不合适，此时需要针对某个路由进行配置

## 解决

>  gateway-router.json是一个包含路由配置信息的文件，通常用于配置 Spring Cloud Gateway 中的路由规则。在这个 JSON 文件中，你可以定义路由的规则、断言、过滤器以及转发地址等信息。这些信息描述了请求应该如何被路由到后端服务以及在路由过程中的各种处理操作。
>
> 路由规则通常由唯一的路由标识符 (id)、路由顺序 (order)、断言 (predicates)、过滤器 (filters) 以及目标地址 (uri) 组成。在 JSON 文件中，你可以按照特定的格式定义这些信息，然后加载到 Spring Cloud Gateway 中以实现这些路由规则。
>
> 总的来说，`gateway-router.json` 是用于存储路由配置信息的 JSON 文件，它允许你以结构化的方式定义和管理 Spring Cloud Gateway 的路由规则。

```json
[
  {
    "id": "a-service",
    "order": 1,
    "predicates": [
      {
        "name": "Path",
        "args": {
          "_genkey_0": "/a-service/**"
        }
      }
    ],
    "filters": [],
    "uri": "lb://a-service"
  },
  {
    //这个路由的唯一标识符是"a-service-exprot"
    "id": "a-service-exprot",
    //路由的顺序为-1，较低顺序的路由会优先匹配
    "order": -1,
    //这里定义了一个断言，根据请求的路径进行匹配，如果请求的路径是"/a-service/user/export"，就会匹配上这个路由
    "predicates": [
      {
        "name": "Path",
        "args": {
          "_genkey_0": "/a-service/user/export"
        }
      }
    ],
    //这个路由定义了一个名为 "StripPrefix" 的过滤器，它会移除请求路径中的第一个部分
    "filters": [{ "args": { "parts": 1 }, "name": "StripPrefix" }],
    //在元数据中设置了响应超时时间和连接超时时间为300000毫秒（即300秒）
    "metadata": {
      "response-timeout": 300000,
      "connect-timeout": 300000
    },
    //匹配上这个路由之后，请求会被转发到负载均衡的"a-service"服务
    "uri": "lb://a-service"
  }
]
```

a服务中`/user/export`接口，耗时非常长，我们就特别针对这个接口设置超时时间300000毫秒
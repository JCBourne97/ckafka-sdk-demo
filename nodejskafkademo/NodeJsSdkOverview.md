Node.js 客户端可以通过消息队列 CKafka 版提供的多种接入点接入并收发消息。

| 项目     | **默认接入点**         | **SASL接入点**                                               |
| :------- | ---------------------- | ------------------------------------------------------------ |
| 网络     | VPC                    | VPC                                                          |
| 协议     | PLAINTEXT              | SASL_PLAINTEXT                                               |
| 端口     | 9092                   | 请在CKafka控制台实例基本信息页面的【接入方式模块】获取SASL接入点信息。<br/>![](https://main.qcloudimg.com/raw/6855a9d500dcbefbabed91515b695050.png) |
| SASL机制 | 不适用                 | PLAIN：一种简单的用户名密码校验机制。 |
| Demo     | [PLAINTEXT]()         | [SASL_PLAINTEXT/PLAIN]()                                   |
| 文档     | 默认接入点收发消息        | SASL接入点PLAIN机制收发消息                                   |


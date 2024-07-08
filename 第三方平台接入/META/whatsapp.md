官方文档地址：https://developers.facebook.com/docs/whatsapp/cloud-api/overview/?translation

官方postman接口地址，请求返回内容都有，比如发消息，接收webhook事件（收消息）：https://www.postman.com/meta/workspace/whatsapp-business-platform/request/13382743-1b657873-9991-436e-91f2-b6583314acef

获取 临时/永久 口令 第三方视频教程（全局翻墙打开）：https://www.yijia-soft.com/CloudAPI.html

这个是获取口令官方文档：https://developers.facebook.com/blog/post/2022/12/05/auth-tokens/?locale=zh_CN

1. 使用云端api，本地api需要部署且弃用了
2. token在创建应用的时候会生成，有临时24小时的，也有永久的，测试的时候用临时的
3. sdk可以网上看看，有的JDK8不能用，要升级

webhook：一种用来实时接收和处理平台上发生的事件的机制。通过Webhook，服务器能够监听并自动响应来自平台的特定事件，如接收到的消息、状态更新等。



配置webhook地址文档：https://developers.facebook.com/docs/graph-api/webhooks/getting-started#create-endpoint

配置了这个地址后，以后比如收到用户消息，就会发给这个地址

但是配置这个地址前会验证这个地址是否是你的保证安全

1. 验证使用get请求，返回官方发给你的challenge参数即可
2. 收事件使用post，json接收参数

Webhooks 通知参数文档：https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks/components#messages-object

收到的消息文档都有写，拿到后做你需要的事情即可

**最后重要的事就是，想要收到消息应用必须上线，不然收不到**



上传下载媒体文件文档：https://developers.facebook.com/docs/whatsapp/cloud-api/reference/media/#download-media

发送消息请求可能需要，上传到它们的服务器才能发
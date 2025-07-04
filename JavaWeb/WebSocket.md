### 一、WebSocket介绍
* **WebSocket**是基于TCP的一种网络协议，实现了**浏览器和服务器只需完成一次握手就能创建持久性的连接，进行双向数据传输**。
### 二、Http和WebSocket的对比
* Http是短链接，WebSocket是长链接
* Http通信是单向的，基于请求响应模式，WebSocket通信是双向的，基于事件驱动模式
* 二者底层都是基于TCP协议
### 三、WebSocket使用场景
* 视频弹幕
* 聊天室
* 体育实况
* 股票基金报价实时更新
### 四、WebSocket使用步骤
* 1.使用webSocket.html页面作为WebSocket客户端
* 2.导入WebSocket的maven坐标
* 3.导入WebSocket的服务端组件WebSocketServer, 用于和客户端通信
* 4.导入WebSocket的配置类WebSocketConfig，注册WebSocket的服务端组件

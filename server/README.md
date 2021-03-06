服务端orm框架：使用[gorm](https://github.com/jinzhu/gorm)

# TaskTalk服务端

## 方案

* 采用websocket, uri=/client
* 协议使用[jsonrpc2.0](https://www.jsonrpc.org/specification)

### jsonrpc

#### 官方协议解释

rpc, 即远程过程调用, 也就是将远端的接口在异步全双工网络中实现类似调用本地接口一样的同步调用

例如本地函数:

```javascript
function sum(a, b)
{
	let c = a+b
	return c
}

sum(1, 2) => 3
```

用jsonrpc在websocket上表示这个function:

```json
{// 调用方
	"method":"sum",
	"id":1,
	"params":{
		"a": 1,
		"b": 2
	}
}

{// 被调用方
	"id":1,
	"result":{
		"c": 3
	}
}
```

* id, 这个是网络异步下的同步字, 由发送方维护, 一般是递增的数字,就是你发出去的消息, 你在阻塞等待时, 收到相同id的回复消息, 就匹配上结果了, rpc的核心就是这个id字段

| 消息类型       | method | id   | result | error                       |
| -------------- | ------ | ---- | ------ | --------------------------- |
| request(请求)  | Yes    | Yes  | No     | No                          |
| response(回复) | No     | Yes  | Maybe  | Maybe(不能和result一起出现) |
| notify(通知)   | Yes    | No   | No     | No                          |

例:

=> {"method":"sum","id":1, "params":{"a":1, "b":2}}

=> {"method":"sum","id":2, "params":{"a":3, "b":4}}

=> {"method":"sum","id":3, "params":{"a":"wefweff", "b":6}}

<= {"id":2, "result":{"c": 7}} // 3+4的结果

<= {"id":3, "error":{"code":100, "message":"params error"}} // 参数错误

<= {"id":1, "result":{"c": 3}} // 1+2的结果



## http协议

/login

* direction

  client->server

* desc

  此协议用于换取真正有状态的的websocket地址, 该ws地址, 一旦连上将产生登录信息

  此协议用于版本检查

* example

```json
POST http://xxx.xxx.xxx.xxx:xxx/login HTTP/1.1
{
	"id":"1111", // 工号
	"name":"张三",
	"version":"1.0.0"
}

// 登录成功
HTTP/1.1 200 OK
{
	"status": 0,
	"wsUrl":"ws://xxx.xxx.xxx.xxx:xxx/xxx?token=xxx"
}

// 登录成功, 该账号第一次登录
HTTP/1.1 200 OK
{
	"status": 1,
	"wsUrl":"ws://xxx.xxx.xxx.xxx:xxx/xxx?token=xxx"
}

// 版本不对
HTTP/1.1 200 OK
{
	"status": 2,
	"lastVersion": "1.0.1"
}

// 和上一次登录ip不同, 可以强行登录
HTTP/1.1 200 OK
{
	"status": 3,
	"wsUrl":"ws://xxx.xxx.xxx.xxx:xxx/xxx?token=xxx"
}
```





## websocket协议



### notify.auth.offline

* direction

  server->client

* desc

  通知客户端掉线

* example

  ```json
  {// 通知
  	"method":"notify.auth.offline",
  	"params":{
  		"reason": 0, // 1=被另一个人挤下线
  		"ip": "", // 另一个人的ip, reason=1时出现该字段
  	}
  }
  ```
  
  
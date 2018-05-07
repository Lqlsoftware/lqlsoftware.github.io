Packet: github.com/Lqlsoftware/gopcap
``` go
func Start(port layers.TCPPort)
描述: 启动服务器
	port: 		监听的TCP端口(默认80)
	
func Bind(Url string, method http.HttpMethod, handler func(*http.HttpRequest,*http.HttpResponse))
描述: 绑定处理函数到URL上 具有有该URL和请求方式的Request会调用绑定的处理函数
	Url:	 	URL
	method:		HTTP请求方式
	handler:	处理函数
	
func DeBind(Url string, method http.HttpMethod)
描述: 解除URL上的绑定
	Url:	 	URL
	method:		HTTP请求方式
```
Packet: github.com/Lqlsoftware/gopcap/http
``` go
func (req *HttpRequest)GetParam(key string) string
描述: 获取request的 GET/POST 方法的表单参数
	key:		参数名
	return:		参数值
	
func (req *HttpRequest)GetAllParamKey() []string
描述: 获取request的 GET/POST 方法的表单的所有参数名
	return:		所有参数名
	
func (rep *HttpResponse)Write(Data ...interface{})
描述: 向response中写入数据 即返回数据
	Data:		任意类型需要返回的数据
	
func (rep *HttpResponse)SetHeader(key string, value string) {
描述: 设置response的首部
	key:		header的参数名
	value:		header的参数值
```

返回

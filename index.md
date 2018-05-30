Packet: github.com/Lqlsoftware/gopcap
``` go
func Start(port layers.TCPPort)
描述: 启动服务器
	port: 		监听的TCP端口(默认80)
```
``` go	
func Bind(Url string, method http.HttpMethod, handler func(*http.HttpRequest,*http.HttpResponse))
描述: 绑定处理函数到URL上 具有有该URL和请求方式的Request会调用绑定的处理函数
	Url:	 	URL
	method:		HTTP请求方式
	handler:	处理函数
```
``` go	
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
```
``` go
func (req *HttpRequest)GetAllParamKey() []string
描述: 获取request的 GET/POST 方法的表单的所有参数名
	return:		所有参数名
```
``` go
func (rep *HttpResponse)Write(Data ...interface{})
描述: 向response中写入数据 即返回数据
	Data:		任意类型需要返回的数据
```
``` go	
func (rep *HttpResponse)SetHeader(key string, value string) {
描述: 设置response的首部
	key:		header的参数名
	value:		header的参数值
```
```go
package main

import (
	"fmt"
	"math"
	"sort"
)

// 样本
type Data struct {
	// 特征值
	x1		float64
	x2 		float64
	// class = 0 未分类
	class	int
}

// 样本距离 采用曼哈顿距离
func (data *Data)DistanceWith(data2 *Data) float64 {
	return math.Abs(data.x1 - data2.x1) + math.Abs(data.x2 - data2.x2)
}

// w1类
var (
	X1 = Data{1,0,1}
	X2 = Data{0,1,1}
	X3 = Data{0,-1,1}
)
// w2类
var (
	X4 = Data{0,0,2}
	X5 = Data{0,2,2}
	X6 = Data{0,-2,2}
	X7 = Data{-2,0,2}
)
// 训练数据集
var trainData = []Data{X1,X2,X3,X4,X5,X6,X7}

// 测试数据
var X = Data{0.2,0.3, 0}

// KNN分类
func KNNClassfiy(trainData []Data, target *Data, k int) {
	// 得到前k个最小的距离
	distance := getMinDistance(trainData, *target, k)

	// 计数器计数
	counter := make(map[int]int)
	for _,d := range distance {
		counter[d[1].(int)]++
	}

	// 测试数据分入计数最多的类
	for class, count := range counter {
		if count > counter[target.class] {
			target.class = class
		}
	}
}

// 计算前k个最小的距离
func getMinDistance(data []Data, target Data, k int) [][]interface{} {
	// 边界条件
	if k <= 0 {
		return [][]interface{}{}
	} else if k > len(data) {
		k = len(data)
	}

	// 计算所有样本与测试数据的距离
	distance := make([][]interface{}, len(data))
	for i,sample := range data {
		distance[i] = []interface{}{target.DistanceWith(&sample),sample.class}
	}

	// 递增排序
	sort.Slice(distance, func(i, j int) bool {
		return distance[i][0].(float64) < distance[j][0].(float64)
	})

	// 取前k个
	return distance[:k]
}

func main() {
	KNNClassfiy(trainData, &X, 6)
	fmt.Println(X.class)
}
```

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
	"os"
)

// Buffer大小定义
const InputBufferSize = 80

func main() {
	// input
	Buffer := NewBuffer()
	fmt.Println("Input a string (end with '#'):")
	Buffer.Read(os.Stdin)

	// process
	res, err := LexicalAnalysis(Buffer)
	if err != nil {
		fmt.Print(&res[0])
		fmt.Println(err.Error())
		return
	}

	// output
	Print(res)
}

// 是否为字母
func isCharacter(char byte) bool {
	return (char <= 'z') && (char >= 'a') || (char <= 'Z') && (char >= 'A')
}

// 是否为数字
func isNumber(char byte) bool {
	return (char >= '0') && (char <= '9')
}

package main

import (
	"errors"
)

// 缓冲区大小定义
const TokenBufferSize = 8

// 关键字及其Syntax值
var keywords = map[string]int{
	"begin": 	1,
	"if": 		2,
	"then": 	3,
	"while": 	4,
	"do": 		5,
	"end": 		6,
}

// 词法分析
func LexicalAnalysis(buffer *CharBuffer) ([]Tuple, error) {
	var lexicons []Tuple
	var lexicon = &Tuple{Syntax:-1}
	for lexicon.Syntax != 0 {
		lexicon = nextLexicon(buffer)
		if !lexicon.Valid() {
			return []Tuple{*lexicon}, errors.New("ERROR: Wrong character ")
		}
		lexicons = append(lexicons, *lexicon)
	}
	return lexicons, nil
}

// 获取下一个词
func nextLexicon(buffer *CharBuffer) *Tuple {
	res := &Tuple {
		Token: make([]byte, 0, TokenBufferSize),
		Syntax: 0,
	}
	// 去除 ' ' 和 '\n'
	for buffer.Peak() == ' ' || buffer.Peak() == '\n' {
		buffer.Pop()
	}
	// 处理
	if isCharacter(buffer.Peak()) {
		// 字母及数字 letter(letter|digit)*
		for isCharacter(buffer.Peak()) || isNumber(buffer.Peak()) {
			res.Token = append(res.Token, buffer.Pop())
		}
		if val, ok := keywords[string(res.Token)]; ok {
			// 关键字
			res.Syntax = val
		} else {
			// 非关键字
			res.Syntax = 10
		}
	} else if isNumber(buffer.Peak()) {
		// 数字
		var number = int(buffer.Pop() - '0')
		for isNumber(buffer.Peak()) {
			number = number * 10 + int(buffer.Pop() - '0')
		}
		res.Number = number
		res.Syntax = 11
	} else {
		res.Token = append(res.Token, buffer.Peak())
		// 符号
		switch buffer.Pop() {
		case '+':
			res.Syntax = 13
		case '-':
			res.Syntax = 14
		case '*':
			res.Syntax = 15
		case '/':
			res.Syntax = 16
		case ':':
			if buffer.Peak() == '=' {
				// :=
				res.Token = append(res.Token, buffer.Pop())
				res.Syntax = 18
			} else {
				// :
				res.Syntax = 17
			}
		case '<':
			if buffer.Peak() == '=' {
				// <=
				res.Token = append(res.Token, buffer.Pop())
				res.Syntax = 22
			} else if buffer.Peak() == '>' {
				// <>
				res.Token = append(res.Token, buffer.Pop())
				res.Syntax = 21
			} else {
				// <
				res.Syntax = 20
			}
		case '>':
			if buffer.Peak() == '=' {
				// >=
				res.Token = append(res.Token, buffer.Pop())
				res.Syntax = 24
			} else {
				// >
				res.Syntax = 23
			}
		case '=':
			res.Syntax = 25
		case ';':
			res.Syntax = 26
		case '(':
			res.Syntax = 27
		case ')':
			res.Syntax = 28
		case '#':
			res.Syntax = 0
		default:
			res.Syntax = -1
		}
	}
	return res
}

package main

import "fmt"

// 二元组
type Tuple struct {
	Token	[]byte
	Number 	int
	Syntax 	int
}

// Syntax是否有效
func (tuple *Tuple)Valid() bool {
	return tuple.Syntax != -1
}

// 二元组字符串输出
func (tuple *Tuple)String() string {
	if tuple.Syntax == 11 {
		return fmt.Sprintf("│ %-10d│ %3d │", tuple.Number, tuple.Syntax)
	}
	return fmt.Sprintf("│ %-10s│ %3d │", string(tuple.Token), tuple.Syntax)
}

// 输出二元组列表
func Print(tuples []Tuple) {
	fmt.Println("┌───────────┬─────┐")
	fmt.Println("│   token   │ syn │")
	fmt.Println("├───────────┼─────┤")
	for _,v := range tuples {
		fmt.Println(&v)
	}
	fmt.Println("└───────────┴─────┘")
}

package main

import (
	"fmt"
	"os"
)

// 字符缓冲区
type CharBuffer struct {
	buffer 	[]byte
	idx		int
}

// 新建字符缓冲区
func NewBuffer() *CharBuffer {
	return &CharBuffer{
		buffer:	make([]byte, 0, InputBufferSize),
		idx:	0,
	}
}

// 从标准输入中读取字符
func (charBuffer *CharBuffer)Read(in *os.File) {
	var ch byte
	for ch != '#' {
		fmt.Fscanf(in, "%c", &ch)
		charBuffer.buffer = append(charBuffer.buffer, ch)
	}
}

// 从缓冲区读取一个字符 读取位置后移1个字符
func (charBuffer *CharBuffer)Pop() byte {
	char := charBuffer.buffer[charBuffer.idx]
	charBuffer.idx++
	return char
}

// 从缓冲区读取一个字符
func (charBuffer *CharBuffer)Peak() byte {
	return charBuffer.buffer[charBuffer.idx]
}
```

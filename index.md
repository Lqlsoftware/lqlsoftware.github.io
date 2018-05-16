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

func (hs *HttpStream)Output(start, end uint32) []byte {
	// 重置Buffer
	hs.buffer.Reset()

	// 比头部长度小 输出头部
	hcLen := uint32(len(*hs.hc))
	if start < hcLen {
		if end > hcLen {
			hs.buffer.Write((*hs.hc)[start:hcLen])
			start = hcLen
		} else {
			hs.buffer.Write((*hs.hc)[start:end])
			return hs.buffer.Bytes()
		}
	}
	end = end - hcLen + hs.start
	start = start - hcLen + hs.start

	// 根据流类型输出
	if hs.flag == 1 {
		// file
		data := make([]byte, end - start)
		hs.fd.ReadAt(data, int64(start))
		hs.buffer.Write(data)
	} else if hs.flag == 2 {
		// []byte
		hs.buffer.Write(hs.content.Bytes()[start:end])
	}
	return hs.buffer.Bytes()
}

func DefaultGETHandler(request *HttpRequest, response *HttpResponse, phpPlugin *php.Plugin) {
	// 检查URL
	if *request.url == "/" {
		if !checkFileIsExist("root/index.html") {
			response.stateCode = OK
			response.contents.WriteString(defaultIndex)
			return
		}
		*request.url = "/index.html"
	} else if strings.HasSuffix(*request.url,"/") {
		response.stateCode = Forbidden
		response.contents.WriteString(page403)
		return
	} else if !checkFileIsExist("root" + *request.url) {
		response.stateCode = NotFound
		response.contents.WriteString(page404)
		return
	} else if phpPlugin != nil && strings.HasSuffix(*request.url,".php") {
		var b bytes.Buffer
		w := io.Writer(&b)
		phpPlugin.SetPhpWriter(w)
		err := phpPlugin.Exec("root" + *request.url)
		if err != nil {
			response.stateCode = InternalServerError
			response.contents.WriteString(defaultIndex)
			return
		}
		response.stateCode = OK
		response.contents.Write(b.Bytes())
		(*response.header)["Content-Type"] = "text/html; charset=utf-8"
		return
	}

	cachePath := "root/_temp" + *request.url
	filePath := "root" + *request.url
	f,err := os.Open(filePath)
	check(err)
	var dat []byte

	useGzip,useCache,useSlice := false,false,false
	sliceSize,sliceStart,sliceEnd := getFileSize(f),int64(0),int64(0)

	// 检查断点续传和 gzip 压缩
	if slice,ok := (*request.header)["Range"];ok {
		useSlice = true
		start := strings.Index(slice, "bytes=") + 6
		if start < 0 || start > len(slice) {
			response.stateCode = Forbidden
			response.contents.WriteString(page403)
			return
		}

		var err error
		for i := start;i < len(slice);i++ {
			if slice[i] == '-' {
				sliceStart,err = strconv.ParseInt(slice[start:i],10,64)
				if err != nil {
					response.stateCode = Forbidden
					response.contents.WriteString(page403)
					return
				}

				// 默认全部
				if i + 1 == len(slice) {
					sliceEnd = sliceSize - 1
				} else {
					sliceEnd,err = strconv.ParseInt(slice[i + 1:],10,64)
					if err != nil || sliceSize <= sliceEnd {
						response.stateCode = Forbidden
						response.contents.WriteString(page403)
						return
					}
				}
				break
			}
		}
		if etag,ok := (*request.header)["If-Range"];ok {
			if etag != strconv.FormatInt(getFileModTime(f),10) {
				useSlice = false
			}
		}
	} else if encoding,ok := (*request.header)["Accept-Encoding"];ok && checkType(*request.url) {
		encodes := strings.Split(encoding, ", ")
		// 检查浏览器是否支持gzip压缩
		for _,v := range encodes {
			// 支持gzip压缩
			if v == "gzip" {
				useGzip = true

				// 检查缓存是否有已压缩文件 缓存文件和新文件的修改时间
				if checkFileIsExist(cachePath) && getFileModTime(f) >= getFileModTime(f) && getFileSize(f) < 1 << 30 {
					useCache = true
				}
				break
			}
		}
	}

	if useCache {
		// 设置返回header 通知浏览器压缩格式
		(*response.header)["Content-Encoding"] = "gzip"

		// 直接返回缓存数据
		fd,err := os.Open(cachePath)
		check(err)

		response.contents.SetFileDescriptor(fd, 0, uint32(getFileSize(fd)))
	} else if useSlice {
		response.SetHeader("Content-Range","bytes " + strconv.FormatInt(sliceStart,10) + "-" + strconv.FormatInt(sliceEnd,10) + "/" + strconv.FormatInt(sliceSize,10))
		response.contents.SetFileDescriptor(f, uint32(sliceStart), uint32(sliceEnd - sliceStart + 1))

		if sliceStart == 0 {
			response.stateCode = OK
		} else {
			response.stateCode = PartialContent
		}
		return
	} else {
		// gzip 压缩
		if useGzip {
			// 读入文件
			dat, err = ioutil.ReadFile(filePath)
			check(err)

			// 设置返回header 通知浏览器压缩格式
			(*response.header)["Content-Encoding"] = "gzip"

			// 压缩数据
			var b bytes.Buffer
			w := gzip.NewWriter(&b)
			w.Write(dat)
			w.Flush()
			dat = b.Bytes()
			response.contents.Write(dat)

			// 缓存
			ioutil.WriteFile(cachePath, dat, 0666)
		} else {
			// 文件最后修改时间 - 文件大小字节数转为16进制
			(*response.header)["ETag"] = strconv.FormatInt(getFileModTime(f),10)
			response.contents.SetFileDescriptor(f, 0, uint32(sliceSize))
		}
	}
	// 设置Content-Type
	(*response.header)["Content-Type"] = getContentType(*request.url) + "; charset=utf-8"
	response.stateCode = OK
}

// Default POST method
func DefaultPOSTHandler(request *HttpRequest, response *HttpResponse) {
	response.stateCode = OK
	response.contents.WriteString("POST REQUEST: " + *request.url)
}

// Default HEAD method
func DefaultHEADHandler(request *HttpRequest, response *HttpResponse) {
	response.stateCode = OK
}
```

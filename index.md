``` go
// HTTP请求
type HttpRequest struct {
	// URL
	url			*string
	// HTTP-Version
	version		*string
	// HTTP-Header
	header 		*map[string]string
	// HTTP-Method
	method		HttpMethod
	// HTTP-Content
	contents	*string
	// Param
	param		*map[string]string
}
// HTTP响应
type HttpResponse struct {
	// HTTP-Header
	header 		*map[string]string
	// HTTP-Version
	version		*string
	// HTTP-State
	stateCode	HttpStateCode
	// HTTP-Content
	contents	[]byte
}

// 将TCP交付的HTTP数据包进行解析 返回HTTPREQUEST
func parserRequest(raw []byte) (*HttpRequest,error) {
	// REQUEST
	// 		METHOD URL HTTP-VERSION
	idx,start := 0,0
	for idx < len(raw) && raw[idx] != 13 {
		idx++
	}
	first := strings.Split(string(raw[:idx])," ")
	if len(first) < 3 {
		return nil,errors.New("ERROR: UNKNOWN HTTP CONTENT")
	}
	// 中文URL UNESCAPE
	url, err := unescape(first[1])
	if err != nil {
		return nil,errors.New("ERROR: UNKNOWN HTTP URL ENCODE")
	}
	version := first[2]

	// REQUEST-HEADER
	// 		Connection: keep-alive
	idx += 2
	start = idx
	header := make(map[string]string)
	var key string
	for idx < len(raw) {
		if v := raw[idx];v == 13 {
			if idx <= start {
				break
			}
			header[key] = string(raw[start:idx])
			idx += 2
			start = idx
		} else if v == 58 && raw[idx + 1] == 32 {
			key = string(raw[start:idx])
			idx += 2
			start = idx
		} else {
			idx++
		}
	}

	// REQUEST-CONTENT
	var contents string
	if idx < len(raw) {
		contents = string(raw[idx:])
	}


	// Generate request
	request := &HttpRequest{
		method: 	HttpMethod(raw[0]),
		header: 	&header,
		contents: 	&contents,
		url:		&url,
		version:	&version,
	}

	// Parse request parameter
	request.parseParameter()

	return request, nil
}

// 判断是否keep-alive
if (*request.header)["Connection"] == "keep-alive" {
	isKeepAlive = true
}

// Get router key:  url[0] ^= method -> url
key := []byte(*request.url)
key[0] ^= uint8(request.method)
// 查找URL路由表
if handler,exist := routerMap[string(key)];exist {
	handler(request, response)
	response.stateCode = OK
} else {
	switch request.method {
	case GET:
		DefaultGETHandler(request, response)
	case POST:
		DefaultPOSTHandler(request, response)
	case HEAD:
		DefaultHEADHandler(request, response)
	default:
		DefaultGETHandler(request, response)
	}
}

// response变成字节流
func (rep *HttpResponse)getBytes() []byte {
	// 设置默认头部
	(*rep.header)["Content-Length"] = strconv.Itoa(len(rep.contents))

	// 计算byte总共长度 防止append申请内存拷贝
	length := 38 + len(rep.contents)
	for key,value := range *rep.header {
		length += len(key) + len(value) + 4
	}

	// 申请固定capacity的内存
	buf := make([]byte, 0, length)
	buf = append(buf, []byte(*rep.version)...)
	buf = append(buf, 32)
	buf = append(buf, []byte(strconv.Itoa(int(rep.stateCode)))...)
	buf = append(buf, 32)
	buf = append(buf, []byte(getStateName(rep.stateCode))...)
	buf = append(buf, CRLF...)

	// header
	for key,value := range *rep.header {
		buf = append(buf, []byte(key)...)
		buf = append(buf, SEP...)
		buf = append(buf, []byte(value)...)
		buf = append(buf, CRLF...)
	}
	buf = append(buf, CRLF...)
	
	// content
	buf = append(buf, rep.contents...)
	return buf
}

// 检查URL
if *request.url == "/" {
	response.stateCode = OK
	response.contents = []byte(defaultIndex)
	return
} else if strings.HasSuffix(*request.url,"/") {
	response.stateCode = Forbidden
	response.contents = []byte(page403)
	return
}

useGzip,useCache := false,false
cachePath := "root/_temp" + *request.url
filePath := "root" + *request.url
// 检查支持 gzip 压缩
if encoding,ok := (*request.header)["Accept-Encoding"];ok {
	encodes := strings.Split(encoding, ", ")
	// 检查浏览器是否支持gzip压缩
	for _,v := range encodes {
		// 支持gzip压缩
		if v == "gzip" {
			useGzip = true
			break
		}
	}
}
// 检查缓存是否有已压缩文件
if useGzip && checkFileIsExist(cachePath) && checkFileIsExist(filePath) {
	// 检查缓存文件和新文件的修改时间
	if getFileModTime(cachePath) >= getFileModTime(filePath) {
		useCache = true
	}
}

// 读入文件
dat, err = ioutil.ReadFile("root" + *request.url)
if err != nil {
	response.stateCode = NotFound
	response.contents = []byte(page404)
	return
}

// gzip 压缩
if useGzip {
	// 设置返回header 通知浏览器压缩格式
	(*response.header)["Content-Encoding"] = "gzip"

	// 压缩数据
	var b bytes.Buffer
	w := gzip.NewWriter(&b)
	w.Write(dat)
	w.Flush()
	dat = b.Bytes()

	// 检查是否为静态text类文件
	if checkType(*request.url) {
		// 缓存
		ioutil.WriteFile(cachePath, dat, 0666)
	}
}

// 设置Content-Type
(*response.header)["Content-Type"] = getContentType(*request.url) + "; charset=utf-8"
response.stateCode = OK
response.contents = dat
```

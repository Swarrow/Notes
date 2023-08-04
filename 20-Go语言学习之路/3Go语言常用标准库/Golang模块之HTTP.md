# Golang模块之HTTP

目录

- 0、前言
- 1、HTTP服务端
- 2、HTTP客户端
  - 2.1、GET请求示例
  - 2.2、GET请求URL带参数示例
  - 2.3、POST请求携带Json数据示例1
  - 2.4、POST请求携带Json数据示例1
  - 2.5、POST请求携带Json数据示例2

# 0、前言

Go语言中内置`net/http`包提供了HTTP客户端和服务端的实现

# 1、HTTP服务端

模拟一个HTTP服务端。

```go
package main

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
)

// 定义客户端提交的post请求的json数据内容
type  Auth struct {
	Username string `json: username`
	Password string `json: password`
}

// 定义服务端返回json数据给客户端的内容
type Resp struct {
	Code string `json: code`
	Msg string `json: msg`
}


func f1(w http.ResponseWriter,r *http.Request){
	str := `from home`
	w.Write([]byte(str))
}

func f2(w http.ResponseWriter,r *http.Request){
	b,err := ioutil.ReadFile("./html/index.html")  // 读取到html文件（byte类型切片）
	if err != nil {
		w.Write([]byte(fmt.Sprintf("%v",err)))
	}
	w.Write(b)  // 返回响应数据（必须传入byte类型切片）
}

func f3(w http.ResponseWriter,r *http.Request){
	// 对于GET请求,参数都放在URL上(query param)，请求体中是没有数据的
	queryParam := r.URL.Query() // 自动帮我们识别URL中的urlParam
	query := queryParam.Get("query")
	page  := queryParam.Get("page")
	fmt.Println(query,page)
	fmt.Println(r.URL)     // 查看请求url
	fmt.Println(r.Method)  // 查看请求方法
	fmt.Println(ioutil.ReadAll(r.Body)) // 查看请求的body
	w.Write([]byte("ok"))
}


// post接口接收json数据
func f4(w http.ResponseWriter,r *http.Request){

	// 检查是否为POST请求
	if r.Method != "POST"{
		w.WriteHeader(405) // 返回错误代码
		return
	}
	body,_ := ioutil.ReadAll(r.Body)
	//body_str := string(body)
	//fmt.Println(body_str)

	var auth Auth
	var result Resp
	if err := json.Unmarshal(body,&auth);err == nil {
		// 拿到json数据
		fmt.Printf("用户名:%v 密码:%v",auth.Username,auth.Password)
		
		result.Code = "200"
		result.Msg = "Success"
		// 将返回的数据转化成json格式
		ret,_ := json.Marshal(result)
		w.Write(ret)
	}else{
		result.Code = "500"
		result.Msg = "Failed"
		ret,_ := json.Marshal(result)
		w.Write(ret)
	}
}

func main(){
	http.HandleFunc("/home",f1)
	http.HandleFunc("/index",f2)
	http.HandleFunc("/xxx",f3)
	http.HandleFunc("/login",f4)

	// 启动HTTP服务（监听地址和端口）
	http.ListenAndServe("0.0.0.0:9090",nil)
}
```

# 2、HTTP客户端

HTTP客户端能够发送HTTP请求，如：GET/POST

## 2.1、GET请求示例

```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"net/url"
)

// net/http client

func main(){
	resp,err := http.Get("http://127.0.0.1:9090/index")
	if err != nil {
		fmt.Printf("get url failed,err:%v\n",err)
		return
	}
	body,err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Printf("read resp.Body failed,err:%v",err)
	}
	fmt.Println(string(body))

}
```

## 2.2、GET请求URL带参数示例

我们可以在发送Get请求的时候在url上携带参数，例如：http://xx/xx?query=xx&page=xx

```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"net/url"
)

// net/http client

func main(){
	//resp,err := http.Get("http://127.0.0.1:9090/xxx?query=jack&page=1")
	data := url.Values{} // url encode（携带get请求参数）
	urlObj,_ := url.Parse("http://127.0.0.1:9090/xxx")
	data.Set("query","jack")
	data.Set("page","1")
	queryStr := data.Encode()  // url encode之后的地址
	fmt.Println(queryStr)
	urlObj.RawPath = queryStr // 添加url
	req,err := http.NewRequest("GET",urlObj.String(),nil)

	// 发送请求
	resp,err := http.DefaultClient.Do(req)

	if err != nil {
		fmt.Printf("get url failed,err:%v\n",err)
		return
	}
	defer resp.Body.Close() // 关闭连接

	body,err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Printf("read resp.Body failed,err:%v",err)
	}
	fmt.Println(string(body))
}
```

**短连接**

默认情况下浏览器开启了长连接，如果请求频繁的话，可能会存在长连接还没有关闭，又启动了新的连接，一直这样循环下去，就会导致连接超额，每个连接都会占用资源/网络IO，那么其实可以通过关闭长连接的方式来避免这个问题

```go
// 禁用KeepAlive的client
tr := &http.Transport{
	DisableKeepAlives:      true,
}
client := http.Client{
	Transport:     tr,
}
client.Do(req)
```

## 2.3、POST请求携带Json数据示例1

很多时候，我们在实现POST请求都需要携带对应规范的json格式数据，例如

```go
{
	"username":"admin",
	"password":"123456"	
}
```

实现上面的规范来提交json数据

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
)

type auth struct {
	Username string `json: username`
	Password string `json: password`
}

func main(){
	// post请求
	auths := auth{"admin","123456"}
	bs,_ := json.Marshal(auths) // 将结构体数据转换成json格式
	resp,_ := http.Post("http://127.0.0.1:9090/login","application/json", bytes.NewBuffer([]byte(bs)))
	body, _ := ioutil.ReadAll(resp.Body)
	fmt.Printf("Post request with json result: %s\n", string(body))
}
```

## 2.4、POST请求携带Json数据示例1

很多时候，我们在实现POST请求都需要携带对应规范的json格式数据，例如

```go
{
	"username":"admin",
	"password":"123456"	
}
```

实现上面的规范来提交json数据

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
)

// 我们需要在结构体中添加注解来映射对应的key
type  auth struct {
	Username string `json: username`
	Password string `json: password`
}

func main(){
	// post请求
	auths := auth{"admin","123456"}
	bs,_ := json.Marshal(auths) // 将结构体数据转换成json格式
	resp,_ := http.Post("http://127.0.0.1:9090/login","application/json", bytes.NewBuffer([]byte(bs)))
	body, _ := ioutil.ReadAll(resp.Body)
	fmt.Printf("Post request with json result: %s\n", string(body))
}
```

## 2.5、POST请求携带Json数据示例2

很多时候，我们在实现POST请求都需要携带对应规范的json格式数据，例如

```go
{
	"username":"admin",
	"password":"123456"	
}
```

实现上面的规范来提交json数据

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"go_dev/Project/EyeSkyAgent/conf"
	"io/ioutil"
	"net/http"
)

type  auth struct {
	Username string `json: username`
	Password string `json: password`
}

func main(){
	// post请求
	var data auth
	data.Username = "Jack"
	data.Password = "Jack123"
	bs, err := json.Marshal(data)

	reader := bytes.NewReader(bs)
	request, err := http.NewRequest("POST", "http://127.0.0.1:9090/login", reader)
	if err != nil{
		conf.Logger.Error("请求server端失败...")
	}
	// 携带头部
	request.Header.Set("Content-Type", "application/json;charset=UTF-8")
	client := http.Client{}
	// 返回服务端的响应数据
	resp, err := client.Do(request)
	if err != nil {
		fmt.Println("请求获取响应失败")
	}
	body, _ := ioutil.ReadAll(resp.Body)
	fmt.Printf("Post request with json result: %s\n", string(body))
}
```


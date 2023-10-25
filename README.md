# 对于sylar框架的应用
基础sylar框架可见 https://github.com/JiaoZixun/WebNetwork_Sylar

### 1.对用户提供可改写接口实现网络服务
/apply_sylar/sylar/service
包含：
1）service_base
提供网络服务的基类，包含了request、response、session类对象，进行存储，完成跨域头的设置，以及参数校验功能，为上层提供Handle()实现具体操作逻辑
2）service_check
实现参数校验，并且根据request报文跳转到不同执行函数
3）service_demo
实际逻辑函数的具体实现，通过下层返回的信息进行response报文组装，并返回

### 2.具体使用
/apply_sylar/examples/sylar_demo.cpp
```c++
void Listen() {
    // 事件组
    EventVec events;
    // 服务执行函数
    auto reuqest_func = [](sylar::http::HttpRequest::ptr request
                    , sylar::http::HttpResponse::ptr response
                    , sylar::http::HttpSession::ptr session){
        sylar::ServiceDemo::ptr service(new sylar::ServiceDemo(request, response, session));
        service->processRequest();
        return 0;
    };
    // 服务路由
    auto request_route = "/sylar/uploadfile";
    events.push_back(std::make_pair(request_route, reuqest_func));
    // 开始执行函数
    sylar::Listen(events);
}

int main() {
    sylar::InitServer();
    sylar::IOManager iom(2);
    iom.scheduler(Listen);
    // Listen();
    return 0;
}
```
设置执行函数，交由ServiceDemo执行，最后发送响应报文
设置服务路由和执行函数相绑定
然后开始监听全部事件
整体逻辑函数底层使用协程执行


## 1.文件上传功能实现

1）request报文格式
```C++
// POST /sylar/uploadfile HTTP/1.1
// connection: close
// Accept:application/json, text/plain, */*
// Accept-Encoding:gzip, deflate
// Accept-Language:zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
// Content-Length:197
// Content-Type:multipart/form-data; boundary=----WebKitFormBoundaryKsQT8wF6o17XAJI1
// credentials:same-origin
// Host:xxx.xxx.xxx.xxx:8020
// Origin:http://localhost:8080
// Referer:http://localhost:8080/
// User-Agent:Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/118.0.0.0 Safari/537.36 Edg/118.0.2088.61
// content-length: 197

// ------WebKitFormBoundaryKsQT8wF6o17XAJI1
// Content-Disposition: form-data; name="file"; filename="hello.txt"
// Content-Type: text/plain

// hello world！
// ------WebKitFormBoundaryKsQT8wF6o17XAJI1--
```
Content-Type:中boundary=指向了body中文件内容的标记
Content-Disposition:中filename=指向了文件名
上传任务：
1.从Content-Type中找到分割标记
2.从body中取出分割标记中间的内容
3.Content-Disposition: form-data; name="file"; filename="hello.txt"，取出filename
4.Content-Type: text/plain 表示文件 还有其他标记表示音视频等
5.剩下的是文件内容

```c++
// 上传任务
void ServiceDemo::PostHandle() {
    setCORS();  // 设置跨域
    const char test_response_data[] = "hello world! PostHandle";
    m_res->setBody(test_response_data);
    // SYLAR_LOG_INFO(g_logger) << *m_req;
    // 上传任务：
    // 1.从Content-Type中找到分割标记
    // 2.从body中取出分割标记中间的内容
    // 3.Content-Disposition: form-data; name="file"; filename="hello.txt"，取出filename
    // 4.Content-Type: text/plain 表示文件
    // 5.剩下的是文件内容
    int flag_pos = m_req->getHeader("Content-Type").find("boundary=") + 9;
    std::string flag = m_req->getHeader("Content-Type").substr(flag_pos);
    SYLAR_LOG_INFO(g_logger) << "flag: " << flag;
    std::string body = m_req->getBody();
    int filename_pos = body.find("filename") + 10;
    std::string filename = "";
    while(body[filename_pos] != '"') {
        filename += body[filename_pos];
        ++filename_pos;
    }
    SYLAR_LOG_INFO(g_logger) << "filename: " << filename;
    int tmp_pos = body.find("Content-Type:");   // 类型可能有视频、文本
    int filedata_pos_st = body.find("\n", tmp_pos)+3;     // 找到Content-Type:后面第一个空行
    int filedata_pos_end = body.find("--"+flag+"--")-2;

    std::string filedata = body.substr(filedata_pos_st, filedata_pos_end-filedata_pos_st);
    // SYLAR_LOG_INFO(g_logger) << "filedata: " << filedata;
    std::string uproot = "xxx/Up_Files/" + filename;   // 服务器存储路径，将xxx替换为自己的路径即可
    std::ofstream file(uproot, std::ios::binary);
    file << filedata;
}
```
需要注意的是，根据标记取出body中文件部分，还包含了许多其他信息，
Content-Disposition: form-data; name="file"; filename="hello.txt"
Content-Type: text/plain
和空行，文件需要将前后空行都剔除才可以完整还原文件内容，text文件不去除没影响，但是video文件如果不去除前后空行会变成乱码，无法打开


2）跨域问题
在发送POST报文之前会先发送一个OPTIONS请求，需要加上跨域头返回后，接收到POST请求报文
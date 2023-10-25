# 对于sylar框架的应用
基础sylar框架可见 https://github.com/JiaoZixun/WebNetwork_Sylar

## 1.对用户提供可改写接口实现网络服务
/apply_sylar/sylar/service
包含：
1）service_base
提供网络服务的基类，包含了request、response、session类对象，进行存储，完成跨域头的设置，以及参数校验功能，为上层提供Handle()实现具体操作逻辑
2）service_check
实现参数校验，并且根据request报文跳转到不同执行函数
3）service_demo
实际逻辑函数的具体实现，通过下层返回的信息进行response报文组装，并返回

## 2.具体使用
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
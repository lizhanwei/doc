#### 概述
关于跨域，有N种类型，本文只专注于ajax请求跨域(ajax跨域只是属于浏览器“同源策略”中的一部分,其它的还有Cookie跨域iframe跨域,LocalStorage跨域等这里不做介绍

#### 什么是ajax请求跨域
ajax出现请求跨域错误问题,主要原因就是因为浏览器的“同源策略”【协议，域名，端口相同】,可以参考
>[浏览器同源政策及其规避方法](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)  
>[跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)  
>[HTTP访问控制](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)

##### 是否跨域举例

| URL        | 说明    |  是否允许通信  |
| --------   | -----: | :----: |
|http://www.a.com/a.js
|http://www.a.com/b.js    | 同一域名下   |允许
|http://www.a.com/lab/a.js
|http://www.a.com/script/b.js |同一域名下不同文件夹 |允许
|http://www.a.com:8000/a.js
|http://www.a.com/b.js    | 同一域名，不同端口  |不允许
|http://www.a.com/a.js
|https://www.a.com/b.js |同一域名，不同协议 |不允许
|http://www.a.com/a.js
|http://70.32.92.74/b.js |域名和域名对应ip |不允许
|http://www.a.com/a.js
|http://script.a.com/b.js |主域相同，子域不同 |不允许
|http://www.a.com/a.js
|http://a.com/b.js |同一域名，不同二级域名（同上） |不允许（cookie这种情况下也不允许访问）
|http://www.cnblogs.com/a.js
|http://www.a.com/b.js |不同域名 |不允许

#### 如何处理

* 开发环境处理办法
>[Vue-cli proxyTable 解决开发环境的跨域问题](https://www.jianshu.com/p/95b2caf7e0da)

* 线上环境处理
> 客户端【非简单请求，会产生Options请求】
```javaScript 
instance.interceptors.request.use(
  (axiosConfig): any => {
    axiosConfig.headers['Content-Type'] = 'application/json';
    return axiosConfig;
  },

  (error: any) => {
    // 对请求错误做些什么
    return Promise.reject(error);
  }
);
```
>服务端
```java
public class CORSInterceptor extends HandlerInterceptorAdapter {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        response.setHeader("Access-Control-Allow-Origin", "*");
        response.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE");
        response.setHeader("Access-Control-Max-Age", "3600");
        response.setHeader("Access-Control-Allow-Headers", "x-requested-with,Content-Type,Authorization,token");
        return !"option".equalsIgnoreCase(request.getMethod());
    }
}
	<mvc:interceptors>
	    <bean class="com.xxx.xxx.interceptor.CORSInterceptor" />
	</mvc:interceptors>
```

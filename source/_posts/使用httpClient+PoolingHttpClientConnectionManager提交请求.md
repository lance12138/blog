---
title: 使用httpClient+PoolingHttpClientConnectionManager提交请求
date: 2016-08-06 22:17:27
tags: httpClient,连接池
---

#### 使用连接池的好处

大家都知道http连接是基于tcp的，而tcp创建连接需要三次握手，断开连接四次挥手，如果我们不使用连接池，那么每发出一个请求，就需要三次握手和四次挥手，而三次握手和四次挥手都是耗资源的操作。试想如果频繁的发出请求，性能是不是会是个瓶颈。所以HttpClient在4之后就出现了连接池的概念，当请求结束并不是直接断开连接，而是返回给连接池方便下次调用。

#### 连接池配置

使用连接池主要是用到PoolingHttpClientConnectionManager这个类，基本上的配置像Cookie配置策略、连接数的控制都是在ConnectionManager中配置的,所以在调用httpClient时必须先初始化ConnectionManager。
```
private static PoolingHttpClientConnectionManager clientConnectionManager=null;
    private static CloseableHttpClient httpClient=null;
    private static RequestConfig config = RequestConfig.custom().setCookieSpec(CookieSpecs.STANDARD_STRICT).build();
    private final static Object syncLock = new Object();
    
      @PostConstruct
    private void init(){

        Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder.<ConnectionSocketFactory>create()
                .register("https", SSLConnectionSocketFactory.getSocketFactory())
                .register("http", PlainConnectionSocketFactory.getSocketFactory())
                .build();
        clientConnectionManager =new PoolingHttpClientConnectionManager(socketFactoryRegistry);
        clientConnectionManager.setMaxTotal(50);
        clientConnectionManager.setDefaultMaxPerRoute(25);
    }
```
<!--more-->
#### 创建httpClient实例
一般在这个实例中我们将CookieStore传入并让其一直持有Cookie

```
public static CloseableHttpClient getHttpClient(){
        if(httpClient == null){
            synchronized (syncLock){
                if(httpClient == null){
                      CookieStore cookieStore = new BasicCookieStore();
                    BasicClientCookie cookie = new BasicClientCookie("sessionID", "######");
                    cookie.setDomain("#####");
                    cookie.setPath("/");
                    cookieStore.addCookie(cookie);
                    httpClient =HttpClients.custom().setConnectionManager(clientConnectionManager).setDefaultCookieStore(cookieStore).setDefaultRequestConfig(config).build();
                }
                }
            }
        }
        return httpClient;
    }
```
#### 创建POST/GET请求方法
```
 public static HttpEntity httpGet(String url, Map<String,Object> headers){
        CloseableHttpClient httpClient = getHttpClient();
        HttpRequest httpGet = new HttpGet(url);
        if(headers!=null&&!headers.isEmpty()){
            httpGet = setHeaders(headers, httpGet);
        }
        CloseableHttpResponse response = null;
        try{
            response =httpClient.execute((HttpGet)httpGet);
            HttpEntity entity = response.getEntity();
            return entity;
        }catch (Exception e){
            e.printStackTrace();

        }
        return null;
    }

    /**
     * post请求,使用json格式传参
     * @param url
     * @param headers
     * @param data
     * @return
     */
    public static HttpEntity httpPost(String url,Map<String,Object> headers,String data){
        CloseableHttpClient httpClient = getHttpClient();
        HttpRequest request = new HttpPost(url);
        if(headers!=null&&!headers.isEmpty()){
            request = setHeaders(headers,request);
        }
        CloseableHttpResponse response = null;

        try {
            HttpPost httpPost = (HttpPost) request;
            httpPost.setEntity(new StringEntity(data, ContentType.create("application/json", "UTF-8")));
            response=httpClient.execute(httpPost);
            HttpEntity entity = response.getEntity();
            return entity;
        } catch (IOException e) {
            e.printStackTrace();

        }
        return null;
    }
    /**
    使用表单键值对传参
    */
    public static HttpEntity PostForm(String url,Map<String,Object> headers,List<NameValuePair> data){
        CloseableHttpClient httpClient = getHttpClient();
        HttpRequest request = new HttpPost(url);
        if(headers!=null&&!headers.isEmpty()){
            request = setHeaders(headers,request);
        }
        CloseableHttpResponse response = null;
        UrlEncodedFormEntity uefEntity;
        try {
            HttpPost httpPost = (HttpPost) request;
            uefEntity = new UrlEncodedFormEntity(data,"UTF-8");
            httpPost.setEntity(uefEntity);
           // httpPost.setEntity(new StringEntity(data, ContentType.create("application/json", "UTF-8")));
            response=httpClient.execute(httpPost);
            HttpEntity entity = response.getEntity();
            return entity;
        } catch (IOException e) {
            e.printStackTrace();

        }
        return null;
    }
    /**
     * 设置请求头信息
     * @param headers
     * @param request
     * @return
     */
    private static HttpRequest setHeaders(Map<String,Object> headers, HttpRequest request) {
        for (Map.Entry entry : headers.entrySet()) {
            if (!entry.getKey().equals("Cookie")) {
                request.addHeader((String) entry.getKey(), (String) entry.getValue());
            } else {
                Map<String, Object> Cookies = (Map<String, Object>) entry.getValue();
                for (Map.Entry entry1 : Cookies.entrySet()) {
                    request.addHeader(new BasicHeader("Cookie", (String) entry1.getValue()));
                }
            }
        }
        return request;
    }

```
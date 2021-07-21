[TOC]

# 利用Mock在单元测试中模拟Client

## 简介

在单元测试的过程中，往往会因为远端服务的异常而导致测试失败，因此，利用Mock就可以在测试方法中模拟本机对远端服务Client的请求和对应的返回内容；

## 实现方式

目前来说学习到了两种方式，一种方式是通过创建`MockClient`实现，另一种方式则是通过注解`@MockBean`来实现；

### 创建新的MockClient

通过在测试类中创建新的`MockClient`，我们需要在测试类内定义一个方法以创建新的Client。值得注意的是，**通过创建`MockClient`方法的Client只能在测试类里面使用，在多层调用的时候无法使用。**

```JAVA
@BeforeEach
// 用@BeforeEach注解保证这个MockClient在其余的测试方法执行前已经执行过了
void setUp() throws IOException {
    // 在try后面接上括号是一种try-with-resource的方式，这种方式能够实现自动关闭流
    try (InputStream input = getClass().getResourceAsStream("/tests/isearch.json")) {
        //当我们将json文件读进来之后，转为了字节流的形式以模拟HTTP传输
        byte[] data = toByteArray(input);
        //创建一个新的MockClient
        MockClient mockClient = new MockClient();
        //创建一个新的模拟Feign服务
        isearchService = Feign.builder()
            .contract(new SpringMvcContract())
            //设置解码方式为Json
            .decoder((response, type) -> Json.fromJson(type, response.body().asReader(
                Charset.defaultCharset())))
            //在这一部分最为重要，我们需要定义我们访问的带参数的远端服务的URL，并且将返回的数据与其绑定
            .client(mockClient
                    .ok(HttpMethod.GET,
                        "/v1/search/insite?content_source=techmap&weight=weight_true"
                        + "&user=willieyin&query=k8s&offset=1&limit=10",
                        data)
                    .ok(HttpMethod.GET,
                        "/v1/search/insite?content_source=techmap&weight=weight_true"
                        + "&user=willieyin&offset=1&limit=10",
                        data))
            //此外，在target这里我们要将Client原本的类注入
            .target(new MockTarget<>(ISearchService.class));
    }
}
```



---

### 使用@MockBean注解

在前面说过，如果远端服务`Client`在被测试方法的内部被调用，那么在测试类里进行Mock就没有用。这时候，我们采用`@MockBean`实现，将我们Mock出来的远端服务类的Bean，替换掉所有的项目中原本的Bean。

```java
@MockBean
//用@MockBean注解修饰我们希望Mock的远端服务的Bean
RemoteWikiService iwikiClient;

@BeforeEach
//保证在所有Test方法执行前，就把Bean初始化
public void init() {
    //与上一个Mock方式不同的是，这一次识别的是调用的方法，重载的方法需要以传参的方式进行区分
    Mockito.when(iwikiClient.searchTrees(31129323, "0"))
        .thenAnswer(new Answer<Object>() {
            @Override
            public Object answer(InvocationOnMock invocation) throws Throwable {
                Map ans = Json.fromJsonFile(Map.class, new File("./src/test/resources/tests/wikiService.json"));
                return ans;
            }
        });
    // Mockito中的方法可以让我们模拟参数输入，比如下面的方法，通过Mockito.anyLong()，我们截取了所有以Long为参数的getContent方法调用
    Mockito.when(iwikiClient.getContent(Mockito.anyLong()))
        .thenAnswer(new Answer<Object>() {
            @Override
            public Object answer(InvocationOnMock invocation) throws Throwable {
                //在invocation中，会携带invoke的方法名，传入参数等信息，我们可以通过getArguments()来获取传入的参数，并以此简化代码，避免相同方法不同参数带来的冗余定义
                Object[] arguments = invocation.getArguments();
                Long cfId = (Long) arguments[0];
                //由于page_id是连续的，可以用这个小trick避免重复定义相似的方法
                String path = "./src/test/resources/tests/wikiService" + (cfId - 31129322) + ".json";
                Map ans = Json.fromJsonFile(Map.class, new File(path));
                return ans;
            }
        });
}
```



---

## 具体细节

### MockClient类

首先我们从创建一个新的MockClient开始，首先其调用了`ok()`方法并往里面传入了访问远端服务的HTTP请求方法，确定性URL和返回数据的字节流。

```java
            .client(mockClient
                    .ok(HttpMethod.GET,
                        "/v1/search/insite?content_source=techmap&weight=weight_true"
                        + "&user=willieyin&query=k8s&offset=1&limit=10",
                        data)
                    .ok(HttpMethod.GET,
                        "/v1/search/insite?content_source=techmap&weight=weight_true"
                        + "&user=willieyin&offset=1&limit=10",
                        data))
```

接下来的内部调用如下：

```java
    public MockClient ok(HttpMethod method, String url, byte[] responseBody) {
        //第一步，我们将请求方法和URL打包为了一个requestKey,其次将responseBody继续返回回去
        return ok(RequestKey.builder(method, url).build(), responseBody);
    }
    MockClient ok(RequestKey requestKey, byte[] responseBody) {
        //第二步，我们进一步调用ok方法的重载，并设定HTTP的回复状态码，这里为OK，状态码为200
        return add(requestKey, HttpURLConnection.HTTP_OK, responseBody);
    }
    MockClient add(RequestKey requestKey, int status, byte[] responseBody) {
        //第三步，我们要将responseBody以及status等等信息包装为一个Response.Builder
        return add(requestKey, Response.builder()
                .status(status)
                .reason("Mocked")
                .headers(EMPTY_HEADERS)
                .body(responseBody)); //其中我们的字节流数据就保存在这个Response.Builder的body里面
    }
    MockClient add(RequestKey requestKey, Response.Builder response) {
        //第四部，我们利用前面封装好的requestKey和Response.Builder构建一个RequestResponse对象，并加入到responses链表中，
        // private final List<RequestResponse> responses = new ArrayList<>();
        responses.add(new RequestResponse(requestKey, response));
        return this;
    }
```

至此，我们将传入的URL和data结合其他信息（例如HTTP状态）全都封装到了`MockClient`里面的`responses`List中。



那么，一个`MockClient`是怎么启动并回应请求的呢，可以看的`MockClient`继承了`Client`并重写了其中的`execute()`方法。虽然还没看到调用此方法的源码，但是可以猜测到，这类重写的方法一般都是要被调用的~

#### execute()

```java
    @Override
    public synchronized Response execute(Request request, Request.Options options)
            throws IOException {
        //execute()方法传入了一个request（表示外部请求），以及一个Request.Options
        
        //首先通过外部请求request，我们可以构建出一个requestKey，注意，在上面构建MockClient的时候，我们在第四步，将requestKey以及response.Builder构建为了一个RequestResponse并存在了MockClient的List中，想必我们通过这个请求进来的request构建的requestKey，可以用类似遍历查表的方式从而找对我们Mock出来的对应的ResponseBody!
        RequestKey requestKey = RequestKey.create(request);
        //声明一个新的 Response.Builder
        Response.Builder responseBuilder;
        //sequential是一个boolean值，表示是否进行序列化查找，默认是false
        if (sequential) {
            responseBuilder = executeSequential(requestKey);
        } else {
            responseBuilder = executeAny(request, requestKey);
        }

        return responseBuilder
                .request(request)
                .build();
        //最后调用了build()，相当于吧Builder里面的属性变量都赋给了最终返回的Response，而我们预先定义好的data就在Response.body里面！
    }
```

#### executeSequential()

从名字上来看，这个方法实现了顺序查找**下一个**并返回response。

```java
    private Response.Builder executeSequential(RequestKey requestKey) {
        if (responseIterator == null) {
            //responses是一个List，可以直接调用其的iterator()进行迭代查找
            responseIterator = responses.iterator();
        }
        if (!responseIterator.hasNext()) {
            throw new VerificationAssertionError("Received excessive request %s", requestKey);
        }

        RequestResponse expectedRequestResponse = responseIterator.next();
        //equalsExtended()方法会返回一个boolean值，其作用就是判断传进来的key和List中存的这个key是否相同
        if (!expectedRequestResponse.requestKey.equalsExtended(requestKey)) {
            throw new VerificationAssertionError("Expected %s, but was %s",
                    expectedRequestResponse.requestKey, requestKey);
        }
		//如果这个request确实对应了一个responseBuilder，那就返回
        Response.Builder responseBuilder = expectedRequestResponse.responseBuilder;
        return responseBuilder;
    }
```

#### executeAny()

```java
    private Response.Builder executeAny(Request request, RequestKey requestKey) {
        Response.Builder responseBuilder;
        //requests是一个map，用来存储对应requestKey下的对应请求List
        //private final Map<RequestKey, List<Request>> requests = new HashMap<>();
        if (requests.containsKey(requestKey)) {
            requests.get(requestKey).add(request);
        } else {
            requests.put(requestKey, new ArrayList<>(Arrays.asList(request)));
        }
		//调用getResponseBuilder()方法遍历寻找我们需要的responseBody
        responseBuilder = getResponseBuilder(request, requestKey);
        return responseBuilder;
    }
```

#### getResponseBuilder()

```java
    private Response.Builder getResponseBuilder(Request request, RequestKey requestKey) {
        Response.Builder responseBuilder = null;
        //遍历responses链表，寻找外部请求request对应的RequestKey
        for (RequestResponse requestResponse : responses) {
            if (requestResponse.requestKey.equalsExtended(requestKey)) {
                responseBuilder = requestResponse.responseBuilder;
                // Don't break here, last one should win to be compatible with previous
                // releases of this library!
            }
        }
        //如果遍历完了没找到，就返回404not found
        if (responseBuilder == null) {
            responseBuilder = Response.builder()
                    .status(HttpURLConnection.HTTP_NOT_FOUND)
                    .reason("Not mocker")
                    .headers(request.headers());
        }
        return responseBuilder;
    }
```
































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


[TOC]

# 一些零散的小笔记

### @Table：声明当前实体类对应的数据库表的名字

<img src="C:\Users\emoli\Desktop\typora pic\一些零散的笔记\image-20210716112045621.png" style ="zoom:80%"/>

---

### @Builder：实现建造者模式，可以灵活实现实体类内属性的赋值

```JAVA
import lombok.Builder;
import lombok.ToString;

/**
 * @author wulongtao
 */
@ToString
@Builder
public class UserExample {
    private Integer id;
    private String name;
    private String address;
}
 
UserExample userExample = UserExample.builder()
                .id(1)
                .name("aaa")
                .address("bbb")
                .build();

System.out.println(userExample);
```

---

### @Scheduled：springboot提供的用于定时任务控制的注解

学习资料：https://juejin.cn/post/6844903470793752584

---

### Serializable接口

学习资料：https://zhuanlan.zhihu.com/p/66210653

Serializable是http://java.io包中定义的、用于实现Java类的序列化操作而提供的一个语义级别的接口。Serializable序列化接口没有任何方法或者字段，只是用于标识可序列化的语义。实现了Serializable接口的类可以被ObjectOutputStream转换为字节流，同时也可以通过ObjectInputStream再将其解析为对象。**例如，我们可以将序列化对象写入文件后，再次从文件中读取它并反序列化成对象，也就是说，可以使用表示对象及其数据的类型信息和字节在内存中重新创建对象。**

* 序列化是指把对象转换为字节序列的过程，我们称之为对象的序列化，就是把内存中的这些对象变成一连串的字节(bytes)描述的过程。
* 反序列化则相反，就是把持久化的字节文件数据恢复为对象的过程。那么什么情况下需要序列化呢?大概有这样两类比较常见的场景：1)、需要把内存中的对象状态数据保存到一个文件或者数据库中的时候，这个场景是比较常见的，例如我们利用mybatis框架编写持久层insert对象数据到数据库中时;2)、网络通信时需要用套接字在网络中传送对象时，如我们使用RPC协议进行网络通信时;

---

### @ActiveProfiles("cron")

放在测试类上，可以让@Profile("cron")注解修饰的定时任务类也可以被注入进来测试，但是存在问题，**如果测试的时候刚好有别的定时任务时间到了，就会启动一起进行，所以建议不要在测试类上加入该注解.**
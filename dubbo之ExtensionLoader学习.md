# Dubbo学习之ExtensionLoader类介绍
---
## 简介
ExtensionLoader类是根据扩展点名称来对扩展点接口实现进行的一系列操作，如果获取扩展点接口实现实例，适配类实例，更新实现实例等等。
### JDK SPI
在学习ExtensionLoader之前，先了解一下什么是SPI。

SPI的全称是Service Provider Interface。其目的就是为某接口寻找服务实现的机制，有些像IOC的思想，就是将装配的控制权移到程序之外，在模块化设计中这个机制尤其重要。

Java SPI的具体约定是：当服务的Provider提供了服务接口的一种实现之后，在jar包的META-INF/services/目录下同时创建一个以服务接口命名的文件，该文件中就是实现该服务接口的具体实现类。当外部程序装配这个模块的时候，就能通过该jar包的META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入，从而不需要在代码中指定具体的实现类。

### 实例
目前有一个搜索模块，是基于接口编程的，搜索的实现可能是基于文件搜索的，也可能是基于数据库搜索的。

接口定义如下：
```java
public interface Search {  
   public List serch(String keyword);  
}
```
A公司使用基于文件搜索的方式实现了Search接口，B公司使用基于数据库搜索的方式实现了Search接口。

A公司的实现类：
```txt
com.A.spi.impl.FileSearch
```
那么A公司在发布实现jar包时，就要在jar包下的META-INF/services/xx.xx.xx.Search文件中写出如下内容：
com.A.spi.impl.FileSearch

B公司的实现类：
```txt
com.B.spi.impl.DatabaseSearch
```
那么B公司在发布实现jar包时，就要在jar包下的META-INF/services/xx.xx.xx.Search文件中写出如下内容：
com.B.spi.impl.DatabaseSearch

以下是创建Search的工厂类，使用JDK的serviceLoader的load方法查找实现类的loader
```java
public class SearchFactory {  
    private SearchFactory() {  
    }  
    public static Search newSearch() {  
        // Search search = null;  
        ServiceLoader<Search> serviceLoader = ServiceLoader.load(Search.class);  
        // Iterator<Search> searchs = serviceLoader.iterator();  
        // if (searchs.hasNext()) {  
            // search = searchs.next();  
        // }  
        // return search;  
    }  
}
```

### ExtensionLoader成员变量
变量名|值|意义
--|--|--
SERVICES_DIRECTORY|META-INF/services/|存放接口实现的文件的位置
DUBBO_DIRECTORY|META-INF/dubbo/|存放dubbo其他相关文件的位置
DUBBO_INTERNAL_DIRECTORY|META-INF/dubbo/internal/|dubbo内部文件位置
NAME_SEPARATOR|\s*[,]+\s*|分割正则表达式
EXTENSION_LOADERS|ConcurrentMap<Class<?>, ExtensionLoader<?>>|--
EXTENSION_INSTANCES|ConcurrentHashMap<Class<?>, Object>|--
type|Class<?>|希望加载的扩展点类型
objectFactory|ExtensionFactory|扩展工厂类
cachedNames|ConcurrentMap<Class<?>, String>|缓存的扩展接口名
cachedClasses|Holder<Map<String, Class<?>>>|--
cachedActivates|Map<String, Activate>|--
cachedInstances|ConcurrentMap<String, Holder<Object>>|缓存的实例
cachedAdaptiveInstance|Holder<Object>|--
cachedAdaptiveClass|Class<?>|--
cachedDefaultName|String|缓存默认名称
cachedWrapperClasses|Set<Class<?>>|--

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
EXTENSION_INSTANCES|ConcurrentHashMap<Class<?>, Object>|全局缓存，存放着所有的扩展点实现类型与其对应的已经实例化的实例对象
type|Class<?>|希望加载的扩展点类型
objectFactory|ExtensionFactory|扩展工厂类
cachedNames|ConcurrentMap<Class<?>, String>|扩展点实现类对应的名称(如配置多个名称则值为第一个)
cachedClasses|Holder<Map<String, Class<?>>>|缓存的扩展类型
cachedActivates|Map<String, Activate>|当前Extension实现自动激活实现缓存(map,无序)
cachedInstances|ConcurrentMap<String, Holder<Object>>|缓存的Extension实例
cachedAdaptiveInstance|Holder<Object>|缓存的自适应Extension实例
cachedAdaptiveClass|Class<?>|当前Extension类型对应的AdaptiveExtension类型(只能一个)
cachedDefaultName|String|缓存默认实现名称
cachedWrapperClasses|Set<Class<?>>|当前Extension类型对应的所有Wrapper实现类型(无顺序)。用装饰者模式包装Extension的类,wrapper有且只有一个以interface为参数的构造函数
由Set<Class<?>> cachedWrapperClasses持有其引用

### 构造方法
```java
private ExtensionLoader(Class<?> type) {
    this.type = type;
    // 如果扩展类型是ExtensionFactory,那么则设置为null
    // 这里通过getAdaptiveExtension方法获取一个运行时自适应的扩展类型
    //(每个Extension只能有一个@Adaptive类型的实现，如果没有dubbo会动态生成一个类)
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```
这里设置了一个objectFactory，这是一个ExtensionFactory类型，作用是加载类的扩展。
```java
@SPI
public interface ExtensionFactory {

    /**
     * Get extension.
     *
     * @param type object type.
     * @param name object name.
     * @return object instance.
     */
    <T> T getExtension(Class<T> type, String name);

}
```
它也被SPI注解，是一个扩展点。
在com.alibaba.dubbo.common.extension.factory包中有两个ExtensionFactory接口的实现，一个是AdaptiveExtensionFactory，另一个是SpiExtensionFactory.
![](images/ExtensionFactory实现图.png)
```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {
```
```java
public class SpiExtensionFactory implements ExtensionFactory {
```
可以看出，AdaptiveExtensionFactory类上有@Adaptive注解，而SpiExtensionFactory类没有。说明AdaptiveExtensionFactory就是ExtensionFactory的自适应扩展实现。

Dubbo中，每个扩展点最多只有一个自适应扩展实现，如果所有实现类中没有一个带@Adaptive注解，那么Dubbo会动态的生成一个自适应扩展实现。

也就是说所有调用ExtensionFactory调用的地方都是调用了AdaptiveExtensionFactory的实现方法。
```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        // 将所有ExtensionFactory实现保存起来
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    public <T> T getExtension(Class<T> type, String name) {
    	// 依次遍历各个ExtensionFactory实现的getExtension方法，一旦获取到Extension即返回
        // 如果遍历完所有的ExtensionFactory实现均无法找到Extension,则返回null
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
}
```
构造方法中，首先获取ExtensionFactory的ExtensionLoader对象，调用loader的getSupportedExtensions方法获取系统中所有的ExtensionFactory实现，并保存在factories中。

getExtension()方法是通过系统中所有的ExtensionFactory实现获取Extension，一旦获得，立刻返回，如果遍历完还没有Extension，则返回空。

### getExtension()方法
方法流程：

getExtension(name)</br>
　　->createExtension(name) #如果缓存cachedInstances中没有实例，需要创建　</br>
　　　　->getExtensionClasses() #获得所有的扩展类型（在文件中写的）</br>
　　　　　　　->loadExtensionClasses() #如果缓存cachedClasses中没有类型，需要创建，通过文件获取扩展类</br>
　　　　　　　　　　->loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY) #从"META-INF/dubbo/internal/"中获取</br>
　　　　　　　　　　->loadFile(extensionClasses, DUBBO_DIRECTORY) #从"META-INF/dubbo/"中获取</br>
　　　　　　　　　　->loadFile(extensionClasses, SERVICES_DIRECTORY) #从"META-INF/services/"中获取</br>
　　　　->injectExtension() #扩展点注入 通过instance的set方法截取属性名以及类型，通过objectFactory</br>
　　　　　　　　　　　　　（ExtensionFactory的getExtension()方法获取扩展点，如果获取到了，通过动态代理执行</br>
　　　　　　　　　　　　　　instance的set方法，得以注入扩展点）</br>
　　　　->injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance)) #将缓存包装类的扩展点也进行注入</br>
　　　　->return instance #返回instance
### getAdaptiveExtension()方法
方法流程：

getAdaptiveExtension()</br>
　　->createAdaptiveExtension() #如果缓存cachedAdaptiveInstance中没有自适应实例，则创建</br>
　　　　->getAdaptiveExtensionClass()  # 获取自适应扩展类</br>
　　　　　　->getExtensionClasses() #获得所有的扩展类型（在文件中写的）</br>
　　　　　　->createAdaptiveExtensionClass() #如果没有缓存的自适应类，需要创建一个</br>
　　　　　　　　->createAdaptiveExtensionClassCode()  #动态实现实现类Java代码</br>
　　　　　　　　->compiler.compile(code, classLoader) #动态编译java代码，加载类并实例化返回</br>
　　　　->injectExtension() #注入instance

### 总结
* 扩展点和Extension实例是一对一的关系
* 一个扩展点实现可以对应多个名称（逗号分隔）
* 对于需要等到运行时才能决定使用哪一个具体实现的扩展点，应获取其自适应扩展点实现
* @Adapter注解要么注释在扩展点@SPI的方法上，要么注释在其实现类的类定义上
* 每个扩展点最多只能有一个被AdaptiveExtension
* 由于每个扩展点实现最多只有一个实例，因此扩展点实现应保证线程安全
* 如果扩展点有多个Wrapper，name最终其执行的顺序不确定（使用ConcurrentHashSet存储）

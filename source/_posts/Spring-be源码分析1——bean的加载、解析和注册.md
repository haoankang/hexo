---
title: Spring源码分析1——bean的加载、解析和注册
date: 2019-04-30 14:32:06
categories:
tags: spring
---

##1. 解析Spring源码WWW原则
 * what—— spring现在已经算是java开发的基石了，核心是IOC和AOP.
 * why—— 1.既然作为基石，需要更加彻底的了解并掌握它，工作中可能用不到太细节的东西，但出了问题
    就不好找原因，因此解析源码可以了解一些底层细节；2. Spring作为一个基础框架，可以
    通过阅读源码的方式来入门源码解析的门槛，提升自己的各方面能力；
 * how—— 源码太多有60w行，但不需要全部去从头到尾阅读，也没必要；这里作为个人阅读的
   方法：1. 根据实际使用作为入口，来解析主要功能和必要的细节； 2. 带着问题，例如bean
   的scope，几种方式怎么实现的？spring的xml约束文档DTD和XSD，怎么找到的？等等.
   3. 从简到繁，spring源码包含20+个module，我们需要从最基本的功能开始解析，这里可以
   通过观察各module之间的依赖来了解熟悉.
---
##2. bean的加载、解析和注册
  1. 依赖于spring-core(作为spring的基础工具包).
  2. 容器的基本用法：
     创建一个xml文件，基于XmlBeanFactory(已废弃，官方推荐用DefaultListableBeanFactory+XmlBeanDefinitionReader来替代)去解析，
     从而获得bean.
     ```
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        Resource resource = new ClassPathResource("spring.xml");
        new XmlBeanDefinitionReader(beanFactory).loadBeanDefinitions(resource);
        TestBean testBean = beanFactory.getBean(TestBean.class);
     ```
     Bean的加载过程：加载解析静态配置文件，转换成BeanDefinition，基于BeanDefinition创建bean实例;
  3. Bean的解析和注册
     * 加载资源Resource,继承InputStreamSource,主要负责资源的加载管理，可以通过实现Resource来实现自己的资源加载；
         ```
            public interface Resource extends InputStreamSource {
                boolean exists();
            
                default boolean isReadable() {
                    return true;
                }
            
                default boolean isOpen() {
                    return false;
                }
            
                default boolean isFile() {
                    return false;
                }
            
                URL getURL() throws IOException;
            
                URI getURI() throws IOException;
            
                File getFile() throws IOException;
            
                default ReadableByteChannel readableChannel() throws IOException {
                    return Channels.newChannel(this.getInputStream());
                }
            
                long contentLength() throws IOException;
            
                long lastModified() throws IOException;
            
                Resource createRelative(String var1) throws IOException;
            
                @Nullable
                String getFilename();
            
                String getDescription();
            }
         ```
         一些重要的方法：
         - getInputStream(): 找到并打开资源，返回InputStream从资源中读取的内容；每次调用都会返回一个新的InputStream,
                调用者需要关闭流.
         - exists(): 返回资源是否实际以物理形式存在.
         - isOpen(): 返回一个指示资源是否表示具有打开流的句柄。如果true,IputStream不能多次读取，必须只读一次然后关闭
                以避免资源泄露。false是所有常规资源实现的返回值；这里涉及到jdk1.8的一个新特性：接口中可以有默认方法.
         ---
         一些内置Resource实现：
         + UrlResource: 包装URL并可用于访问通常可通过URL访问的任何对象，例如文件(file:)、http目标(http:)、FTP目标(ftp:)；
         + ClassPathResource: 用于从classpath下加载配置文件，使用线程上下文类加载器、给定的类加载器或者给定的类加载.
         + FileSystemResource: 是java.io.File和java.nio.file.Path处理的实现，支持作为File和RUL解析.
         + ServletContextResource: 作为ServletContext资源的实现，用于web根目录中的相对路径.
         + InputStreamResource: InputStream资源是给定InputStream的资源实现。只有在没有特定的资源实现可用时才应该使用它。特别是，尽可能选择ByteArrayResource或任何基于文件的资源实现。
                                与其他资源实现相比，这是对已打开资源的描述符。因此，它从isOpen()返回true。如果需要将资源描述符保存在某个地方，或者需要多次读取流，请不要使用它。
         + ByteArrayResource:  作为一个给定字符数组的资源实现;
         + ..
     * 利用EncodedResource二次包装Resource，主要是编码格式和字符集的解析,将之前获取的资源包装成InputSource，用于SAX解析配置.
     * 先判断xml的验证模式.(排除掉注释<!-- -->的影响，检查是否包含DOCTYPE字符串，如果有就是DTD，没有就是XSD，报错就是VALIDATION_AUTO)
     * 利用jdk的XML解析出Document.(组装DocumentBuilderFactory，根据factory创建DocumentBuilder，然后解析出Document)
     * 然后就是解析标签注册bean.(以DefaultListableBeanFactory为例，就是放到一个ConcurrentHashMap中)
       ```
         /** Map from dependency type to corresponding autowired value. */
        private final Map<Class<?>, Object> resolvableDependencies = new ConcurrentHashMap<>(16);
     
        /** Map of bean definition objects, keyed by bean name. */
        private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
     
        /** Map of singleton and non-singleton bean names, keyed by dependency type. */
        private final Map<Class<?>, String[]> allBeanNamesByType = new ConcurrentHashMap<>(64);
     
        /** Map of singleton-only bean names, keyed by dependency type. */
        private final Map<Class<?>, String[]> singletonBeanNamesByType = new ConcurrentHashMap<>(64);
     
        /** List of bean definition names, in registration order. */
        private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
     
        /** List of names of manually registered singletons, in registration order. */
        private volatile Set<String> manualSingletonNames = new LinkedHashSet<>(16);
       ```
##3. 部分问题和总结
    1. 以上是bean的加载、解析和注册流程；其中采用了多种设计模式，如工厂设计模式、模板设计模式，这两种设计模式在
       spring中随处可见，可以更好的解耦；核心类是XmlBeanDefinitionReader，bean的解析注册流程也是在这里定义
       的，然后调用各个模板解耦组装；这里可以注入验证模式等参数以实际控制解析注册过程；
    2. 对不同的验证模式,Spring 使用了不同的解析器解析。这里简单描述一下
       原理，比如加载 DTD 类型的 BeansDtdResolver 的 resolveEntity 是直接截取 systemld 最后的 xx.dtd
       然后去当前路径下寻找，而加载载 xsd 类型的 PluggableSchemaResolver 类的 resolveEntity 是默
       认到 META-INF/Spring.schemas 文件巾找到 systemid 所对应的 XSD 文件并加载。两者其实最终的加载
       文件路径都是org.springframework.beans.factory.xml包下.
     
首先我们会想一下，我们在进行SpringMVC配置的时候是怎样配置的(不是web.xml)？我们会在SpringMVC的配置文件中添加这样的一些东西：  
```xml
xmlns:mvc="http://www.springframework.org/schema/mvc"
 xsi:schemaLocation="
        http://www.springframework.org/schema/mvc
		http://www.springframework.org/schema/mvc/spring-mvc-4.3.xsd"
```
然后再配置一下： 
```
<mvc:annotation-driven/>
```
就可以进行简单的Web开发工作了。那么Spring、SpringMVC框架是怎么做的呢？我们注意到在SpringMVC工程的META-INF下面有spring.handlers和spring.schemas这两个文件
![web工程](./img/20180821(1)1.png)  
如果是jar包的话，则目录结构是这样的。  
![jar包](./img/20180821(1)2.png)  
我们可以看看这两个文件中的内容是什么：  
spring.handlers:
```
http\://www.springframework.org/schema/mvc=org.springframework.web.servlet.config.MvcNamespaceHandler
```
spring.schemas:  
```
http\://www.springframework.org/schema/mvc/spring-mvc-3.0.xsd=org/springframework/web/servlet/config/spring-mvc.xsd
http\://www.springframework.org/schema/mvc/spring-mvc-3.1.xsd=org/springframework/web/servlet/config/spring-mvc.xsd
http\://www.springframework.org/schema/mvc/spring-mvc-3.2.xsd=org/springframework/web/servlet/config/spring-mvc.xsd
http\://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd=org/springframework/web/servlet/config/spring-mvc.xsd
http\://www.springframework.org/schema/mvc/spring-mvc-4.1.xsd=org/springframework/web/servlet/config/spring-mvc.xsd
http\://www.springframework.org/schema/mvc/spring-mvc-4.2.xsd=org/springframework/web/servlet/config/spring-mvc.xsd
http\://www.springframework.org/schema/mvc/spring-mvc-4.3.xsd=org/springframework/web/servlet/config/spring-mvc.xsd
http\://www.springframework.org/schema/mvc/spring-mvc.xsd=org/springframework/web/servlet/config/spring-mvc.xsd
```
这两个文件中的key看起来是不是很熟悉？看看文件开头的内容。Spring在对配置文件进行解析的时候会读取META-INF/spring.handlers
这个文件来获取一个org.springframework.beans.factory.xml.NamespaceHandler的实现类。这个解析过程比较复杂，我们先不展开的（想了解的
可以先看一下这个方法：org.springframework.beans.factory.xml.DefaultNamespaceHandlerResolver#resolve），
你只需要记住Spring在解析xml配置文件的时候可以读取到org.springframework.web.servlet.config.MvcNamespaceHandler这个类就行了。
我们先看一下NamespaceHandler这个类的UML类图：    
![NamespaceHandler](./img/20180821(1)3.png)
在上面这张图中我们不仅看到了MvcNamespaceHandler、还有AopNamespaceHandler、TxNamespaceHandler、ContextNamespaceHandler等等(套路都是一样的)。
spring.schemas中的xsd文件描述了xml文档的一些信息，比如有哪些根节点、根节点下面有哪些子节点、有哪些属性信息、数据类型等等。
我们先来看一下MvcNamespaceHandler这个类的内容:      
![MvcNamespaceHandler](./img/20180821(1)4.png)  
```
	@Override
	public void init() {
		//解析<mvc:annotation-driven>标签
		registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
		//解析<mvc:default-servlet-handler>标签
		registerBeanDefinitionParser("default-servlet-handler", new DefaultServletHandlerBeanDefinitionParser());
		//解析<mvc:interceptors>标签
		registerBeanDefinitionParser("interceptors", new InterceptorsBeanDefinitionParser());
		registerBeanDefinitionParser("resources", new ResourcesBeanDefinitionParser());
		registerBeanDefinitionParser("view-controller", new ViewControllerBeanDefinitionParser());
		registerBeanDefinitionParser("redirect-view-controller", new ViewControllerBeanDefinitionParser());
		registerBeanDefinitionParser("status-controller", new ViewControllerBeanDefinitionParser());
		registerBeanDefinitionParser("view-resolvers", new ViewResolversBeanDefinitionParser());
		registerBeanDefinitionParser("tiles-configurer", new TilesConfigurerBeanDefinitionParser());
		registerBeanDefinitionParser("freemarker-configurer", new FreeMarkerConfigurerBeanDefinitionParser());
		registerBeanDefinitionParser("velocity-configurer", new VelocityConfigurerBeanDefinitionParser());
		registerBeanDefinitionParser("groovy-configurer", new GroovyMarkupConfigurerBeanDefinitionParser());
		registerBeanDefinitionParser("script-template-configurer", new ScriptTemplateConfigurerBeanDefinitionParser());
		registerBeanDefinitionParser("cors", new CorsBeanDefinitionParser());
	}
```
在这个方法中注册了很多的BeanDefinitionParser,，而`<mvc:annotation-driven/>`这个标签是通过AnnotationDrivenBeanDefinitionParser这个类来解析的。
这个类中我们要分析的重点就是parse这个方法，在后面的文章中我们会挑一些重点的地方来分析一下这个parse方法中的内容。



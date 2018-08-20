我们在用SpringBoot进行项目开发的时候，基本上都使用过@ConfigurationProperties这个注解，我们在之前的文章中也说过ConfigurationPropertiesBindingPostProcessor会对标注@ConfigurationProperties注解的Bean进行属性值的配置，但是我们之前没有说ConfigurationPropertiesBindingPostProcessor这个Bean是什么时候注入到Spring容器中的。在Spring容器中如果没有这个Bean存在的话，那么通过@ConfigurationProperties注解对Bean属性值的配置行为将无从谈起。我们在本篇文章中将会说明SpringBoot在什么时机会将ConfigurationPropertiesBindingPostProcessor注入到SpringBoot容器中，但它不是我们这篇文章的主角，我们的主角是EnableConfigurationProperties注解。下面我们来请EnableConfigurationProperties登场。
关于EnableConfigurationProperties，在SpringBoot的注释中是这样说明的：为带有@ConfigurationProperties注解的Bean提供有效的支持。这个注解可以提供一种方便的方式来将带有@ConfigurationProperties注解的类注入为Spring容器的Bean。你之前可能听说过类似这样的说法@ConfigurationProperties注解可以生效是因为在SpringBoot中有一个类叫ConfigurationPropertiesAutoConfiguration，它为@ConfigurationProperties注解的解析提供了支持的工作，这种说法更准确一点的说法是在这个类上还存在了@Configuration和@EnableConfigurationProperties这两个注解！我们去ConfigurationPropertiesAutoConfiguration这个类中看一下这个类中的内容：
```java
@Configuration
@EnableConfigurationProperties
public class ConfigurationPropertiesAutoConfiguration {

}
```
我们会发现这个类没有任何内容，它其实只是一个**标识性的类**，用来标识ConfigurationProperties自动配置，重要的就是@Configuration和@EnableConfigurationProperties。这个类在SpringBoot中唯一被引用到的位置是在spring.factories中，
![ConfigurationPropertiesAutoConfiguration ](./img/20180820(1)(1).png)
关于@EnableAutoConfiguration的内容我们先不展开，你现在先记着由于在@EnableAutoConfiguration中存在
```java
@Import(EnableAutoConfigurationImportSelector.class)
```
Import这个注解，这个注解中又引入了EnableAutoConfigurationImportSelector这个类，而这个类会在某一个时机某一个地方(org.springframework.context.annotation.ConfigurationClassParser#processImports)被创建并调用它的selectImports方法，在它的selectImports方法中会获取到ConfigurationPropertiesAutoConfiguration这个类，
![EnableAutoConfigurationImportSelector](./img/20180820(1)(2).png)
而在ConfigurationPropertiesAutoConfiguration这个类上存在@EnableConfigurationProperties这个注解，在EnableConfigurationProperties这个注解中又存在@Import这个注解，在这个注解中又引入了EnableConfigurationPropertiesImportSelector.class这个类，所以这个类同样会在某一个地方(org.springframework.context.annotation.ConfigurationClassParser#processImports)被实例化，然后调用它的selectImports方法。关于@Import这个注解中所引入的类可以分为三种类型，**第一种是实现了ImportSelector接口的类，ImportSelector接口的实现类又可以分为两种，一种是直接实现ImportSelector接口的类，另一种是实现了DeferredImportSelector接口的类，DeferredImportSelector是ImportSelector接口的子类，也只有直接实现了ImportSelector接口的类，才会在ConfigurationClassParser#processImports中被调用它的selectImports方法**；第二种是实现了ImportBeanDefinitionRegistrar接口的类，实现了ImportBeanDefinitionRegistrar接口的类将会调用它的registerBeanDefinitions方法，向Spring容器中注册Bean定义；最后一种是只要不属于上面那两种的都是第三种。
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EnableConfigurationPropertiesImportSelector.class)
public @interface EnableConfigurationProperties {

	/**
	 * Convenient way to quickly register {@link ConfigurationProperties} annotated beans
	 * with Spring. Standard Spring Beans will also be scanned regardless of this value.
	 * @return {@link ConfigurationProperties} annotated beans to register
	 */
	Class<?>[] value() default {};
}
```
我们在上面说了会调用EnableConfigurationPropertiesImportSelector这个类的selectImports方法，我们去看看这个方法的内容：
```java
class EnableConfigurationPropertiesImportSelector implements ImportSelector {

	@Override
	public String[] selectImports(AnnotationMetadata metadata) {
		MultiValueMap<String, Object> attributes = metadata.getAllAnnotationAttributes(
				EnableConfigurationProperties.class.getName(), false);
		Object[] type = attributes == null ? null
				: (Object[]) attributes.getFirst("value");
		if (type == null || type.length == 0) {
			return new String[] {
					ConfigurationPropertiesBindingPostProcessorRegistrar.class
							.getName() };
		}
		return new String[] { ConfigurationPropertiesBeanRegistrar.class.getName(),
				ConfigurationPropertiesBindingPostProcessorRegistrar.class.getName() };
	}
.........
```
在selectImports中会先通过
```java
//false的意思是 不将 获取到的注解中的值转换为String
metadata.getAllAnnotationAttributes(EnableConfigurationProperties.class.getName(), false);
```
来获取类上的EnableConfigurationProperties注解中的value值，注意这里返回的值MultiValueMap<String, Object>，我们用过Map的都了解，通常Map中一个key对应一个value，而MultiValueMap是一个key对应多个value的map(底层是通过List搞定的)。这里为什么用MultiValueMap呢?因为我们的注解中的元数据可能是多个值的(好像一句废话。。。)。
```java
//如果前面没有获取到EnableConfigurationProperties注解的话 则type为null，获取到EnableConfigurationProperties注解的则取value这个元数据的值
Object[] type = attributes == null ? null: (Object[]) attributes.getFirst("value");
```
```java
//如果上一步获取到的type为null或者是一个空数组的话则返回ConfigurationPropertiesBindingPostProcessorRegistrar的全限定类名
if (type == null || type.length == 0) {
	return new String[] {
		ConfigurationPropertiesBindingPostProcessorRegistrar.class.getName() 
	};
}
//如果在上一步中获取到了EnableConfigurationProperties注解中的value元数据的值，则返回ConfigurationPropertiesBeanRegistrar和ConfigurationPropertiesBindingPostProcessorRegistrar这两个类的全限定类名
return new String[] { ConfigurationPropertiesBeanRegistrar.class.getName(),
				ConfigurationPropertiesBindingPostProcessorRegistrar.class.getName() };
```
如果我们的类上只是标注了EnableConfigurationProperties注解，如ConfigurationPropertiesAutoConfiguration这个类的用法。那么就会返回ConfigurationPropertiesBindingPostProcessorRegistrar的全限定类名。按照我们之前说的，这个类将会在某一个地方（org.springframework.context.annotation.ConfigurationClassParser#processImports）被实例化，
```java
public class ConfigurationPropertiesBindingPostProcessorRegistrar
		implements ImportBeanDefinitionRegistrar 
```
请注意这个类是一个实现了ImportBeanDefinitionRegistrar接口的类，所以将会在某一个地方(org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitions)调用它的registerBeanDefinitions方法。我们先去看下这个类中registerBeanDefinitions方法的内容：
```java
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
			BeanDefinitionRegistry registry) {
		//	BINDER_BEAN_NAME 的值是：' ConfigurationPropertiesBindingPostProcessor.class.getName()
		//如果在Spring Bean注册器中=不存在beanBame为org.springframework.boot.context.properties.ConfigurationPropertiesBindingPostProcessor的Bean定义
		if (!registry.containsBeanDefinition(BINDER_BEAN_NAME)) {
			BeanDefinitionBuilder meta = BeanDefinitionBuilder
					.genericBeanDefinition(ConfigurationBeanFactoryMetaData.class);
			//ConfigurationPropertiesBindingPostProcessor bean定义的构造器
			BeanDefinitionBuilder bean = BeanDefinitionBuilder.genericBeanDefinition(
					ConfigurationPropertiesBindingPostProcessor.class);
			bean.addPropertyReference("beanMetaDataStore", METADATA_BEAN_NAME);
			//将beanName为org.springframework.boot.context.properties.ConfigurationPropertiesBindingPostProcessor的bean定义放入到Springbean注册器
			//如果之前看过Spring的源码的话，看到这里你就会会心一笑了。
			//在这里就将我们开篇说的ConfigurationPropertiesBindingPostProcessor注入到Spring容器中了
			registry.registerBeanDefinition(BINDER_BEAN_NAME, bean.getBeanDefinition());
			registry.registerBeanDefinition(METADATA_BEAN_NAME, meta.getBeanDefinition());
		}
	}
```
我们再看如果我们的类上标注了@EnableConfigurationProperties注解，并在注解中value赋值的话，它会返回ConfigurationPropertiesBeanRegistrar 和ConfigurationPropertiesBindingPostProcessorRegistrar这两个类的全限定名，关于ConfigurationPropertiesBindingPostProcessorRegistrar我们已经说过了，下面说ConfigurationPropertiesBeanRegistrar 这个类。首先EnableConfigurationPropertiesImportSelector这个类是EnableConfigurationPropertiesImportSelector中的一个静态内部类，同样的它也实现了ImportBeanDefinitionRegistrar这个接口(ImportBeanDefinitionRegistrar这个接口注意是对注解的Bean定义进行处理)。
```java
public static class ConfigurationPropertiesBeanRegistrar
			implements ImportBeanDefinitionRegistrar
```
```java
		@Override
		public void registerBeanDefinitions(AnnotationMetadata metadata,
				BeanDefinitionRegistry registry) {
			//这里再次从metadata中获取EnableConfigurationProperties注解相关的东西
			//这里的metadata即是我们带有EnableConfigurationProperties类的Bean的元数据
			MultiValueMap<String, Object> attributes = metadata
					.getAllAnnotationAttributes(
							EnableConfigurationProperties.class.getName(), false);
			//这个是用的attributes.get("value")获取value的值 而前面是用的attributes.getFirst("value")
			//将获取到的value转换为List
			List<Class<?>> types = collectClasses(attributes.get("value"));
			for (Class<?> type : types) {
				//这里是获取EnableConfigurationProperties注解中value中的类的ConfigurationProperties的prefix
				//说的太绕了。。。。
				String prefix = extractPrefix(type);
				//如果ConfigurationProperties中的prefix有值的话，则将beanName拼接为prefix+"-"+类的全限定名
				String name = (StringUtils.hasText(prefix) ? prefix + "-" + type.getName()
						: type.getName());
				//如果Spring bean注册器中不存在这个beanName的bean定义的话，则注入到Spring bean注册器中
				if (!registry.containsBeanDefinition(name)) {
					registerBeanDefinition(registry, type, name);
				}
			}
		}
		private void registerBeanDefinition(BeanDefinitionRegistry registry,
				Class<?> type, String name) {
			//bean定义的构造器
			BeanDefinitionBuilder builder = BeanDefinitionBuilder
					.genericBeanDefinition(type);
			//获取bean定义
			AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
			//注入到Spring bean注册器中
			registry.registerBeanDefinition(name, beanDefinition);
			//获取EnableConfigurationProperties value引入的类上的ConfigurationProperties注解
			ConfigurationProperties properties = AnnotationUtils.findAnnotation(type,
					ConfigurationProperties.class);
			//如果EnableConfigurationProperties value引入的类上没有ConfigurationProperties注解的话则抛出异常
			//这个检测为什么不放到方法的开头处呢？？？？
			Assert.notNull(properties,
					"No " + ConfigurationProperties.class.getSimpleName()
							+ " annotation found on  '" + type.getName() + "'.");
		}
```
通过这篇文章你应该了解了ConfigurationPropertiesAutoConfiguration这个类什么会使ConfigurationProperties注解生效了(因为这个类放到了spring.factories中，作为org.springframework.boot.autoconfigure.EnableAutoConfiguration的一个value值，在SpringBoot启动的时候会先被加载，确保能获取到ConfigurationPropertiesAutoConfiguration这个类，并解析它上面的注解！)，也应该明白了@EnableConfigurationProperties这个注解的作用，同时也应该了解了ConfigurationPropertiesBindingPostProcessorRegistrar这个类是怎么被注入到Spring容器中的。下面以一个小例子结束这篇文章：
```java
import com.zkn.springboot.analysis.domain.EnableConfigurationPropertiesDomain;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.stereotype.Component;

/**
 *
 * @author zkn
 * @date 2018/1/28
 */
@Component
@EnableConfigurationProperties(value = EnableConfigurationPropertiesDomain.class)
public class EnableConfigurationPropertiesBean {

}
```
```java
/**
 * @author zkn
 * @date 2018/1/28
 */
//这里只有ConfigurationProperties注解
@ConfigurationProperties(prefix = "configuration.properties")
public class EnableConfigurationPropertiesDomain implements Serializable {

    private static final long serialVersionUID = -3485524455817230192L;
	
    private String name;
	//省略getter setter
    @Override
    public String toString() {
        return "EnableConfigurationPropertiesDomain{" +
                "name='" + name + '\'' +
                '}';
    }
}
```
```java
configuration:
  properties:
    name: This is EnableConfigurationProperties!
```
```java
    @Autowired
    private EnableConfigurationPropertiesDomain propertiesDomain;

    @RequestMapping("index")
    public String index() {
        System.out.println(propertiesDomain);
        return "success";
    }
```
我们通过浏览器访问一下，输出结果如下：
![这里写图片描述](./img/20180820(1)(3).png)
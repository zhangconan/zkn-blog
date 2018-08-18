我们在之前的文章中说了Advisor的创建过程，Advice的创建过程以及为目标类挑选合适的Advisor的过程。通过之前的分析我们知道，SpringAOP将切面类中的通知方法都封装成了一个个的Advisor，这样就统一了拦截方法的调用过程。我们在这一篇文章中说一下SpringAOP中代理对象的创建过程。先看下面的一张图：
![AopProxy](https://img-blog.csdn.net/20180330221910312?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3prbnh4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
在SpringAOP中提供了两种创建代理对象的方式，一种是JDK自带的方式创建代理对象，另一种是使用Cglib的方式创建代理对象。所以在SpringAOP中抽象了一个AopProxy接口，这个接口有两个实现类：JDKDynamicAopProxy和CglibAopProxy。从名字我们应该能看出来这两个类的作用了吧。在为目标类创建代理对象的时候，根据我们的目标类型和AOP的配置信息选择不同的创建代理对象的方式。在SpringAOP中创建代理对象没有直接依赖AopProxy，而是又抽象了一个AopProxyFactory的接口，通过这个接口（工厂模式）来创建代理对象。下面我们来看看具体的创建过程。在开篇的文章中我们写了这样的一段代码来获取代理对象：
```
 //aspectJProxyFactory是AspectJProxyFactory实例
 AspectJService proxyService = aspectJProxyFactory.getProxy();
```
```
 //从这段代码中我们可以看到这里又调用了createAopProxy()的方法
 //通过调用createAopProxy()生成的对象调用getProxy()方法生成代理对象
 public <T> T getProxy() {
		return (T) createAopProxy().getProxy();
}
//这是一个同步方法  ProxyCreatorSupport#createAopProxy
protected final synchronized AopProxy createAopProxy() {
	if (!this.active) {
		//这里会监听调用AdvisedSupportListener实现类的activated方法
		activate();
	}
	//获取AopProxyFactory
	//调用createAopProxy的时候传入了this对象
	return getAopProxyFactory().createAopProxy(this);
}
//在SpringAOP中 AopProxyFactory只有一个实现类，这个实现类就是DefaultAopProxyFactory
public AopProxyFactory getAopProxyFactory() {
	return this.aopProxyFactory;
}	
```
我们先看看一个关于Advised的UML类图：
![Advised](https://img-blog.csdn.net/20180330224725401?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3prbnh4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
从上面的图中我们可以看到AdvisedSupport继承了ProxyConfig类实现了Advised接口。如果你去翻看这两个类的代码的话，会发现在Advised中定义了一些列的方法，而在ProxyConfig中是对这些接口方法的一个实现，但是Advised和ProxyConfig却是互相独立的两个类。但是SpringAOP通过AdvisedSupport将他们适配到了一起。AdvisedSupport只有一个子类，这个子类就是ProxyCreatorSupport。从这两个类的名字我们可以看到他们的作用：一个是为Advised提供支持的类，一个是为代理对象的创建提供支持的类。还记得在Advised中有什么内容吗？
我们去DefaultAopProxyFactory中看一下调用createAopProxy(this)这个方法的时候发生了什么：
```
	//注意 这里传入了一个参数 AdvisedSupport   
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		//这段代码用来判断选择哪种创建代理对象的方式
		//config.isOptimize()   是否对代理类的生成使用策略优化 其作用是和isProxyTargetClass是一样的 默认为false
		//config.isProxyTargetClass() 是否使用Cglib的方式创建代理对象 默认为false
		//hasNoUserSuppliedProxyInterfaces目标类是否有接口存在 且只有一个接口的时候接口类型不是
		//SpringProxy类型 
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			//上面的三个方法有一个为true的话，则进入到这里
			//从AdvisedSupport中获取目标类 类对象
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			//判断目标类是否是接口 如果目标类是接口的话，则还是使用JDK的方式生成代理对象
			//如果目标类是Proxy类型 则还是使用JDK的方式生成代理对象
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			//配置了使用Cglib进行动态代理  或者目标类没有接口 那么使用Cglib的方式创建代理对象
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			//上面的三个方法没有一个为true 那使用JDK的提供的代理方式生成代理对象
			return new JdkDynamicAopProxy(config);
		}
	}
```
我们先看JdkDynamicAopProxy生成代理对象的方法。在上面的代理中创建JdkDynamicAopProxy对象的时候，传入了AdvisedSupport对象。
```
	public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
		Assert.notNull(config, "AdvisedSupport must not be null");
		//如果不存在Advisor或者目标对象为空的话 抛出异常
		if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("No advisors and no TargetSource specified");
		}
		this.advised = config;
	}
```
```
	//这两个方法的区别是否传入类加载器
	@Override
	public Object getProxy() {
		//使用默认的类加载器
		return getProxy(ClassUtils.getDefaultClassLoader());
	}

	@Override
	public Object getProxy(ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		//获取AdvisedSupport类型对象的所有接口
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		//接口是否定义了 equals和hashcode方法 正常是没有的
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		//创建代理对象 this是JdkDynamicAopProxy  
		//JdkDynamicAopProxy 同时实现了InvocationHandler 接口
		//这里我们生成的代理对象可以向上造型为 任意 proxiedInterfaces 中的类型
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
	
	//AopProxyUtils#completeProxiedInterfaces
	//这个方法主要是获取目标类上的接口 并且判断是否需要添加SpringProxy  Advised DecoratingProxy 接口
	static Class<?>[] completeProxiedInterfaces(AdvisedSupport advised, boolean decoratingProxy) {
		//获取AdvisedSupport 类型中  目标类的接口
		Class<?>[] specifiedInterfaces = advised.getProxiedInterfaces();
		//如果目标类没有实现接口的话，
		if (specifiedInterfaces.length == 0) {
			// No user-specified interfaces: check whether target class is an interface.
			//获取目标类
			Class<?> targetClass = advised.getTargetClass();
			if (targetClass != null) {
				//如果目标类是接口 则把目标类添加到 AdvisedSupport的接口集合中
				if (targetClass.isInterface()) {
					advised.setInterfaces(targetClass);
				}
				//如果是Proxy类型
				else if (Proxy.isProxyClass(targetClass)) {
					advised.setInterfaces(targetClass.getInterfaces());
				}
				//重新获取接口
				specifiedInterfaces = advised.getProxiedInterfaces();
			}
		}
		//接口中有没有 SpringProxy类型的接口
		//是否需要添加 SpringProxy接口
		boolean addSpringProxy = !advised.isInterfaceProxied(SpringProxy.class);
		//isOpaque 代表生成的代理是否避免转化为Advised类型 默认为false  如果目标类没有实现 Advised接口
		//是否需要添加 Advised接口
		boolean addAdvised = !advised.isOpaque() && !advised.isInterfaceProxied(Advised.class);
		//是否需要添加 DecoratingProxy接口
		boolean addDecoratingProxy = (decoratingProxy && !advised.isInterfaceProxied(DecoratingProxy.class));
		int nonUserIfcCount = 0;
		if (addSpringProxy) {
			nonUserIfcCount++;
		}
		if (addAdvised) {
			nonUserIfcCount++;
		}
		if (addDecoratingProxy) {
			nonUserIfcCount++;
		}
		Class<?>[] proxiedInterfaces = new Class<?>[specifiedInterfaces.length + nonUserIfcCount];
		//扩展接口数组
		System.arraycopy(specifiedInterfaces, 0, proxiedInterfaces, 0, specifiedInterfaces.length);
		int index = specifiedInterfaces.length;
		if (addSpringProxy) {
			//为目标对象接口中添加 SpringProxy接口
			proxiedInterfaces[index] = SpringProxy.class;
			index++;
		}
		if (addAdvised) {
			//为目标对象接口中添加 Advised接口
			proxiedInterfaces[index] = Advised.class;
			index++;
		}
		if (addDecoratingProxy) {
			//为目标对象接口中添加 DecoratingProxy接口
			proxiedInterfaces[index] = DecoratingProxy.class;
		}
		return proxiedInterfaces;
	}
```
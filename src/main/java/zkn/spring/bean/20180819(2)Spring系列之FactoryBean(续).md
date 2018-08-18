我们在上一篇文章中说了一下FactoryBean类型的Bean的getObjectType方法被使用到的一个地方，我们在这一篇文章中会说一下FactoryBean是怎么让Spring容器管理调用它的getObject所生成的Bean的。在这篇文章中我们从getBean方法开始说起(我们这里还是要说一下我们现在的beanName的值为：factoryBeanLearn，Class类型为：FactoryBeanService.class )：
getBean(beanName, requiredType, args)方法，这个方法又调用了doGetBean方法，doGetBean可以说是Spring容器中一个很核心的一个类，里面的功能很多很复杂，我们在这篇文章中只关注和FactoryBean相关的内容。思路是：先分析大内容里面的具体的点，再由点及面的分析。
```
public <T> T getBean(String name, Class<T> requiredType, Object... args) throws BeansException {
	return doGetBean(name, requiredType, args, false);
}
protected <T> T doGetBean(
	final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
		throws BeansException {
	//转换传入的BeanName的值，如&name变为name 以及别名(alias)的转换
	final String beanName = transformedBeanName(name);
	Object bean;
	//调用getSingleton方法 从Spring容器中获取单例Bean 具体的获取过程见下面分析
	//我们在上一篇文章中分析过 这里的beanName为factoryBeanLearn 
	//beanName为factoryBeanLearn的Bean 已经在Spring容器中创建过了(创建过程见AbstractApplicationContext的refresh方法)
	//所以这里会获取到一个FactoryBeanLearn实例
	//这里根据beanName获取bean实例的方法 beanName都是经过处理之后的beanName
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		//这个方法是从实例自身获取对象 像这个实例是一个FactoryBean类型的实例
		//我们在下面分析这个方法
		//这里要注意的是：我们在这个方法中传入了一个name 又传入了一个beanName
		//这里为什么要传入两个beanName？可以想一下
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}
```
getSingleton从Spring容器中获取单例Bean
```
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		//先从singletonObjects中获取单例Bean singletonObjects是一个ConcurrentHashMap
		//key是beanName value是单例Bean
		Object singletonObject = this.singletonObjects.get(beanName);
		//如果没有获取到，则判断是不是当前在创建中的单例Bean
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			//这里加锁
			synchronized (this.singletonObjects) {
				//提前暴露创建的Bean中是否存在beanName的单例Bean
				singletonObject = this.earlySingletonObjects.get(beanName);
				//如果没有获取到 并且允许提前引用响应的Bean
				if (singletonObject == null && allowEarlyReference) {
					//singletonFactories是一个HashMap key是beanName，value是ObjectFactory 又一个Factory
					//在Spring中很多地方用到了ObjectFactory  包括我们在之前的文章中分析的 Autowired Request
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						//从ObjectFactory中获取Bean实例
						singletonObject = singletonFactory.getObject();
						//放入earlySingletonObjects这个Map中
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
```
getObjectForBeanInstance的分析
```
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

		//这里判断 name是不是以&开头，不是经过处理的beanName  并且这个bean实例 不是FactoryBean类型的
		//如果是&开头 并且不是FactoryBean类型 则抛出异常
		if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}
		
		//不是FactoryBean类型 或者name以&开头 直接返回bean实例
		//想一下我们关于FactoryBean的知识：如果要根据beanName获取真正的FactoryBean实例的时候
		//需要在beanName前面加上&  这里就可以看到为什么要这样做了。
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd == null) {
			//factoryBeanObjectCache 看看是不是在缓存中存在
			//factoryBeanObjectCache
			object = getCachedObjectForFactoryBean(beanName);
		}
		//如果没有 
		if (object == null) {
			//如果能走到这里来 这个bean实例是FactoryBean类型的
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			//从这个方法的名字我们可以看到这个方法的意思是:从FactoryBean中
			//获取对象
			//getObjectFromFactoryBean的分析在下面 
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```
getObjectFromFactoryBean的分析
```
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		//FactoryBean类型的实例 调用isSingleton方法返回的是true 
		//所传入的bean实例也要求是单例类型的
		//对应我们这里就是FactoryBeanLearn中的方法返回true 
		if (factory.isSingleton() && containsSingleton(beanName)) {
			//加锁
			synchronized (getSingletonMutex()) {
				//再从缓存中获取一次
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					//调用doGetObjectFromFactoryBean方法从FactoryBean中获取bean对象
					//这里是调用的FactoryBean的getObject方法来获取的
					object = doGetObjectFromFactoryBean(factory, beanName);
					//再从缓存中获取一次
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					//如果上一步的缓存中获取到了 则用缓存中的替代我们
					//我们从FactoryBean中获取的bean
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
						if (object != null && shouldPostProcess) {
							try {
								//调用BeanPostProcessor中的postProcessAfterInitialization方法进行处理
								//这里只会调用BeanPostProcessor的postProcessAfterInitialization方法了
								//
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							//抛出异常
						}
						//放入到factoryBeanObjectCache中缓存起来  key为beanName
						this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
					}
				}
				return (object != NULL_OBJECT ? object : null);
			}
		}
		else {
			//非单例
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (object != null && shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}
```
doGetObjectFromFactoryBean的分析
```
	private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
			throws BeanCreationException {

		Object object;
		try {
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					object = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
						@Override
						public Object run() throws Exception {
								//调用FactoryBean中的getObject()方法获取bean
								return factory.getObject();
							}
						}, acc);
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				//看这里 调用FactoryBean中的getObject()方法获取bean
				object = factory.getObject();
			}
		}
		if (object == null && isSingletonCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(
					beanName, "FactoryBean which is currently in creation returned null from getObject");
		}
		return object;
	}
```
整个流程图大致如下:
![FactoryBean](./img/2018-08-19(2)FactoryBean2.jpg)
流程简化起来就是：
	循环Spring容器中所有的beanNames，再根据beanName获取对应的Bean实例，判断获取的Bean实例是不是FactoryBean类型的Bean，如果是，则调用Bean的getObjectType方法获取Class，将获取到的Class和传入的Class进行匹配，如果匹配到，则将此beanName和传入的Class建立一个映射关系。再根据beanName获取到Spring容器中对应的Bean，调用Bean的getObject方法来获取对应的实例。
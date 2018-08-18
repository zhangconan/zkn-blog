我们在上一篇文章中简单的说了一下SpringAOP使用JDK动态代理生成目标对象的过程，我们在这一篇文章中说一下SpringAOP对生成的动态代理对象的方法的拦截过程(即SpringAOP拦截过程)，这个分析的过程可能会比较长。
在上一篇文章中我们说的使用JDK创建动态代理对象是用的JdkDynamicAopProxy这个类，这个类同时实现了InvocationHandler这个接口，实现了它的invoke方法，熟悉JDK动态代理的同学都知道，当我们调用动态代理对象的方法的时候，会进入到生成代理对象时所传入的InvocationHandler实现类的invoke方法中，在这里也就是指JdkDynamicAopProxy的invoke方法，我们进入到这个invoke方法中看一下：
```java
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;
		//目标对象
		TargetSource targetSource = this.advised.targetSource;
		Class<?> targetClass = null;
		Object target = null;

		try {
			//接口中没有定义 equals方法(这个方法定义形式和Object中保持一致 )，并且调用的方法是equals方法(即Object中定义的equals方法) 
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				//调用 JdkDynamicAopProxy 中写的equals方法
				return equals(args[0]);
			}else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				//和上面的分析一样
				return hashCode();
			}else if (method.getDeclaringClass() == DecoratingProxy.class) {
				//没用过 留作以后再说
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}else if (!this.advised.opaque && method.getDeclaringClass().isInterface() && method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			  //如果 方法所在的类是接口 并且是Advised的子类，则直接调用下面的方法，这个方法在下面分析 
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;
			//是否对外暴露代理对象
			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				//把之前创建的代理对象放到线程上下文中
				//oldProxy为之前线程上下文中的对象
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}
			//从TargetSource中获取目标对象
			target = targetSource.getTarget();
			if (target != null) {
				targetClass = target.getClass();
			}
			//从Advised中根据方法名和目标类获取 AOP拦截器执行链 重点要分析的内容
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
			//如果这个执行链为空的话
			if (chain.isEmpty()) {
				//直接进行方法调用
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				//如果AOP拦截器执行链不为空  说明有AOP通知存在
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				//开始调用
				retVal = invocation.proceed();
			}
			//方法的返回值类型
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				//return this
				retVal = proxy;
			}
			//返回值类型错误
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			//如果目标对象不为空  且目标对象是可变的 如prototype类型
			//通常我们的目标对象都是单例的  即targetSource.isStatic为true
			if (target != null && !targetSource.isStatic()) {
				//释放目标对象
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				//线程上下文复位
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}
```
在上面的方法中大致了说明了一下整个代理对象方法调用的执行过程，像equals和hashcode方法的调用，Advised子类的调用。重点就是在连接点处执行不同的通知类型的调用，即获取AOP拦截执行链的调用。下面我们要分析的就是这个过程：
this.advised.getInterceptorsAndDynamicInterceptionAdvice。
我们这里的advised是一个AdvisedSupport类型的实例，它可能是ProxyFactory的实例也可能是AspectJProxyFactory实例。我们进入到getInterceptorsAndDynamicInterceptionAdvice这个方法中去看一下：
```java
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class<?> targetClass) {
		//创建一个method的缓存对象 在MethodCacheKey中实现了equals和hashcode方法同时还实现了compareTo方法
		MethodCacheKey cacheKey = new MethodCacheKey(method);
		List<Object> cached = this.methodCache.get(cacheKey);
		//先从缓存中获取 如果缓存中获取不到 则再调用方法获取，获取之后放入到缓存中
		if (cached == null) {
			//调用的是advisorChainFactory的getInterceptorsAndDynamicInterceptionAdvice方法
			cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
					this, method, targetClass);
			this.methodCache.put(cacheKey, cached);
		}
		return cached;
	}
```
上面的方法的调用过程是先从缓存中获取，缓存中获取不到的话，再交给AdvisorChainFactory，通过调用AdvisorChainFactory中的getInterceptorsAndDynamicInterceptionAdvice方法来获取拦截器执行链，放入到缓存中。AdvisorChainFactory在SpringAOP中只有一个默认的实现类：DefaultAdvisorChainFactory，所以我们去这个DefaultAdvisorChainFactory类中看一下这个方法的内容。
```java
	//在这个方法中传入了三个实例，一个是Advised的实例 一个是目标方法 一个是目标类可能为null
	//想想我们在前面的文章中说过的，在Advised中都有什么内容
	@Override
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, Class<?> targetClass) {
		//创建一个初始大小为 之前获取到的 通知个数的 集合
		List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
		//如果目标类为null的话，则从方法签名中获取目标类
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		//判断目标类是否存在引介增强 通常为false
		boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
		//这里用了一个单例模式 获取DefaultAdvisorAdapterRegistry实例
		//在Spring中把每一个功能都分的很细，每个功能都会有相应的类去处理 符合单一职责原则的地方很多 这也是值得我们借鉴的一个地方
		//AdvisorAdapterRegistry这个类的主要作用是将Advice适配为Advisor 将Advisor适配为对应的MethodInterceptor 我们在下面说明
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
		//循环 目标方法匹配的 通知
		for (Advisor advisor : config.getAdvisors()) {
			//如果是PointcutAdvisor类型的实例  我们大多数的Advisor都是PointcutAdvisor类型的
			if (advisor instanceof PointcutAdvisor) {
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				//如果提前进行过 切点的匹配了  或者当前的Advisor适用于目标类
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					//将Advisor适配为MethodInterceptor
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					//检测Advisor是否适用于此目标方法
					if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
						//MethodMatcher中的切点分为两种 一个是静态的 一种是动态的
						//如果isRuntime返回true 则是动态的切入点 每次方法的调用都要去进行匹配
						//而静态切入点则回缓存之前的匹配结果值 
						if (mm.isRuntime()) {
							//动态切入点 则会创建一个InterceptorAndDynamicMethodMatcher对象
							//这个对象包含MethodInterceptor和MethodMatcher 的实例
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							//添加到列表中
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			//如果是引介增强
			else if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
					//将Advisor转换为Interceptor
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			//以上两种都不是
			else {
				//将Advisor转换为Interceptor
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}
		return interceptorList;
	}
```
在上面这个方法中主要干了这几件事：
1、循环目标方法的所有Advisor
2、判断Advisor的类型
如果是PointcutAdvisor的类型，则判断此Advisor是否适用于此目标方法
如果是IntroductionAdvisor引介增强类型，则判断此Advisor是否适用于此目标方法
如果以上都不是，则直接转换为Interceptor类型。
在上面的三个步骤中都干了这样的一件事，**将Advisor转换为Interceptor类型**。这里用到了一个很重要的一个类：DefaultAdvisorAdapterRegistry。从类名我们可以看出这是一个Advisor的适配器注册类。它的源码如下：
```java
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {
	//初始化了一个size为3的集合 因为下面就添加了三个AdvisorAdapter
	private final List<AdvisorAdapter> adapters = new ArrayList<AdvisorAdapter>(3);
	/**
	 * Create a new DefaultAdvisorAdapterRegistry, registering well-known adapters.
	 */
	public DefaultAdvisorAdapterRegistry() {
		//在SpringAOP中只默认提供了这三种通知类型的适配器
		//为什么没有其他通知类型的呢？参考AbstractAspectJAdvice下面的几个通知类型
		//前置通知适配器
		registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
		//后置返回通知适配器
		registerAdvisorAdapter(new AfterReturningAdviceAdapter());
		//后置异常通知适配器
		registerAdvisorAdapter(new ThrowsAdviceAdapter());
	}
	//这个方法的作用主要是将Advice转换为Advisor的
	@Override
	public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
		//如果传入的实例是Advisor 则直接返回
		if (adviceObject instanceof Advisor) {
			return (Advisor) adviceObject;
		}
		//如果传入的实例不是 Advice类型 则直接抛出异常
		if (!(adviceObject instanceof Advice)) {
			throw new UnknownAdviceTypeException(adviceObject);
		}
		Advice advice = (Advice) adviceObject;
		//如果这个Advice是MethodInterceptor类型的实例，则直接包装为DefaultPointcutAdvisor
		//DefaultPointcutAdvisor中的Pointcut为Pointcut.TRUE matches始终返回true
		if (advice instanceof MethodInterceptor) {
			return new DefaultPointcutAdvisor(advice);
		}
		//如果不是Advisor的实例 也不是MethodInterceptor类型的实例
		//看看是不是 上面的那种通知类型适配器所支持的类型
		for (AdvisorAdapter adapter : this.adapters) {
			// Check that it is supported.
			if (adapter.supportsAdvice(advice)) {
				return new DefaultPointcutAdvisor(advice);
			}
		}
		throw new UnknownAdviceTypeException(advice);
	}
	//这个方法是将 Advisor转换为 MethodInterceptor
	@Override
	public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
		List<MethodInterceptor> interceptors = new ArrayList<MethodInterceptor>(3);
		//从Advisor中获取 Advice
		Advice advice = advisor.getAdvice();
		if (advice instanceof MethodInterceptor) {
			interceptors.add((MethodInterceptor) advice);
		}
		for (AdvisorAdapter adapter : this.adapters) {
			if (adapter.supportsAdvice(advice)) {
				//转换为对应的 MethodInterceptor类型
				//AfterReturningAdviceInterceptor MethodBeforeAdviceInterceptor  ThrowsAdviceInterceptor
				interceptors.add(adapter.getInterceptor(advisor));
			}
		}
		if (interceptors.isEmpty()) {
			throw new UnknownAdviceTypeException(advisor.getAdvice());
		}
		return interceptors.toArray(new MethodInterceptor[interceptors.size()]);
	}
	//新增的 Advisor适配器
	@Override
	public void registerAdvisorAdapter(AdvisorAdapter adapter) {
		this.adapters.add(adapter);
	}
}
```
所以this.advised.getInterceptorsAndDynamicInterceptionAdvice这个方法获取的是目标方法的AOP拦截器执行链。

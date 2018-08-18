我们在上一篇文章中简单的说了调用动态代理对象方法的过程，也说了AOP拦截器执行链的生成过程。我们接着说AOP对目标对象的拦截过程。下面的代码是我们要分析的重点：
```
//proxy:生成的动态代理对象
//target:目标对象
//method:目标方法
//args:目标方法参数
//targetClass:目标类对象
//chain: AOP拦截器执行链  是一个MethodInterceptor的集合 这个链条的获取过程参考我们上一篇文章的内容
invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
//开始执行AOP的拦截过程
retVal = invocation.proceed();

//构造ReflectiveMethodInvocation对象
protected ReflectiveMethodInvocation(
			Object proxy, Object target, Method method, Object[] arguments,
			Class<?> targetClass, List<Object> interceptorsAndDynamicMethodMatchers) {

		this.proxy = proxy;
		this.target = target;
		this.targetClass = targetClass;
		//桥接方法
		this.method = BridgeMethodResolver.findBridgedMethod(method);
		//转换参数
		this.arguments = AopProxyUtils.adaptArgumentsIfNecessary(method, arguments);
		this.interceptorsAndDynamicMethodMatchers = interceptorsAndDynamicMethodMatchers;
	}
```
```
	public Object proceed() throws Throwable {
		//	currentInterceptorIndex初始值为 -1 
		// 如果执行到链条的末尾 则直接调用连接点方法 即 直接调用目标方法
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			//下面会分析
			return invokeJoinpoint();
		}
		//获取集合中的 MethodInterceptor
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		//如果是InterceptorAndDynamicMethodMatcher类型
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			//这里每一次都去匹配是否适用于这个目标方法
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				//如果匹配则直接调用 MethodInterceptor的invoke方法
				//注意这里传入的参数是this 我们下面看一下 ReflectiveMethodInvocation的类型
				return dm.interceptor.invoke(this);
			}
			else {
				//如果不适用于此目标方法  则继续执行下一个链条 
				//递归调用
				return proceed();
			}
		}
		else {
			//说明是适用于此目标方法的 直接调用 MethodInterceptor的invoke方法  传入this即ReflectiveMethodInvocation实例
			//传入this进入 这样就可以形成一个调用的链条了
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```
invokeJoinpoint方法
```
	protected Object invokeJoinpoint() throws Throwable {
		//this.target 目标对象
		//this.method 目标方法
		this.arguments 目标方法参数信息
		return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
	}
	public static Object invokeJoinpointUsingReflection(Object target, Method method, Object[] args)
			throws Throwable {

		// Use reflection to invoke the method.
		try {
			//设置方法可见性
			ReflectionUtils.makeAccessible(method);
			//反射调用  最终是通过反射去调用目标方法
			return method.invoke(target, args);
		}
		catch (InvocationTargetException ex) {
			// Invoked method threw a checked exception.
			// We must rethrow it. The client won't see the interceptor.
			throw ex.getTargetException();
		}
		catch (IllegalArgumentException ex) {
			throw new AopInvocationException("AOP configuration seems to be invalid: tried calling method [" +
					method + "] on target [" + target + "]", ex);
		}
		catch (IllegalAccessException ex) {
			throw new AopInvocationException("Could not access method [" + method + "]", ex);
		}
	}
```
ReflectiveMethodInvocation的UML类图如下：
![ReflectiveMethodInvocation](https://img-blog.csdn.net/20180425234212403?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3prbnh4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
MethodInterceptorChain
![MethodInterceptorChain](https://img-blog.csdn.net/20180425235650418?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3prbnh4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
OK，我们在proceed()这个方法中看到了AOP对于目标方法的一个拦截的过程，其中很重要的一个点是调用MethodInterceptor的invoke方法。我们先看一下MethodInterceptor的主要UML类图(由于我们在开发中使用AspectJ注解的方式越来越多，所以我们这里说的基本上都是基于AspectJ注解的)：
![MethodInterceptor](https://img-blog.csdn.net/20180426235354889?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3prbnh4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
从上图我们也可以看到不同的通知其实相当于不同的MethodInterceptor类型。像前置通知会交给：MethodBeforeAdviceInterceptor来进行处理，后置通知是由AspectJAfterAdvice来处理的，环绕通知是由AspectJAroundAdvice来处理的。我们也挑几个通知类型来说一下具体的调用过程。先说一下前置通知：
MethodBeforeAdviceInterceptor
```
//实现了MethodInterceptor接口
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {
	//这个对象的获取参考这个方法org.springframework.aop.aspectj.annotation.ReflectiveAspectJAdvisorFactory#getAdvice
	//在之前的文章中有说过 不再描述
	//这个MethodBeforeAdvice是AspectJMethodBeforeAdvice实例
	private MethodBeforeAdvice advice;
	
	public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}
	
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		//这里就会执行  前置通知的逻辑 这里的advice是 AspectJMethodBeforeAdvice
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
		//这里传入的MethodInvocation是ReflectiveMethodInvocation对象，即前面说的  传入this
		//相当于ReflectiveMethodInvocation.proceed() 递归调用。
		return mi.proceed();
	}
}
```
AspectJMethodBeforeAdvice#before代码如下：
```
	@Override
	public void before(Method method, Object[] args, Object target) throws Throwable {
		//这里传进来的目标对象、目标参数、目标方法都没有用到
		invokeAdviceMethod(getJoinPointMatch(), null, null);
	}
	protected JoinPointMatch getJoinPointMatch() {
		//这里从线程上下文中获取MethodInvocation 看到这里你也许会感到奇怪  跟着文章分析来看 我们没有在设置过上下文的值啊
		//这里是怎么获取到MethodInvocation 的对象的呢？
		//不知道你是否还记得 我们在获取Advisor的时候 调用过这样的一个方法org.springframework.aop.aspectj.AspectJProxyUtils#makeAdvisorChainAspectJCapableIfNecessary
		//在这个方法中会有这样的一段代码 
		// if (foundAspectJAdvice && //!advisors.contains(ExposeInvocationInterceptor.ADVISOR)) {
		//     如果获取到了 Advisor  则向 Advisor集合中添加第一个元素 即 ExposeInvocationInterceptor.ADVISOR
		//也就是说 我们的 Advisor列表中的第一个元素为ExposeInvocationInterceptor.ADVISOR 它是一个DefaultPointcutAdvisor的实例
		// 对于任何的目标方法都返回true  它的Advice是ExposeInvocationInterceptor
		//		advisors.add(0, ExposeInvocationInterceptor.ADVISOR);
		//		return true;
		//	}
		// 这样我们就不难理解了，在调用ReflectiveMethodInvocation#proceed的时候第一个调用的MethodInterceptor是ExposeInvocationInterceptor
		//ExposeInvocationInterceptor的invoke方法的内容如下：
		//	public Object invoke(MethodInvocation mi) throws Throwable {
		//   先取出旧的MethodInvocation的值
		//    MethodInvocation oldInvocation = invocation.get();
		//    这里设置新的 MethodInvocation 就是这里了！！！
		//    invocation.set(mi);
		//    try {
		//      递归调用
		//	    return mi.proceed();
		///   }
		//   finally {
		//	   invocation.set(oldInvocation);
		//   }
	//      }
		MethodInvocation mi = ExposeInvocationInterceptor.currentInvocation();
		if (!(mi instanceof ProxyMethodInvocation)) {
			throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
		}
		//这里主要是获取 JoinPointMatch
		return getJoinPointMatch((ProxyMethodInvocation) mi);
	}
```
关于invokeAdviceMethod方法的内容我们在下一篇文章中继续分析
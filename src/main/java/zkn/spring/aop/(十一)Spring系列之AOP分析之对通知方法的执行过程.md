转载请注明出处：https://blog.csdn.net/zknxx/article/details/80261327
我们在上一篇文章中说到了前置通知的方法调用AspectJMethodBeforeAdvice#before，在这个before方法中又调用了invokeAdviceMethod这个方法，invokeAdviceMethod这个方法在AspectJMethodBeforeAdvice的父类AbstractAspectJAdvice中。AbstractAspectJAdvice这个是Aspect的所有通知类型的共同父类。关于AbstractAspectJAdvice中的invokeAdviceMethod方法，有两个重载的方法。前置通知、后置通知、异常通知、后置返回通知都是用的AbstractAspectJAdvice#invokeAdviceMethod(org.aspectj.weaver.tools.JoinPointMatch, java.lang.Object, java.lang.Throwable)这个方法，环绕通知用的是：AbstractAspectJAdvice#invokeAdviceMethod(org.aspectj.lang.JoinPoint, org.aspectj.weaver.tools.JoinPointMatch, java.lang.Object, java.lang.Throwable)这个方法。这两个重载方法的区别是：后置通知调用的方法多了一个JoinPoint的参数。
invokeAdviceMethod方法的源码如下：
```
	//这三个参数 JoinPointMatch 都是相同的
	//returnValue 当执行后置返回通知的时候 传值 其他为null
	//Throwable  当执行后置异常通知的时候 传值，其他为null
	protected Object invokeAdviceMethod(JoinPointMatch jpMatch, Object returnValue, Throwable ex) throws Throwable {
		return invokeAdviceMethodWithGivenArgs(argBinding(getJoinPoint(), jpMatch, returnValue, ex));
	}
	//重载的方法 这个 JoinPoint 是ProceedingJoinPoint
	protected Object invokeAdviceMethod(JoinPoint jp, JoinPointMatch jpMatch, Object returnValue, Throwable t)
			throws Throwable {

		return invokeAdviceMethodWithGivenArgs(argBinding(jp, jpMatch, returnValue, t));
	}
```
我们先看getJoinPoint这个方法，其源码如下：
```
	protected JoinPoint getJoinPoint() {
		return currentJoinPoint();
	}
	public static JoinPoint currentJoinPoint() {
		//这里就不用再多说了  获取当前的MethodInvocation 即ReflectiveMethodInvocation的实例
		MethodInvocation mi = ExposeInvocationInterceptor.currentInvocation();
		if (!(mi instanceof ProxyMethodInvocation)) {
			throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
		}
		ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
		//JOIN_POINT_KEY 的值 为 JoinPoint.class.getName()
		//从ReflectiveMethodInvocation中获取JoinPoint 的值
		//这里在第一次获取的时候 获取到的 JoinPoint是null 
		//然后把下面创建的MethodInvocationProceedingJoinPoint放入到ReflectiveMethodInvocation的userAttributes中
		//这样在第二次获取的是 就会获取到这个 MethodInvocationProceedingJoinPoint
		JoinPoint jp = (JoinPoint) pmi.getUserAttribute(JOIN_POINT_KEY);
		if (jp == null) {
			jp = new MethodInvocationProceedingJoinPoint(pmi);
			pmi.setUserAttribute(JOIN_POINT_KEY, jp);
		}
		return jp;
	}
```
下面我们来看一下argBinding这个方法的作用和内容。从名字我们可以猜测这个方法的作用应该是进行参数绑定用的，我们来看一下：
```
	protected Object[] argBinding(JoinPoint jp, JoinPointMatch jpMatch, Object returnValue, Throwable ex) {
		calculateArgumentBindings();

		// AMC start
		Object[] adviceInvocationArgs = new Object[this.parameterTypes.length];
		int numBound = 0;
		//这个默认值是 -1 重新赋值是在calculateArgumentBindings中进行的
		if (this.joinPointArgumentIndex != -1) {
			adviceInvocationArgs[this.joinPointArgumentIndex] = jp;
			numBound++;
		}
		else if (this.joinPointStaticPartArgumentIndex != -1) {
			adviceInvocationArgs[this.joinPointStaticPartArgumentIndex] = jp.getStaticPart();
			numBound++;
		}
		//这里主要是取通知方法中的参数类型 是除了 JoinPoint和ProceedingJoinPoint参数之外的参数
		//如异常通知参数 返回通知参数
		if (!CollectionUtils.isEmpty(this.argumentBindings)) {
			// binding from pointcut match
			if (jpMatch != null) {
				PointcutParameter[] parameterBindings = jpMatch.getParameterBindings();
				for (PointcutParameter parameter : parameterBindings) {
					String name = parameter.getName();
					Integer index = this.argumentBindings.get(name);
					adviceInvocationArgs[index] = parameter.getBinding();
					numBound++;
				}
			}
			// binding from returning clause
			//后置返回通知参数
			if (this.returningName != null) {
				Integer index = this.argumentBindings.get(this.returningName);
				adviceInvocationArgs[index] = returnValue;
				numBound++;
			}
			// binding from thrown exception
			//异常通知参数
			if (this.throwingName != null) {
				Integer index = this.argumentBindings.get(this.throwingName);
				adviceInvocationArgs[index] = ex;
				numBound++;
			}
		}
		if (numBound != this.parameterTypes.length) {
			throw new IllegalStateException("Required to bind " + this.parameterTypes.length +
					" arguments, but only bound " + numBound + " (JoinPointMatch " +
					(jpMatch == null ? "was NOT" : "WAS") + " bound in invocation)");
		}
		return adviceInvocationArgs;
	}
```
calculateArgumentBindings
```
	public synchronized final void calculateArgumentBindings() {
		// The simple case... nothing to bind.
		//通知方法没有参数直接返回
		if (this.argumentsIntrospected || this.parameterTypes.length == 0) {
			return;
		}
		
		int numUnboundArgs = this.parameterTypes.length;
		Class<?>[] parameterTypes = this.aspectJAdviceMethod.getParameterTypes();
		//从这里可以看出来我们的JoinPoint和ProceedingJoinPoint要放在通知方法的第一个参数
		if (maybeBindJoinPoint(parameterTypes[0]) || maybeBindProceedingJoinPoint(parameterTypes[0])) {
			numUnboundArgs--;
		}
		else if (maybeBindJoinPointStaticPart(parameterTypes[0])) {
			numUnboundArgs--;
		}
		//这里是对其他的参数的处理  处理过程还是比较复杂一点的 这里不再多说了。
		if (numUnboundArgs > 0) {
			// need to bind arguments by name as returned from the pointcut match
			bindArgumentsByName(numUnboundArgs);
		}
		this.argumentsIntrospected = true;
	}
```
这里还有再说一下AbstractAspectJAdvice这个类的构造函数，这个类只有这一个构造函数
```
	public AbstractAspectJAdvice(
			Method aspectJAdviceMethod, AspectJExpressionPointcut pointcut, AspectInstanceFactory aspectInstanceFactory) {
		//通知方法不能为空
		Assert.notNull(aspectJAdviceMethod, "Advice method must not be null");
		//切面类
		this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
		//通知方法的名字
		this.methodName = aspectJAdviceMethod.getName();
		//通知方法参数
		this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
		//通知方法
		this.aspectJAdviceMethod = aspectJAdviceMethod;
		//切点类
		this.pointcut = pointcut;
		//切面实例的工厂类
		this.aspectInstanceFactory = aspectInstanceFactory;
	}
```
在创建通知类实例的时候，进行了上面的赋值的动作，把和通知相关的方法都传了进来。最后我们来看一下invokeAdviceMethodWithGivenArgs这个方法的内容：
```
	protected Object invokeAdviceMethodWithGivenArgs(Object[] args) throws Throwable {
		Object[] actualArgs = args;
		//判断通知方法是否有参数
		if (this.aspectJAdviceMethod.getParameterTypes().length == 0) {
			actualArgs = null;
		}
		try {
			ReflectionUtils.makeAccessible(this.aspectJAdviceMethod);
			// TODO AopUtils.invokeJoinpointUsingReflection
			//反射调用通知方法
			//this.aspectInstanceFactory.getAspectInstance()获取的是切面的实例
			return this.aspectJAdviceMethod.invoke(this.aspectInstanceFactory.getAspectInstance(), actualArgs);
		}
		catch (IllegalArgumentException ex) {
			throw new AopInvocationException("Mismatch on arguments to advice method [" +
					this.aspectJAdviceMethod + "]; pointcut expression [" +
					this.pointcut.getPointcutExpression() + "]", ex);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}
```
转载请注明出处：https://blog.csdn.net/zknxx/article/details/80261327
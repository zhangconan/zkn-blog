我们在前面的文章中分析了从切面类中获取Advisor的过程，我们最后创建的Advisor实例为：InstantiationModelAwarePointcutAdvisorImpl，它是一个Advisor和PointcutAdvisor的实现类，所以我们可以从这个类中获取Advice和Pointcut。从之前的分析中我们也看到了Pointcut的赋值，在这一篇文章中我们将会具体分析Advice的创建过程。
我们在上一篇文章的末尾说到了这一段代码可以实例化Advice。我们来看看这个方法的代码：
```java
this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
```
```java
private Advice instantiateAdvice(AspectJExpressionPointcut pcut) {
	//入参为切点表达式类
	//这里是通过调用aspectJAdvisorFactory来获取Advice
	//aspectJAdvisorFactory的实例是ReflectiveAspectJAdvisorFactory所以最终我们还是要到
	//ReflectiveAspectJAdvisorFactory中去分析Advice的获取过程
	//ReflectiveAspectJAdvisorFactory真是一个重要的类啊Advisor和Advice的获取都是在这个类中完成的
	//入参为：通知方法、切点表达式类、切面实例、切面的一个顺序、切面类名
	//倒腾了倒腾去基本上还是这几个参数
	return this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pcut,
			this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
	}
```
ReflectiveAspectJAdvisorFactory中getAdvice方法的代码如下
```java
	public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
		//切面类 带有@Aspect注解的类
		Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		//验证切面类 不再分析了
		validate(candidateAspectClass);
		//又来一遍 根据通知方法 获取通知注解相关信息
		//在获取Advisor的方法 我们已经见过这个调用。这个在Spring的AnnotationUtils中会缓存方法的注解信息
		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}
		//再去校验一遍 如果不是切面类 则抛出异常
		if (!isAspect(candidateAspectClass)) {
			throw new AopConfigException("Advice must be declared inside an aspect type: " +
					"Offending method '" + candidateAdviceMethod + "' in class [" +
					candidateAspectClass.getName() + "]");
		}
		AbstractAspectJAdvice springAdvice;
		//在上一篇文章的分析中 我们说过在AspectJAnnotation中会存放通知类型
		switch (aspectJAnnotation.getAnnotationType()) {
			//如果是前置通知，则直接创建AspectJMethodBeforeAdvice实例
			//入参为：通知方法、切点表达式、切面实例
			case AtBefore:
				springAdvice = new AspectJMethodBeforeAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			//如果是后置通知，则直接创建AspectJAfterAdvice实例
			//入参为:通知方法、切点表达式、切面实例
			case AtAfter:
				springAdvice = new AspectJAfterAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			//如果是后置返回通知，则直接创建AspectJAfterReturningAdvice实例
			//入参为：通知方法、切点表达式、切面实例
			case AtAfterReturning:
				springAdvice = new AspectJAfterReturningAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
				//设置后置返回值的参数name
				if (StringUtils.hasText(afterReturningAnnotation.returning())) {
					springAdvice.setReturningName(afterReturningAnnotation.returning());
				}
				break;
			//如果是后置异常通知，则直接创建AspectJAfterThrowingAdvice实例
			//入参为：通知方法、切点表达式、切面实例
			case AtAfterThrowing:
				springAdvice = new AspectJAfterThrowingAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
				//设置后置异常通知 异常类型参数name
				if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
					springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
				}
				break;
			//如果是后置异常通知，则直接创建AspectJAfterThrowingAdvice实例
			//入参为：通知方法、切点表达式、切面实例
			case AtAround:
				springAdvice = new AspectJAroundAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			//如果是切点方法，则什么也不做
			case AtPointcut:
				if (logger.isDebugEnabled()) {
					logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
				}
				return null;
			//上面的那几种情况都不是的话，则抛出异常
			default:
				throw new UnsupportedOperationException(
						"Unsupported advice type on method: " + candidateAdviceMethod);
		}

		// Now to configure the advice...
		//切面的名字
		springAdvice.setAspectName(aspectName);
		springAdvice.setDeclarationOrder(declarationOrder);
		//通知注解中的参数名
		String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
		if (argNames != null) {
			springAdvice.setArgumentNamesFromStringArray(argNames);
		}
		//参数绑定
		springAdvice.calculateArgumentBindings();
		return springAdvice;
	}
```
上面即是获取Advice的过程。我们简单的看一下calculateArgumentBindings这个方法做了什么事：
calculateArgumentBindings
```java
	public synchronized final void calculateArgumentBindings() {
		//如果已经进行过参数绑定了  或者通知方法中没有参数
		if (this.argumentsIntrospected || this.parameterTypes.length == 0) {
			return;
		}
		
		int numUnboundArgs = this.parameterTypes.length;
		//通知方法参数类型
		Class<?>[] parameterTypes = this.aspectJAdviceMethod.getParameterTypes();
		//如果第一个参数是JoinPoint或者ProceedingJoinPoint
		if (maybeBindJoinPoint(parameterTypes[0]) || 
		//这个方法中还有一个校验 即只有在环绕通知中第一个参数类型才能是ProceedingJoinPoint
		maybeBindProceedingJoinPoint(parameterTypes[0])) {
			numUnboundArgs--;
		}
		//如果第一个参数是JoinPoint.StaticPart
		else if (maybeBindJoinPointStaticPart(parameterTypes[0])) {
			numUnboundArgs--;
		}

		if (numUnboundArgs > 0) {
			//进行参数绑定 绑定过程略复杂
			//常见的场景是我们使用 后置返回通知和后置异常通知的时候 需要指定 returning和throwing的值 
			bindArgumentsByName(numUnboundArgs);
		}
		this.argumentsIntrospected = true;
	}
```
**通过前面的分析我们可以了解到一个切面中的通知方法会生成一个Advisor实例(如InstantiationModelAwarePointcutAdvisorImpl，其实这个也是我们在SpringAOP中最常用的一个Advisor实现类)，在生成这个Advisor实例的过程中会创建一个相应的Advice实例!  一个通知方法---->一个Advisor(包含Pointcut)------>一个Advice！**
PS：这里只是一个生成Advice的地方，在其他的地方也会生成Advice，我们在以后再分析。

我们在之前的文章中分析了Advisor的生成过程以及在Advisor中生成Advise的过程。在这一篇文章中我们说一下为目标类挑选合适的Advisor的过程。通过之前的分析我们知道，一个切面类可以生成多个Advisor(多个切面类的话那就更多多的Advisor了)，这些Advisor是否都能适用于我们的目标类呢？这就需要通过Advisor中所拥有的Pointcut来进行判断了。先回到我们最开始的例子：
```
//手工创建一个实例
AspectJService aspectJService = new AspectJServiceImpl();
//使用AspectJ语法 自动创建代理对象
AspectJProxyFactory aspectJProxyFactory = new AspectJProxyFactory(aspectJService);
```
我们在我们的AspectJProxyFactory中传入了我们的目标对象。我们再回到AspectJProxyFactory的addAdvisorsFromAspectInstanceFactory方法中。
```
	private void addAdvisorsFromAspectInstanceFactory(MetadataAwareAspectInstanceFactory instanceFactory) {
		//获取Advisor的过程我们在之前分析了
		List<Advisor> advisors = this.aspectFactory.getAdvisors(instanceFactory);
		//这句代码的意思是为我们的目标类挑选合适的Advisor也是我们这一次要分析的内容
		advisors = AopUtils.findAdvisorsThatCanApply(advisors, getTargetClass());
		AspectJProxyUtils.makeAdvisorChainAspectJCapableIfNecessary(advisors);
		//为Advisor进行排序
		AnnotationAwareOrderComparator.sort(advisors);
		addAdvisors(advisors);
	}
```
在AspectJProxyFactory中是通过调用AopUtils中的findAdvisorsThatCanApply方法来为目标类挑选合适的Advisor的或者是进判断哪些Advisor可以作用于目标类。在这个方法中传入了两个参数，一个参数是Advisor的集合，一个参数是目标类Class。我们看一下getTargetClass()这个方法的内容：
AdvisedSupport#getTargetClass
```
	@Override
	public Class<?> getTargetClass() {
		//直接调用targetSource的getTargetClass方法
		//其实也是相当于调用target.getClass()
		return this.targetSource.getTargetClass();
	}
```
OK，下面我们进入到AopUtils#findAdvisorsThatCanApply中看一下这个方法的内容
```
	public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		//如果传入的Advisor集合为空的话，直接返回这个空集合
		//这里没有判断candidateAdvisors不为null的情况 因为在获取Advisor的地方是先创建一个空的集合，再进行添加Advisor的动作
		//不过还是加一下不为null的判断更好一点
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
		//创建一个合适的Advisor的集合 eligible
		List<Advisor> eligibleAdvisors = new LinkedList<Advisor>();
		//循环所有的Advisor
		for (Advisor candidate : candidateAdvisors) {
			//如果Advisor是IntroductionAdvisor  引介增强 可以为目标类 通过AOP的方式添加一些接口实现
			//引介增强是一种比较我们接触的比较少的增强 我们可以在以后的文章单独做个分析
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				eligibleAdvisors.add(candidate);
			}
		}
		//是否有引介增强
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
			//如果是IntroductionAdvisor类型的话 则直接跳过
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}
			//判断此Advisor是否适用于target
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		return eligibleAdvisors;
	}
```
canApply方法的内容
```
	public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
		//如果是IntroductionAdvisor的话，则调用IntroductionAdvisor类型的实例进行类的过滤
		//这里是直接调用的ClassFilter的matches方法
		if (advisor instanceof IntroductionAdvisor) {
			return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
		}
		//通常我们的Advisor都是PointcutAdvisor类型
		else if (advisor instanceof PointcutAdvisor) {
			PointcutAdvisor pca = (PointcutAdvisor) advisor;
			//这里从Advisor中获取Pointcut的实现类 这里是AspectJExpressionPointcut
			return canApply(pca.getPointcut(), targetClass, hasIntroductions);
		}
		else {
			// It doesn't have a pointcut so we assume it applies.
			return true;
		}
	}
```
重载canApply方法的内容。
```
	public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
		Assert.notNull(pc, "Pointcut must not be null");
		//进行切点表达式的匹配最重要的就是 ClassFilter 和 MethodMatcher这两个方法的实现。
		//MethodMatcher中有两个matches方法。一个参数是只有Method对象和targetclass，另一个参数有
		//Method对象和targetClass对象还有一个Method的方法参数 他们两个的区别是：
		//两个参数的matches是用于静态的方法匹配 三个参数的matches是在运行期动态的进行方法匹配的
		//先进行ClassFilter的matches方法校验
		//首先这个类要在所匹配的规则下
		if (!pc.getClassFilter().matches(targetClass)) {
			return false;
		}
		//再进行 MethodMatcher 方法级别的校验
		MethodMatcher methodMatcher = pc.getMethodMatcher();
		if (methodMatcher == MethodMatcher.TRUE) {
			// No need to iterate the methods if we're matching any method anyway...
			return true;
		}

		IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
		if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
			introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
		}

		Set<Class<?>> classes = new LinkedHashSet<Class<?>>(ClassUtils.getAllInterfacesForClassAsSet(targetClass));
		classes.add(targetClass);
		for (Class<?> clazz : classes) {
			Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
			//只要有一个方法能匹配到就返回true
			//这里就会有一个问题：因为在一个目标中可能会有多个方法存在，有的方法是满足这个切点的匹配规则的
			//但是也可能有一些方法是不匹配切点规则的，这里检测的是只有一个Method满足切点规则就返回true了
			//所以在运行时进行方法拦截的时候还会有一次运行时的方法切点规则匹配
			for (Method method : methods) {
				if ((introductionAwareMethodMatcher != null &&
						introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions)) ||
						methodMatcher.matches(method, targetClass)) {
					return true;
				}
			}
		}

		return false;
	}
```
从上面的代码来看，这次我们要分析的重点就在AspectJExpressionPointcut这个类中了。在AspectJExpressPointcut中预先初始化了这些内容：你能看出来这是什么内容吗？
```
	static {
		//execution
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.EXECUTION);
		//args
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.ARGS);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.REFERENCE);
		//this
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.THIS);
		//target
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.TARGET);
		//within
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.WITHIN);
		//@annotation
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_ANNOTATION);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_WITHIN);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_ARGS);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_TARGET);
	}
```
我们来看一下这段代码，这是要从AspectJExpressPointcut中获取ClassFilter
```
		if (!pc.getClassFilter().matches(targetClass)) {
			return false;
		}
```
```
	public ClassFilter getClassFilter() {
		checkReadyToMatch();
		return this;
	}
	private void checkReadyToMatch() {
		//如果没有expression的值的话 直接抛出异常
		if (getExpression() == null) {
			throw new IllegalStateException("Must set property 'expression' before attempting to match");
		}
		if (this.pointcutExpression == null) {
			//选择类加载器
			this.pointcutClassLoader = determinePointcutClassLoader();
			//构建PointcutExpression的实例
			this.pointcutExpression = buildPointcutExpression(this.pointcutClassLoader);
		}
	}
	//下面的这一段逻辑完全就是aspectj这个jar中的内容了 很复杂不多说了。。。。
	private PointcutExpression buildPointcutExpression(ClassLoader classLoader) {
		//初始化一个PointcutParser的实例 PointcutParser   aspectj中提供的类
		PointcutParser parser = initializePointcutParser(classLoader);
		PointcutParameter[] pointcutParameters = new PointcutParameter[this.pointcutParameterNames.length];
		for (int i = 0; i < pointcutParameters.length; i++) {
			pointcutParameters[i] = parser.createPointcutParameter(
					this.pointcutParameterNames[i], this.pointcutParameterTypes[i]);
		}
		//解析切点表达式 我们的切点表示有可能会这样写：在切面中定义一个专门的切面表达式方法
		//在不同的通知类型中引入这个切点表达式的方法名 
		return parser.parsePointcutExpression(replaceBooleanOperators(getExpression()),
				this.pointcutDeclarationScope, pointcutParameters);
	}
	//这个方法 将表达式中的 and 替换为 && or 替换为 ||   not 替换为 !
	private String replaceBooleanOperators(String pcExpr) {
		String result = StringUtils.replace(pcExpr, " and ", " && ");
		result = StringUtils.replace(result, " or ", " || ");
		result = StringUtils.replace(result, " not ", " ! ");
		return result;
	}
```
由于我们在项目开发中使用SpringAOP基本上都是用的AspectJ的注解的形式。AspectJ对于切点的匹配规则解析是一个比较复杂的过程，所以我们在使用的AspectJ中的切点表达式的时候要按照它的一个规则来进行书写。
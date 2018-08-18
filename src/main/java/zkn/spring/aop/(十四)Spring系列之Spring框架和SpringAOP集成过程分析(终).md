转载请注明出处：https://blog.csdn.net/zknxx/article/details/80808447
在这篇文章中我们接着上一篇的文章说。在上一篇文章中我们提到了getAdvicesAndAdvisorsForBean这个方法，这个方法的内容是为目标对象挑选合适的Advisor类，其源码如下：
```
	//targetSource 为null
	protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {
		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		//如果获取的Advisor为空的话，则直接返回DO_NOT_PROXY 返回这个值的时候 是不创建代理对象
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}
	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
		//这个方法我们在上一章中分析过了   获取Spring容器中的所有的通知方法  封装为Advisor集合
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		//这一步就是为目标对象挑选合适的Advisor 即目标对象和切点表达式相匹配
		//此处的分析  请参考这里：https://blog.csdn.net/zknxx/article/details/79735405
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		//这个方法做的事是向 上一步获取到的Advisor中 插入ExposeInvocationInterceptor.ADVISOR 插在第一个位置
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
			//排序  很复杂的实现  不说了  如果你对一个目标对象使用多个 相同类型的通知的话 请把这些通知放到不同的Aspect中，并实现Order接口或者使用Ordered注解标注顺序
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}
```
最后我们再看看目标对象的创建过程：
```
	//specificInterceptors 上面的方法中获取到的Advisor 
	//targetSource 为 new SingletonTargetSource(bean) 
	//将Spring容器中创建出来的bean封装为了SingletonTargetSource
	protected Object createProxy(
			Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {
		//这里将目标Bean的Class设置为BeanDefinition中的一个属性 
		//在使用JDK动态代理的时候 可能会用到这个属性  https://stackoverflow.com/questions/45463757/what-is-interface-based-proxying
		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}
		//这里创建了一个新的 ProxyFactory
		ProxyFactory proxyFactory = new ProxyFactory();
		//copy原类的配置 重写赋值的动作   单例变多例
		proxyFactory.copyFrom(this);
		//这里是 判断是否强制使用 cglib
		if (!proxyFactory.isProxyTargetClass()) {
			//这里 判断是 使用jdk动态代理 还是cglib代理
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}
		//这里又重写构建了一次 Advisor  看看是否 设置了通用的拦截
		//这里讲所有的Advisor和配置的通用拦截都转换为了Advisor类型
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		for (Advisor advisor : advisors) {
			proxyFactory.addAdvisor(advisor);
		}
		//targetSource 里是包含目标对象的
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}
		//创建代理对象 关于创建代理对象的分析 请参考：https://blog.csdn.net/zknxx/article/details/79764107
		return proxyFactory.getProxy(getProxyClassLoader());
	}
```
好了，到此关于AOP的分析就先暂时告一段落了。
转载请注明出处：https://blog.csdn.net/zknxx/article/details/80808447
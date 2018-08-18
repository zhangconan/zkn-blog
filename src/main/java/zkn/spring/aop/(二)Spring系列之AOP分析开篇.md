在开始Spring的AOP分析之前，先来看一个很老的一个问题。。。假设你在开发的过程中，需要在类A中的方法之前执行一些逻辑(我们称为逻辑A)，你可能的一个做法是直接修改类A中的方法，在类A中的方法的开始处写上要添加的代码，你还可能会给类A生成一个代理类，去对调用方法进行拦截，在代理类里面去执行相应的逻辑（逻辑A）。直接修改类A中的方法一般是我们不推荐的方式(存在改动量大、不易扩展等问题)，我们通常采用的做法是为类A生成一个代理对象，在执行的时候去执行我们的代理对象。小例子如下(这里使用的JDK动态代理的方式)：  

```
/**
 * @author zkn
 * @date 2018/3/18
 */
public interface ProxyService {
**_``_**
    /**
     * 测试方法
     */
    void testProxy();
}
/**
 * @author zkn
 * @date 2018/3/18
 */
public class ProxyServiceImpl implements ProxyService {

    /**
     * 测试方法
     */
    @Override
    public void testProxy() {
        System.out.println("我是ProxyService中的测试方法......");
    }
}
/**
 * @author zkn
 * @date 2018/3/18
 */
public class LogicClassFir {
    /**
     * 逻辑方法A
     */
    public void logicMethodFir() {
        System.out.println("我是第一个逻辑方法的内容........");
    }
}
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

/**
 * @author zkn
 * @date 2018/3/18
 */
public class ProxyCreator implements InvocationHandler {

    private Object proxy;

    private LogicClassFir logicObj;

    public ProxyCreator(Object proxy, LogicClassFir logicObj) {
        this.proxy = proxy;
        this.logicObj = logicObj;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        logicObj.logicMethodFir();
        return method.invoke(this.proxy, args);
    }
}
public class ProxyTest {

    public static void main(String[] args) {

        LogicClassFir logicClassFir = new LogicClassFir();
        ProxyService targetService = new ProxyServiceImpl();
        ProxyService proxyService = (ProxyService) Proxy.newProxyInstance(ProxyCreator.class.getClassLoader(),
                new Class[]{ProxyService.class}, new ProxyCreator(targetService, logicClassFir));
        proxyService.testProxy();
    }
}
```  

输出结果如下：  

![输出结果](./img/二章输出结果.png)
现在又来问题：我们只想对类A中的部分方法执行逻辑A，相对类A中的另一部分执行逻辑B，这个时候我们应该怎么做呢？如果我们想对类A中的方法先执行一个类B中的逻辑A，接着再执行类C中的逻辑B这个时候我们应该怎么做呢？如果我们想对类A中的方法先执行类B中的逻辑A方法，再执行类A中的方法，再接着执行类C中的逻辑B方法，这个时候我们又改怎么做呢？还有其他的各种不同的拦截需求。那么这个时候我们可能首先需要定义一种拦截的规则(什么样的方法应该被拦截、是在方法的执行前还是执行后进行拦截等)，然后拦截要执行的逻辑如果在不同的类中，那么我们还需要将这些逻辑适配成一个统一的调用形式。这些内容对应到SpringAOP中是：Pointcut(切点表达式)、Advice(通知类型BeforeAdvice、AfterAdvice等)、Interceptor(统一的方法拦截MethodInterceptor等)、Invocation(组装链式调用MethodInvocation等)、AopProxy（创建代理对象JdkDynamicAopProxy、CglibAopProxy）。我们将在下面的系列文章中对这些内容进行一下分析。

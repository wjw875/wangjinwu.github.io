---
layout: post
title:  "Spring-Boot之AOP使用与原理解析"
categories: Spring-Boot
---

AOP(Aspect Oriented Programming)是通过预编译的方式和运行期动态代理实现程序功能的技术，本文通过实例展示了AOP的基本概念，对动态代理技术和AOP的实现原理进行了解析，并且展示了在Spring-Boot项目中通过AOP技术实现接口日志拦截。

## 一、AOP概念分析

单独的AOP概念分析比较枯燥，我们结合实例来分析AOP概念，先在Spring-Boot工程定义类文件SkeletonAspect.java，内容如下。

```java
@Aspect
@Configuration
public class SkeletonAspect {
    @Pointcut("execution(* com.wjw875.SuperDemo.control.AccountControl.healthy(..))")
    public void healthyControl() {
    }

    @Around("healthyControl()")
    public Object doAround(ProceedingJoinPoint pjp) throws Throwable {
        return pjp.proceed();
    }

    @Before("healthyControl()")
    public void doBefore(JoinPoint joinPoint) {
        System.out.println("执行方法前 : " + Arrays.toString(joinPoint.getArgs()));
    }

    @After("healthyControl()")
    public void after(JoinPoint joinPoint) {
        System.out.println("执行方法后：" + Arrays.toString(joinPoint.getArgs()));
    }

    @AfterReturning(pointcut = "healthyControl()", returning = "ret")
    public void doAfterReturning(Object ret) {
        System.out.println("方法的返回值 : " + ret);
    }

    @AfterThrowing(pointcut = "healthyControl()", throwing = "ex")
    public void AfterThrowing(JoinPoint joinPoint, Throwable ex) {
        System.out.println("方法执行异常 : " + ex);
    }
}
```

启动服务，通过Postman访问healthy()方法，可以对healthyControl连接点执行前、执行中、执行后、正常返回以及抛出异常进行拦截，结合上述类内容，我们来分析AOP的基本概念。

Aspect（切面）： Aspect 声明类似于 Java 中的类声明，在 Aspect 中会包含着一些 Pointcut 以及相应的 Advice，如SkeletonAspect类就是一个切面。

Joint point（连接点）：表示在程序中明确定义的点，典型的包括方法调用，对类成员的访问以及异常处理程序块的执行等，还可以嵌套其它 joint point，如* com.wjw875.SuperDemo.control.AccountControl.healthy(..)就是一个连接的。

Pointcut（切点）：表示一组 joint point，这些 joint point 或是通过逻辑关系组合起来，或是通过通配、正则表达式等方式集中起来，在SkeletonAspect类中healthyControl()定义的就是切点。

Advice（通知）：Advice 定义了在 Pointcut 里面定义的程序点具体要做的操作，它通过 before、after 和 around 来区别是在每个 joint point 之前、之后还是代替执行的代码，在SkeletonAspect类中定义了四个通知，分别用@Around、@Before、@After、@AfterReturning以及@AfterThrowing注解标注。

## 二、AOP原理解析  

AOP是通过动态代理的方式实现的，我们通过一个简单的示例展示JDK动态代理的实现原理，先定义被代理类的接口DoSomething，方法为doSomething()。

```java
public interface DoSomething {
    void doSomething();
}
```

然后，定义类DoMyThing实现DoSomething接口的doSomething()方法，该方法类似AOP的连接点。

```java
public class DoMyThing implements DoSomething {
    @Override
    public void doSomething() {
        System.out.println("invocation DoMyThing");
    }
}
```

定义拦截类接口Invocation，提供doBefore()和doAfter()两个方法。

```java
public interface Invocation {
    void doBefore();
    void doAfter();
}
```

定义MyInvocation实现接口Invocation中的doBefore()和doAfter()两个方法。

```java
public class MyInvocation implements Invocation {
    @Override
    public void doBefore() {
        System.out.println("doBefore");
    }

    @Override
    public void doAfter() {
        System.out.println("doAfter");
    }
}
```

定义DynamicProxy类实现InvocationHandler接口的invoke()方法，在JDK中是通过InvocationHandler实现动态代理的。

```java
public class DynamicProxy implements InvocationHandler {

    private Object target = null;
    private Invocation invocation = null;

    public static Object getDynamicProxy(Object target, Invocation invocation) {
        DynamicProxy proxy = new DynamicProxy();
        proxy.target = target;
        proxy.invocation = invocation;

        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), proxy);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        invocation.doBefore(); // 执行前的拦截方法
        Object retObject = method.invoke(target, args); // 执行方法
        invocation.doAfter(); // 执行后的拦截方法
        return retObject;
    }
}
```

通过运行以下测试代码，即可看到动态代理的效果。

```java
public class Test {
    public static void main(String[] args) {
        DoSomething doSomething = new DoMyThing();
        DoSomething proxy = (DoSomething) DynamicProxy.getDynamicProxy(doSomething, new MyInvocation());
        proxy.doSomething();
    }
}
```

AOP的实现原理是基于上述动态代理的原理实现的，关于AOP的动态代理执行，有JDK的动态代理和CGLIB的动态代理，在类DefaultAopProxyFactory中选择AOP代理使用的是JdkDynamicAopProxy还是ObjenesisCglibAopProxy，代码如下。

```java
@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
   if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
      Class<?> targetClass = config.getTargetClass();
      if (targetClass == null) {
         throw new AopConfigException("TargetSource cannot determine target class: " +
               "Either an interface or a target is required for proxy creation.");
      }
      if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
         return new JdkDynamicAopProxy(config);
      }
      return new ObjenesisCglibAopProxy(config);
   }
   else {
      return new JdkDynamicAopProxy(config);
   }
}
```

JdkDynamicAopProxy类会在invoke()方法创建ReflectiveMethodInvocation对象，该对象通过proceed()方法实现拦截。ObjenesisCglibAopProxy 类创建的是CglibMethodInvocation对象，该对象扩展了ReflectiveMethodInvocation对象。ReflectiveMethodInvocation的proceed()代码如下所示。

```java
@Override
@Nullable
public Object proceed() throws Throwable {
   // We start with an index of -1 and increment early.
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint();
   }

   Object interceptorOrInterceptionAdvice =
         this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
   if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
      // Evaluate dynamic method matcher here: static part will already have
      // been evaluated and found to match.
      InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
      Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
      if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
         return dm.interceptor.invoke(this);
      }
      else {
         // Dynamic matching failed.
         // Skip this interceptor and invoke the next in the chain.
         return proceed();
      }
   }
   else {
      // It's an interceptor, so we just invoke it: The pointcut will have
      // been evaluated statically before this object was constructed.
      return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
   }
}
```

通过ReflectiveMethodInvocation对象的proceed()方法，就可以实现对连接点方法的拦截效果。

## 三、AOP应用示例

我们通过服务接口日志记录示例展示AOP的实际应用，首先需要在resources目录下添加日志配置文件logback.xml，然后在Spring-Boot的gradle.build文件中引入AOP依赖org.springframework.boot:spring-boot-starter-aop，顺便添加fastjson备用，具体依赖如下所示。

```javascript
dependencies {
   compile fileTree(dir: 'libs', include: ['*.jar'])

   implementation 'com.alibaba:fastjson:1.2.5'

   implementation 'org.springframework.boot:spring-boot-starter'
   implementation 'org.springframework.boot:spring-boot-starter-web'
   implementation 'org.springframework.boot:spring-boot-starter-aop'

   implementation 'mysql:mysql-connector-java'
   implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:1.3.2'

   testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

日志记录的具体要求是对com.wjw875.SuperDemo.control内的接口的访问参数和返回结果都需要记录，对监控检查结果，不需要记录访问日志。

定义账号接口切点，用户记录账号接口的访问日志，如下所示。

```java
@Pointcut("execution(* com.wjw875.SuperDemo.control.*Control.*(..))")
public void accountControl() {
}
```

定义健康检查接口切点，用于排除记录监控检查接口的访问日志，如下所示。

```java
@Pointcut("execution(* com.wjw875.SuperDemo.control.*Control.healthy(..))")
public void healthyControl() {
}
```

定义环绕事件，用户记录账号切点的访问日志且排除监控检查接口的访问日志，环绕事件实现如下所示，注意通过!healthyControl()排除监控检查接口的日志记录。

```java
    @Around("accountControl() && !healthyControl()")
    public Object doAround(ProceedingJoinPoint pjp) throws Throwable {
        Date startTime = new Date();
        StopWatch testWatch = new StopWatch();
        testWatch.start();

        StopWatch sw = new StopWatch();
        sw.start();

        // obj的值就是被拦截方法的返回值
        Object obj = pjp.proceed();

        sw.stop();
        testWatch.stop();

        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
        String url = request.getRequestURL().toString();

        Object[] args = pjp.getArgs(); // 获取访问参数
        StringBuffer buffer = new StringBuffer();
        for (Object arg : args) {
//            if (obj instanceof MultipartFile)
//                continue;
            ObjectMapper mapper = new ObjectMapper();
            if (arg instanceof AccountDomain) {
                AccountDomain domain = (AccountDomain)arg;
                domain.setPassword("******");
            }
            String s = mapper.writeValueAsString(arg);
            buffer.append(s);
        }

        SimpleDateFormat formatter = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        String sStartTime = formatter.format(startTime);
        String sEndTime = formatter.format(new Date());

        String responseContent = JSONObject.toJSONString(obj);
        logger.info("[REQ] RequestIP: {}, RequestURL: {}({}), StartTime: {}, EndTime: {}, ExecutionTime: {} ms, LogRecordTime: {} ms, Response: {}",
                getRequestIP(request), url, buffer.toString(), sStartTime, sEndTime, sw.getTotalTimeMillis(), testWatch.getTotalTimeMillis() - sw.getTotalTimeMillis(), responseContent); // 访问记录

        return obj;
    }  
```

注意日志中有记录用户的密码，在实际情况中，日志需要过滤用户的敏感信息，通过Postman访问login接口，记录的日志如下，通过"******"代替用户真实密码。

```javascript
[2019-08-25 11:19:11] [INFO ] com.wjw875.SuperDemo.record.LogRecordAspect$$EnhancerBySpringCGLIB$$be63c7b0 - [REQ] RequestIP: 127.0.0.1, RequestURL: http://127.0.0.1:8088/account/login({"uid":0,"phone":"123","password":"******","email":null}), StartTime: 2019-08-25 11:19:10, EndTime: 2019-08-25 11:19:11, ExecutionTime: 964 ms, LogRecordTime: 0 ms, Response: {"body":{"code":0,"data":{"status":0,"token":"rnfmg8lnbncmmnw3sadktqg0ixqyqwwx","uid":1},"message":"success"},"headers":{},"statusCode":"OK","statusCodeValue":200}
```

AOP是Spring的核心技术之一，记录接口访问日志是AOP在Spring-Boot框架中的一个典型应用场景，其还可以应用于更多的场景，如数据库事务处理，这将在后续文章中有涉及。

源码链接：https://github.com/wjw875/SuperDemo/tree/func_log_branch

```
2019年09月01日 珠海保利海上五月花  
QQ: 2691819215 Email: wjw875@163.com
```

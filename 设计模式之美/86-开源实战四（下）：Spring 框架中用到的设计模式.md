[toc]

## 86 | 开源实战四（下）：总结 Spring 框架用到的 11 种设计模式

### 适配器模式在 Spring 中的应用

1.  适配器模式，可以让其满足开闭原则，能更好地支持扩展。

2.  利用适配器模式，我们将不同方式定义的 Controller 类中的函数，适配为统一的函数定义。

3.  Spring 定义了统一的接口 HandlerAdapter，并且对每种 Controller 定义了对应的适配器类。包括：AnnotationMethodHandlerAdapter、SimpleControllerHandlerAdapter、SimpleServletHandlerAdapter 等。源码如下：

    -   ```java
        
        public interface HandlerAdapter {
          boolean supports(Object var1);
        
          ModelAndView handle(HttpServletRequest var1, HttpServletResponse var2, Object var3) throws Exception;
        
          long getLastModified(HttpServletRequest var1, Object var2);
        }
        
        // 对应实现Controller接口的Controller
        public class SimpleControllerHandlerAdapter implements HandlerAdapter {
          public SimpleControllerHandlerAdapter() {
          }
        
          public boolean supports(Object handler) {
            return handler instanceof Controller;
          }
        
          public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            return ((Controller)handler).handleRequest(request, response);
          }
        
          public long getLastModified(HttpServletRequest request, Object handler) {
            return handler instanceof LastModified ? ((LastModified)handler).getLastModified(request) : -1L;
          }
        }
        
        // 对应实现Servlet接口的Controller
        public class SimpleServletHandlerAdapter implements HandlerAdapter {
          public SimpleServletHandlerAdapter() {
          }
        
          public boolean supports(Object handler) {
            return handler instanceof Servlet;
          }
        
          public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            ((Servlet)handler).service(request, response);
            return null;
          }
        
          public long getLastModified(HttpServletRequest request, Object handler) {
            return -1L;
          }
        }
        
        //AnnotationMethodHandlerAdapter对应通过注解实现的Controller，
        //代码太多了，我就不贴在这里了
        ```

4.  在 DispatcherServlet 类中，我们就不需要区分对待不同的 Controller 对象了，统一调用 HandlerAdapter 的 handle() 函数就可以了。伪代码如下：

    -   ```java
        
        // 之前的实现方式
        Handler handler = handlerMapping.get(URL);
        if (handler instanceof Controller) {
          ((Controller)handler).handleRequest(...);
        } else if (handler instanceof Servlet) {
          ((Servlet)handler).service(...);
        } else if (hanlder 对应通过注解来定义的Controller) {
          反射调用方法...
        }
        
        // 现在实现方式
        HandlerAdapter handlerAdapter = handlerMapping.get(URL);
        handlerAdapter.handle(...);
        ```

    -   

### 策略模式在 Spring 中的应用

1.  应用场景：针对不同的被代理类，Spring 会在运行时动态地选择不同的动态代理实现方式。

2.  策略模式包含三部分：策略的**定义、创建和使用**。

3.  策略的**定义**，我们只需要定义一个策略接口，让不同的策略类都实现这一个策略接口。

    -   ```java
        
        public interface AopProxy {
          Object getProxy();
          Object getProxy(ClassLoader var1);
        }
        ```

4.  策略的**创建**，一般是通过工厂方法来实现。

    -   ```java
        
        public interface AopProxyFactory {
          AopProxy createAopProxy(AdvisedSupport var1) throws AopConfigException;
        }
        
        public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {
          public DefaultAopProxyFactory() {
          }
        
          public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
            if (!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
              return new JdkDynamicAopProxy(config);
            } else {
              Class<?> targetClass = config.getTargetClass();
              if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
              } else {
                return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass) ? new ObjenesisCglibAopProxy(config) : new JdkDynamicAopProxy(config));
              }
            }
          }
        
          //用来判断用哪个动态代理实现方式
          private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
            Class<?>[] ifcs = config.getProxiedInterfaces();
            return ifcs.length == 0 || ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0]);
          }
        }
        ```

5.  策略模式的典型应用场景，一般是通过环境变量、状态值、计算结果等动态决定使用哪个策略。

### 组合模式在 Spring 中的应用

1.  组合模式主要**应用在能表示成树形结构的一组数据上**。树中的结点分为**子节点和中间节点**两类。
    -   对应到 Spring 源码，EhCacheManager、SimpleCacheManager、NoOpCacheManager、RedisCacheManager 等表示叶子节点，CompositeCacheManager 表示中间节点

### 装饰器模式在 Spring 中的应用

1.  Spring 使用到了装饰器模式.TransactionAwareCacheDecorator 增加了对事务的支持，在事务提交、回滚的时候分别对 Cache 的数据进行处理。

### 工厂模式在 Spring 中的应用

1.  在 Spring 中，工厂模式最经典的应用莫过于实现 IOC 容器，对应的 Spring 源码主要是 BeanFactory 类和 ApplicationContext 相关类（AbstractApplicationContext、ClassPathXmlApplicationContext、FileSystemXmlApplicationContext…）

### 其他模式在 Spring 中的应用

1.  解释器模式，SpEL，全称叫 Spring Expression Language，是 Spring 中常用来编写配置的表达式语言。它定义了一系列的语法规则。我们只要按照这些语法规则来编写表达式，Spring 就能解析出表达式的含义。
2.  单例模式，
3.  职责链模式，拦截器。
4.  代理模式，AOP。

### 重点回顾

1.  我们今天提到的设计模式有 11 种，它们分别是：
    -   适配器模式、策略模式、组合模式、装饰器模式、工厂模式、单例模式、解释器模式、观察者模式、模板模式、职责链模式、代理模式。


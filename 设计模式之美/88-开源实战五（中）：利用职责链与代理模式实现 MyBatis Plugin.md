[toc]

## 88 | 开源实战五（中）：如何利用职责链与代理模式实现 MyBatis Plugin?

1.  MyBatis Plugin 跟之前讲到的 Servlet Filter、Spring Interceptor 类似，设计的初衷都是为了**框架的扩展性**、用到的主要设计模式都是**职责链模式**。
2.  MyBatis Plugin 是借助**动态代理模式**来实现职责链模式。

### MyBatis Plugin 功能介绍

1.  MyBatis Plugin 主要拦截的是 MyBatis 在执行 SQL 的过程中涉及的一些方法。

### MyBatis Plugin 的设计与实现

1.  MyBatis Plugin 是借助**动态代理**来实现职责链的。

2.  职责链模式实现一般包含：处理器（Handler）和处理器链（HandlerChain）两部分。

    -   对应到 MyBatis Plugin 的源码就是 **Interceptor 和 InterceptorChain**。
    -   此外，MyBatis Plugin 还包含一个非常重要的类：**Plugin**。它用来生成被拦截对象的动态代理。

3.  MyBatis 框架会**读取全局配置文件**（mybatis-config.xml），**解析**出 Interceptor（SqlCostTimeInterceptor），并且将它**注入**到 Configuration 类的 InterceptorChain 对象中。源码如下：

    -   ```java
        
        public class XMLConfigBuilder extends BaseBuilder {
          //解析配置
          private void parseConfiguration(XNode root) {
            try {
             //省略部分代码...
              pluginElement(root.evalNode("plugins")); //解析插件
            } catch (Exception e) {
              throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
            }
          }
        
          //解析插件
           private void pluginElement(XNode parent) throws Exception {
            if (parent != null) {
              for (XNode child : parent.getChildren()) {
                String interceptor = child.getStringAttribute("interceptor");
                Properties properties = child.getChildrenAsProperties();
                //创建Interceptor类对象
                Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
                //调用Interceptor上的setProperties()方法设置properties
                interceptorInstance.setProperties(properties);
                //下面这行代码会调用InterceptorChain.addInterceptor()方法
                configuration.addInterceptor(interceptorInstance);
              }
            }
          }
        }
        
        // Configuration类的addInterceptor()方法的代码如下所示
        public void addInterceptor(Interceptor interceptor) {
          interceptorChain.addInterceptor(interceptor);
        }
        ```

4.  我们再来看 Interceptor 和 InterceptorChain 这两个类的代码。

    -   ```java
        
        public class Invocation {
          private final Object target;
          private final Method method;
          private final Object[] args;
          // 省略构造函数和getter方法...
          public Object proceed() throws InvocationTargetException, IllegalAccessException {
            return method.invoke(target, args);
          }
        }
        public interface Interceptor {
          Object intercept(Invocation invocation) throws Throwable;
          Object plugin(Object target);
          void setProperties(Properties properties);
        }
        
        public class InterceptorChain {
          private final List<Interceptor> interceptors = new ArrayList<Interceptor>();
        
          public Object pluginAll(Object target) {
            for (Interceptor interceptor : interceptors) {
              target = interceptor.plugin(target);
            }
            return target;
          }
        
          public void addInterceptor(Interceptor interceptor) {
            interceptors.add(interceptor);
          }
          
          public List<Interceptor> getInterceptors() {
            return Collections.unmodifiableList(interceptors);
          }
        }
        ```

    -   Interceptor 类中的 intecept() 和 plugin() 函数，及 InterceptorChain 类中的 pluginAll() 函数，是最核心的三个函数。

5.  

### 重点回顾


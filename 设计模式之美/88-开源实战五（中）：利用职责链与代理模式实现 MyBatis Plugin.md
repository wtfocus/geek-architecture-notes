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

5.  这些拦截器是什么时候被触发执行的？又是如何被触发执行的呢?

    -   在执行 SQL 的过程中，MyBatis 会创建 Executor、StatementHandler、ParameterHandler、ResultSetHandler 这几个类的对象，源码如下：

    -   ```java
        
        public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
          executorType = executorType == null ? defaultExecutorType : executorType;
          executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
          Executor executor;
          if (ExecutorType.BATCH == executorType) {
            executor = new BatchExecutor(this, transaction);
          } else if (ExecutorType.REUSE == executorType) {
            executor = new ReuseExecutor(this, transaction);
          } else {
            executor = new SimpleExecutor(this, transaction);
          }
          if (cacheEnabled) {
            executor = new CachingExecutor(executor);
          }
          executor = (Executor) interceptorChain.pluginAll(executor);
          return executor;
        }
        
        public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
          ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
          parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
          return parameterHandler;
        }
        
        public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
            ResultHandler resultHandler, BoundSql boundSql) {
          ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
          resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
          return resultSetHandler;
        }
        
        public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
          StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
          statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
          return statementHandler;
        }
        ```

    -   这几个类对象的创建过程都调用了 InteceptorChain 的 pluginAll() 方法。

6.  MyBatis 中的职责链模式的实现方式比较特殊。**它对同一个目标对象嵌套多次代码（也就是 InteceptorChain 中的 pluginAll() 函数都要执行的任务）。每个代理对象（Plugin 对象）代理一个拦截器（Interceptor 对象）功能**。

    -   ```java
        
        public Object pluginAll(Object target) {
          // 嵌套代理
          for (Interceptor interceptor : interceptors) {
            target = interceptor.plugin(target);
            // 上面这行代码等于下面这行代码，target(代理对象)=target(目标对象)+interceptor(拦截器功能)
            // target = Plugin.wrap(target, interceptor);
          }
          return target;
        }
        
        // MyBatis像下面这样创建target(Executor、StatementHandler、ParameterHandler、ResultSetHandler），相当于多次嵌套代理
        Object target = interceptorChain.pluginAll(target);
        ```

### 重点回顾

1.  今天，我们带你剖析了**如何利用职责链模式和动态代理模式来实现 MyBatis Plugin**。
2.  职责链模式的实现一般包含**处理器和处理器链**两部分。对应到 MyBatis Plugin 的源码就是 **Interceptor 和 InterceptorChain**。此外，MyBatis Plugin 还包含另外一个非常重要的类：**Plugin 类**。它用来生成被拦截对象的动态代理。
3.  在职责链模式的实现思路上，MyBatis Plugin 采用**嵌套动态代理**的方法来实现，实现思路很有技巧。


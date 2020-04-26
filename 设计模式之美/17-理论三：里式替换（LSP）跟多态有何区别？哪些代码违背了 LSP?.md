[toc]

## 17 | 理论三：里式替换（LSP）跟多态有何区别？哪些代码违背了 LSP?

-   今天，我们来学习SOLID 中的 L 对应的原则：**里式替换原则**。

-   我主要通过几个反倒，带你看看：

    >   哪些代码是违反里式替换原则的？
    >
    >   我们该如何将它们改造成满足里式替换原则？
    >
    >   此外，这条原则从定义上看起来，跟我们之前讲过的“多态”有点类似。所以，我们一会儿也会讲一下，它跟多态的区别。

### 如何理解“里式替换原则”？

-   里式替换原则（LSP）:

    >   子类对象能够替换程序中父类对象出现的任何地方，并且保证原来程序的逻辑行为不变及正非确性不被破坏。

-   示例：

    -   需求：

        -   父类 Transporter 使用 org.apache.http 库中的 HttpClient 类来传输网络数据。子类 SecurityTransporter 继承父类 Transporter，增加了额外的功能，运行传输 appId 和 appToken 安全认证信息。

    -   代码：

        -   ```java
            
            public class Transporter {
              private HttpClient httpClient;
              
              public Transporter(HttpClient httpClient) {
                this.httpClient = httpClient;
              }
            
              public Response sendRequest(Request request) {
                // ...use httpClient to send request
              }
            }
            
            public class SecurityTransporter extends Transporter {
              private String appId;
              private String appToken;
            
              public SecurityTransporter(HttpClient httpClient, String appId, String appToken) {
                super(httpClient);
                this.appId = appId;
                this.appToken = appToken;
              }
            
              @Override
              public Response sendRequest(Request request) {
                if (StringUtils.isNotBlank(appId) && StringUtils.isNotBlank(appToken)) {
                  request.addPayload("app-id", appId);
                  request.addPayload("app-token", appToken);
                }
                return super.sendRequest(request);
              }
            }
            
            public class Demo {    
              public void demoFunction(Transporter transporter) {    
                Reuqest request = new Request();
                //...省略设置request中数据值的代码...
                Response response = transporter.sendRequest(request);
                //...省略其他逻辑...
              }
            }
            
            // 里式替换原则
            Demo demo = new Demo();
            demo.demofunction(new SecurityTransporter(/*省略参数*/););
            ```

    -   分析：

        -   上面代码中，子类 SecurityTransporter 的设计完全符合里式替换原则，可以替换父类出现的任何位置，并且原来代码的逻辑行为不变且正确性也没有破坏。

    -   代码改造

        -   ```java
            
            // 改造前：
            public class SecurityTransporter extends Transporter {
              //...省略其他代码..
              @Override
              public Response sendRequest(Request request) {
                if (StringUtils.isNotBlank(appId) && StringUtils.isNotBlank(appToken)) {
                  request.addPayload("app-id", appId);
                  request.addPayload("app-token", appToken);
                }
                return super.sendRequest(request);
              }
            }
            
            // 改造后：
            public class SecurityTransporter extends Transporter {
              //...省略其他代码..
              @Override
              public Response sendRequest(Request request) {
                if (StringUtils.isBlank(appId) || StringUtils.isBlank(appToken)) {
                  throw new NoAuthorizationRuntimeException(...);
                }
                request.addPayload("app-id", appId);
                request.addPayload("app-token", appToken);
                return super.sendRequest(request);
              }
            }
            ```

    -   分析

        -   改造后，如果传递给 demoFunction() 函数的是子类 SecurityTransporter 对象，那 demoFunction() 有可能会有异常抛出。这样，当子类替换父类传递进 demoFunction 函数后，整个程序的逻辑行为有了改变。

-   总结：

    -   虽然从定义描述和代码实现上来看，多态和里式替换有点类似，但它们**关注的角度**是不一样的。
        -   **多态**是面向对象编程的一大特性，也是面向对象编程语言的一种语法。它是一种代码实现的思路。
        -   而**里式替换**是一种设计原则，是用来指导继承关系中子类该如何设计，子类的设计要保证在替换父类的时候，不改变原有程序的逻辑以及不破坏原有程序的正确性。

### 哪些代码明显违背了 LSP?

-   里式替换原则还有另外一个更加能落地、更有指导意义的描述。那就是**“按照协议来设计”**。

    >   解读：
    >
    >   ​	**子类有设计的时候，要遵守父类的行为约定（或叫协议）。父类定义了函数的行为约定，那子类可以改变函数的内部实现逻辑，但不能改变函数原有的行为约定。**
    >
    >   ​	这里的行为约定包括：
    >
    >   ​		函数声明要实现的功能; 
    >
    >   ​		对输入、输出、异常的约定;
    >
    >   ​		甚至包括注释中所罗列的任何特殊说明。
    >
    >   ​	实际上，定义中父类和子类间的关系，也可以替换成接口和实现类间的关系。

-   为了更好的理解，我举几个违反里式替换原则的例子来解释一下。

    1.  **子类违背父类声明要实现的功能**
    2.  **子类违背父类对输入、输出、异常的约定**
    3.  **子类违背父类注释中所罗列的任何特殊说明**

    




[toc]

## 22 | 理论八：如何用迪米特法则（LOD）实现“高内聚、松耦合”？

-   今天，我们学习最后一个设计原则：迪米特法则。利用这个原则，能够帮我们实现代码的“高内聚、松耦合”。
-   今天，我们就围绕着下面几个问题，并结合两个代码实战案例来学习
    1.  什么是“高内聚、松耦合”？
    2.  如何利用迪米特法则来实现“高内聚、松耦合”？
    3.  有哪些代码设计明显违背迪米特法则的？对此又该如何重构？

### 何为“高内聚、松耦合”？

#### 到底什么是“高内聚”呢？

-   所谓高内聚，就是指相近的功能应该放在同一个类中，不相近的功能不要放到同一个类中。
-   相近的功能往往会被同时修改，放到同一个类中，修改会比较集中，代码容易维护。

#### 什么是“松耦合”？

-   所谓耦合，在代码中，类与类间的依赖关系简单清晰。
-   即使两个类有依赖关系，一个类的代码改动不会或者很少导致依赖类的代码改动。

#### “内聚”和“耦合”之间的关系

-   如下图，左边部分的代码结构是“高内聚、松耦合”; 右边部分正好相反，是“低内聚、紧耦合”。
    -   ![img](imgs/62275095f1f5817cad8a9ca129a6ec3c.jpg)

-   分析
    -   左边部分的代码设计中，
        -   类的粒度比较小，每个类的职责比较单一。
        -   相近功能都放到一个类中，不相近的功能被分割到了多个类中。
    -   右边部分的代码设计中，
        -   类的粒度比较大，低内聚，功能大而全，
        -   不相近的功能放到了一个类中，

### “迪米特法则”理论描述

-   迪米特法则（LOD）

    >   **最小知识原则**
    >
    >    
    >
    >   不该有的直接依赖关系的类之间，不要有依赖;
    >
    >   有依赖关系的类之间，尽量只依赖必要的接口。

### 理论解读与代码实战一

-   我们先来看下这条原则的前半部分，**“不该有直接依赖关系的类间，不要有依赖”**

-   示例：实现简化版的搜索引擎抓取网页的功能。

    -   代码示例

        -   ```java
            
            public class NetworkTransporter {
                // 省略属性和其他方法...
                public Byte[] send(HtmlRequest htmlRequest) {
                  //...
                }
            }
            
            public class HtmlDownloader {
              private NetworkTransporter transporter;//通过构造函数或IOC注入
              
              public Html downloadHtml(String url) {
                Byte[] rawHtml = transporter.send(new HtmlRequest(url));
                return new Html(rawHtml);
              }
            }
            
            public class Document {
              private Html html;
              private String url;
              
              public Document(String url) {
                this.url = url;
                HtmlDownloader downloader = new HtmlDownloader();
                this.html = downloader.downloadHtml(url);
              }
              //...
            }
            ```

    -   代码分析

        1.  我们来看下 **NetworkTransporter** 类。

            -   作为一个底层网络通信类，我们希望它的功能尽可能通用，而不只是服务于下载 HTML，所以，我们不应该直接依赖太具体的发送对象 HtmlRequest。

            -   代码重构

                -   ```java
                    public class NetworkTransporter {
                        // 省略属性和其他方法...
                        public Byte[] send(String address, Byte[] data) {
                          //...
                        }
                    }
                    ```

                -   

        2.  我们再看下 **HtmlDownloader** 类。

            -   这个类的设计没问题，不过，我们需要对它做相应的修改。

            -   代码重构

                -   ```java
                    
                    public class HtmlDownloader {
                      private NetworkTransporter transporter;//通过构造函数或IOC注入
                      
                      // HtmlDownloader这里也要有相应的修改
                      public Html downloadHtml(String url) {
                        HtmlRequest htmlRequest = new HtmlRequest(url);
                        Byte[] rawHtml = transporter.send(
                          htmlRequest.getAddress(), htmlRequest.getContent().getBytes());
                        return new Html(rawHtml);
                      }
                    }
                    ```

        3.  我们来看下 Document 类。

            -   问题较多，主要有如下三点：

                1.  构造函数中 downloader.downloadHtml() 逻辑复杂，耗时长，不应放到构造函数中，会影响代码的可测试性。
                2.  HtmlDownloader 对象在构造函数中通过 new 来创建，违反了基于接口而非实现编程的设计思想，也会影响到代码的可测试性。
                3.  从业务含义上来讲，Document 网页文档没必要依赖 HtmlDownloader 类，违背了迪米特法则。

            -   代码重构

                -   ```java
                    
                    public class Document {
                      private Html html;
                      private String url;
                      
                      public Document(String url, Html html) {
                        this.html = html;
                        this.url = url;
                      }
                      //...
                    }
                    
                    // 通过一个工厂方法来创建Document
                    public class DocumentFactory {
                      private HtmlDownloader downloader;
                      
                      public DocumentFactory(HtmlDownloader downloader) {
                        this.downloader = downloader;
                      }
                      
                      public Document createDocument(String url) {
                        Html html = downloader.downloadHtml(url);
                        return new Document(url, html);
                      }
                    }
                    ```

### 理论解读与代码实战二

-   我们再来看下这条原则的后半部分：**“有依赖关系的类之间，尽量只依赖必要的接口”**。

-   示例：对象的序列化和反序列化

    -   ```java
        
        public class Serialization {
          public String serialize(Object object) {
            String serializedResult = ...;
            //...
            return serializedResult;
          }
          
          public Object deserialize(String str) {
            Object deserializedResult = ...;
            //...
            return deserializedResult;
          }
        }
        ```

-   拆分

    -   我们将 Serialization 类拆分为两个更小粒度的度，一个只负责序列化，一个只负责反序列化。代码如下：

    -   ```java
        
        public class Serializer {
          public String serialize(Object object) {
            String serializedResult = ...;
            ...
            return serializedResult;
          }
        }
        
        public class Deserializer {
          public Object deserialize(String str) {
            Object deserializedResult = ...;
            ...
            return deserializedResult;
          }
        }
        ```

-   分析

    -   尽管拆分后的代码更能满足迪米特法则，但却违背了高内聚和设计思想。
    -   如果我们既不想违背高内聚的设计思想，也不想违背迪米特法则。那我们该如何解决这个问题？

-   代码重构：通过引入两个接口就能轻松解决这个问题。

    -   ```java
        
        public interface Serializable {
          String serialize(Object object);
        }
        
        public interface Deserializable {
          Object deserialize(String text);
        }
        
        public class Serialization implements Serializable, Deserializable {
          @Override
          public String serialize(Object object) {
            String serializedResult = ...;
            ...
            return serializedResult;
          }
          
          @Override
          public Object deserialize(String str) {
            Object deserializedResult = ...;
            ...
            return deserializedResult;
          }
        }
        
        public class DemoClass_1 {
          private Serializable serializer;
          
          public Demo(Serializable serializer) {
            this.serializer = serializer;
          }
          //...
        }
        
        public class DemoClass_2 {
          private Deserializable deserializer;
          
          public Demo(Deserializable deserializer) {
            this.deserializer = deserializer;
          }
          //...
        }
        ```

-   上面的代码实现思路，也体现了“基于接口而非实现编程”的设计原则，结合了迪米特法则，我们可以总结出一条新的设计原则，那就是“基于最小接口而非最大实现编程”。

### 辩证思考与灵活应用

-   对于实战二最终的设计思路，
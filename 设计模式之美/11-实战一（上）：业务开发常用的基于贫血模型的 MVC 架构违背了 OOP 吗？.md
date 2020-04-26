[toc]

## 11 | 实战一（上）：业务开发常用的基于贫血模型的 MVC 架构违背了 OOP 吗？

-   在正式进入实战项目的讲解前，我先带你搞清楚下面几个问题

    >   什么是贫血模型？什么是充血模型？
    >
    >   为什么说基于贫血模型的传统开发模式违反了 OOP?
    >
    >   基于贫血模型的传统开发模式既然违反了 OOP，那为何如此流行？
    >
    >   什么情况下我们应该考虑使用基于充血模型的 DDD 开发模式？

### 什么是基于贫血模型的传统开发模式？

-   什么是 MVC 三层架构。

    >   M 表示 Model 
    >
    >   V 表示 View
    >
    >   C 表示 Controller

-   什么是贫血模型。

    -   代码示例：

        -   ```java
            
            ////////// Controller+VO(View Object) //////////
            public class UserController {
              private UserService userService; //通过构造函数或者IOC框架注入
              
              public UserVo getUserById(Long userId) {
                UserBo userBo = userService.getUserById(userId);
                UserVo userVo = [...convert userBo to userVo...];
                return userVo;
              }
            }
            
            public class UserVo {//省略其他属性、get/set/construct方法
              private Long id;
              private String name;
              private String cellphone;
            }
            
            ////////// Service+BO(Business Object) //////////
            public class UserService {
              private UserRepository userRepository; //通过构造函数或者IOC框架注入
              
              public UserBo getUserById(Long userId) {
                UserEntity userEntity = userRepository.getUserById(userId);
                UserBo userBo = [...convert userEntity to userBo...];
                return userBo;
              }
            }
            
            public class UserBo {//省略其他属性、get/set/construct方法
              private Long id;
              private String name;
              private String cellphone;
            }
            
            ////////// Repository+Entity //////////
            public class UserRepository {
              public UserEntity getUserById(Long userId) { //... }
            }
            
            public class UserEntity {//省略其他属性、get/set/construct方法
              private Long id;
              private String name;
              private String cellphone;
            }
            ```

    -   定义

        >   Service 层的数据和业务逻辑，被分割为 BO 和 Service 两个类中，像 UserBO 这样，只包含数据，不包含业务逻辑的类，就叫作**贫血模型**,

    -   贫血模型将数据与操作分离，破坏了面向对象的封装特性，是一种典型的面向过程的编程风格。

### 什么是基于充血模型的 DDD 开发模式？

#### 首先，我们先来看一下，什么是充血模型？

-   数据和对应的业务逻辑被封装到同一个类中。
-   因此，这种充血模型满足面向对象的封装特性，是典型的面向对象编程风格。

#### 接下来，我们再来看一下，什么是领域驱动设计？

-   即 DDD，主要是用来指导如何解耦业务系统，划分业务模块，定义业务领域模型及其交互。
-   不要花太多时间过度去研究它。

#### 代码设计

-   基于充血模型的 DDD 开发模式实现的代码，也是按照 MVC 三层架构分层的。及 Controller 层、Repository 层、Service 层。与贫血模型的区别主要在 Service 层。
    1.  基于贫血模型的传统开发模式中，Service 层包含 Service 类和 BO 类两部分，BO 只包含数据，不包含业务逻辑。业务逻辑集中在 Service 类中。
    2.  在基于充血模型的 DDD 开发模式中，Service 层包含 Service 类和 Domain 类两部分。Domain 相当于贫血模型中的 BO。不过。Domain 与 BO 的区别在于它是基于充血模型开发的。既包含数据，也包含业务逻辑。而Service 类变得非常单薄。
-   总结：
    1.  基于贫血模型的传统开发模式：**重 Service 轻 BO**。
    2.  基于充血模型的 DDD 开发模式：**轻 Service 重 Domain**。

### 为什么基于贫血模型的传统开发模式如此受欢迎？

1.  大部分情况下，**我们开发的系统业务可能都比较简单**，简单到就是基于SQL 的 CRUD 操作。所以，我们根本不需要动脑子精心设计充血模型，贫血模型就足以就会这种简单的业务开发工作了。
2.  **充血模型的设计要比贫血模型更加有难度。**
3.  **思维已固化，转型有成本。**

### 什么项目应该考虑使用基于充血模型的 DDD 开发模式？

1.  区别一
    -   基于贫血模型的传统开发模式，比较适合业务比较简单的系统开发。
    -   基于充血模型的 DDD 开发模式，比较适合业务比较复杂的开发。
2.  区别二（非常重要），开发流程
    -   基于贫血模型的传统开发模式，大部分是 SQL 驱动的开发模式。SQL 都是针对特定的业务功能编写的，复用性差。
    -   基于充血模型的 DDD 开发模式。我们需要先理清所有的业务，定义领域模型所包含的属性和方法。领域模型相当于可复用的业务中间层。新功能需求的开发，都基于之前定义好的这些领域模型来完成。
3.  总结
    -   越复杂的系统，对代码的复用性、易维护性要求就越高，我们就越应该花更多的时间和精力在前期设计上。而基于充血模型的 DDD 开发模式，正好需要我们前期做大量的业务调研、领域模型设计，所以它更适合这种复杂系统的开发。

### 重点回顾
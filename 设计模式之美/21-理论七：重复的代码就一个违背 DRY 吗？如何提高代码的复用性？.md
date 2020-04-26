[toc]

## 21 | 理论七：重复的代码就一个违背 DRY 吗？如何提高代码的复用性？

-   今天，我们再学习一个原则：DRY 原则（不要写重复的代码）。

### DRY 原则

#### 实现逻辑重复

-   代码示例（用户认证）

    -   ```java
        
        public class UserAuthenticator {
          public void authenticate(String username, String password) {
            if (!isValidUsername(username)) {
              // ...throw InvalidUsernameException...
            }
            if (!isValidPassword(password)) {
              // ...throw InvalidPasswordException...
            }
            //...省略其他代码...
          }
        
          private boolean isValidUsername(String username) {
            // check not null, not empty
            if (StringUtils.isBlank(username)) {
              return false;
            }
            // check length: 4~64
            int length = username.length();
            if (length < 4 || length > 64) {
              return false;
            }
            // contains only lowcase characters
            if (!StringUtils.isAllLowerCase(username)) {
              return false;
            }
            // contains only a~z,0~9,dot
            for (int i = 0; i < length; ++i) {
              char c = username.charAt(i);
              if (!(c >= 'a' && c <= 'z') || (c >= '0' && c <= '9') || c == '.') {
                return false;
              }
            }
            return true;
          }
        
          private boolean isValidPassword(String password) {
            // check not null, not empty
            if (StringUtils.isBlank(password)) {
              return false;
            }
            // check length: 4~64
            int length = password.length();
            if (length < 4 || length > 64) {
              return false;
            }
            // contains only lowcase characters
            if (!StringUtils.isAllLowerCase(password)) {
              return false;
            }
            // contains only a~z,0~9,dot
            for (int i = 0; i < length; ++i) {
              char c = password.charAt(i);
              if (!(c >= 'a' && c <= 'z') || (c >= '0' && c <= '9') || c == '.') {
                return false;
              }
            }
            return true;
          }
        }
        ```

-   代码重构

    -   ```java
        
        public class UserAuthenticatorV2 {
        
          public void authenticate(String userName, String password) {
            if (!isValidUsernameOrPassword(userName)) {
              // ...throw InvalidUsernameException...
            }
        
            if (!isValidUsernameOrPassword(password)) {
              // ...throw InvalidPasswordException...
            }
          }
        
          private boolean isValidUsernameOrPassword(String usernameOrPassword) {
            //省略实现逻辑
            //跟原来的isValidUsername()或isValidPassword()的实现逻辑一样...
            return true;
          }
        }
        ```

-   分析
    -   合并后的 isValidUsernameOrPassword 函数，负责两件事：验证用户名和验证密码。违反了“单一职责原则”和“接口隔离原则”。
    -   因为 isValidUsername 和 isValidPassword 两个函数，虽然从代码实现逻辑上看起来是重复的，但从语义上并不重复（功能不一样：一个是校验用户名，另一个是校验密码）。
-   小结
    -   尽管代码的实现逻辑不同，但语义不同，我们判定它并不违反 DRY 原则。
    -   对于包含重复代码的问题，我们可以通过抽象更细粒度函数的方式来解决。

####功能语义重复

-   代码示例

    -   ```java
        
        public boolean isValidIp(String ipAddress) {
          if (StringUtils.isBlank(ipAddress)) return false;
          String regex = "^(1\\d{2}|2[0-4]\\d|25[0-5]|[1-9]\\d|[1-9])\\."
                  + "(1\\d{2}|2[0-4]\\d|25[0-5]|[1-9]\\d|\\d)\\."
                  + "(1\\d{2}|2[0-4]\\d|25[0-5]|[1-9]\\d|\\d)\\."
                  + "(1\\d{2}|2[0-4]\\d|25[0-5]|[1-9]\\d|\\d)$";
          return ipAddress.matches(regex);
        }
        
        public boolean checkIfIpValid(String ipAddress) {
          if (StringUtils.isBlank(ipAddress)) return false;
          String[] ipUnits = StringUtils.split(ipAddress, '.');
          if (ipUnits.length != 4) {
            return false;
          }
          for (int i = 0; i < 4; ++i) {
            int ipUnitIntValue;
            try {
              ipUnitIntValue = Integer.parseInt(ipUnits[i]);
            } catch (NumberFormatException e) {
              return false;
            }
            if (ipUnitIntValue < 0 || ipUnitIntValue > 255) {
              return false;
            }
            if (i == 0 && ipUnitIntValue == 0) {
              return false;
            }
          }
          return true;
        }
        ```

-   分析
    -   isValidlp() 和 checklftpValid()两个函数，尽管两个函数命名不同，实现逻辑不同，但功能是相同的，都是用来判定 IP 地址是否合法的。 
    -   这个例子中，尽管两段代码的实现逻辑不重复，但语义重复，也就是功能重复，我们判定它违反了 DRY 原则。

#### 代码执行重复

-   代码示例

    -   ```java
        
        public class UserService {
          private UserRepo userRepo;//通过依赖注入或者IOC框架注入
        
          public User login(String email, String password) {
            boolean existed = userRepo.checkIfUserExisted(email, password);
            if (!existed) {
              // ... throw AuthenticationFailureException...
            }
            User user = userRepo.getUserByEmail(email);
            return user;
          }
        }
        
        public class UserRepo {
          public boolean checkIfUserExisted(String email, String password) {
            if (!EmailValidation.validate(email)) {
              // ... throw InvalidEmailException...
            }
        
            if (!PasswordValidation.validate(password)) {
              // ... throw InvalidPasswordException...
            }
        
            //...query db to check if email&password exists...
          }
        
          public User getUserByEmail(String email) {
            if (!EmailValidation.validate(email)) {
              // ... throw InvalidEmailException...
            }
            //...query db to get user by email...
          }
        }
        ```

-   分析

    1.  上面这段代码，既没有逻辑重复，也没有语义重复，但仍然违反了 DRY 原则。这是因为，代码中存在“执行重复”。
        -   在 login() 函数中，email 的校验逻辑被执行了两次。一次是在调用 checkIfUserExisted() 函数的时候，一次是在调用 getUserByEmail() 函数，

-   代码重构

    -   ```java
        
        public class UserService {
          private UserRepo userRepo;//通过依赖注入或者IOC框架注入
        
          public User login(String email, String password) {
            if (!EmailValidation.validate(email)) {
              // ... throw InvalidEmailException...
            }
            if (!PasswordValidation.validate(password)) {
              // ... throw InvalidPasswordException...
            }
            User user = userRepo.getUserByEmail(email);
            if (user == null || !password.equals(user.getPassword()) {
              // ... throw AuthenticationFailureException...
            }
            return user;
          }
        }
        
        public class UserRepo {
          public boolean checkIfUserExisted(String email, String password) {
            //...query db to check if email&password exists
          }
        
          public User getUserByEmail(String email) {
            //...query db to get user by email...
          }
        }
        ```

    -   

### 代码复用性

-   今天，我们深入地学习一下“代码的复用性”这个知识点。

#### 什么是代码的复用性？

-   我们先来区分一下概念：

    -   **代码复用性**
        -   表示一种行为：我们在开发新功能的时候，尽量复用已经存在的代码
    -   **代码复用**
        -   表示一段代码可被复用的特性或能力：我们在编写代码的时候，让代码尽量可复用。
    -   **DRY 原则**
        -   是一条原则：不要写重复的代码。
-   首先，**“不重复”并不代表“可复用”**。
    -   在一个项目代码中，可能不存在任何重复的代码，但也并不表示里有可复用的代码。
-   其次，**“复用”和“可复用性”关注的角度不同**。
    -   代码“可复用性”是从代码开发者的角度来讲的，“复用”是从代码使用者的角度来讲的。

#### 怎么提高代码的复用性？

1.  **减少代码耦合**
2.  **满足单一职责原则**
3.  **模块化**
4.  **业务与非业务逻辑分离**
5.  **通用代码下沉**
6.  **继承、多态、抽象、封装**
7.  **应用模板等设计模式**

#### 辨证思考和灵活应用

-   除非的非常明确的复用需求，否则，为了暂时用不到的复用需求，花费太多的时间、精力，投入太多的开发成本，并不是一个值得推荐的做法。
-   也就是说，第一次编写代码的时候，我们不考虑复用性;第二次遇到复用场景的时候，再进行重构使其复用。

### 重点回顾
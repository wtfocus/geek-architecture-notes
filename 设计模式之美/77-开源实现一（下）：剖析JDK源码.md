[toc]

## 77 | 开源实战一（下）：剖析 Java JDK 源码

### 模板模式在 Collections 类中的应用

-   Java 中的 Collections 类的 sort() 函数就是利用模板模式的这个扩展特性。

-   Collections.sort() 函数使用：

    -   ```java
        
        public class Demo {
          public static void main(String[] args) {
            List<Student> students = new ArrayList<>();
            students.add(new Student("Alice", 19, 89.0f));
            students.add(new Student("Peter", 20, 78.0f));
            students.add(new Student("Leo", 18, 99.0f));
        
            Collections.sort(students, new AgeAscComparator());
            print(students);
            
            Collections.sort(students, new NameAscComparator());
            print(students);
            
            Collections.sort(students, new ScoreDescComparator());
            print(students);
          }
        
          public static void print(List<Student> students) {
            for (Student s : students) {
              System.out.println(s.getName() + " " + s.getAge() + " " + s.getScore());
            }
          }
        
          public static class AgeAscComparator implements Comparator<Student> {
            @Override
            public int compare(Student o1, Student o2) {
              return o1.getAge() - o2.getAge();
            }
          }
        
          public static class NameAscComparator implements Comparator<Student> {
            @Override
            public int compare(Student o1, Student o2) {
              return o1.getName().compareTo(o2.getName());
            }
          }
        
          public static class ScoreDescComparator implements Comparator<Student> {
            @Override
            public int compare(Student o1, Student o2) {
              if (Math.abs(o1.getScore() - o2.getScore()) < 0.001) {
                return 0;
              } else if (o1.getScore() < o2.getScore()) {
                return 1;
              } else {
                return -1;
              }
            }
          }
        }
        ```

-   Collections.sort() 实现了对集合的排序。为了扩展性，它将其中“比较大小”这部分逻辑，委派给用户来实现。

-   如果我们把比较大小这部分逻辑看作整个排序逻辑的其中一个步骤，那我们就可以把它看作模板模式。

### 观察者模式在 JDK 中的应用

-   Java JDK 也提供了观察者模式的简单框架实现，包含两个类：java.util.Observable 和 java.util.Observer。

    -   ```java
        
        public interface Observer {
            void update(Observable o, Object arg);
        }
        
        public class Observable {
            private boolean changed = false;
            private Vector<Observer> obs;
        
            public Observable() {
                obs = new Vector<>();
            }
        
            public synchronized void addObserver(Observer o) {
                if (o == null)
                    throw new NullPointerException();
                if (!obs.contains(o)) {
                    obs.addElement(o);
                }
            }
        
            public synchronized void deleteObserver(Observer o) {
                obs.removeElement(o);
            }
        
            public void notifyObservers() {
                notifyObservers(null);
            }
        
            public void notifyObservers(Object arg) {
                Object[] arrLocal;
        
                synchronized (this) {
                    if (!changed)
                        return;
                    arrLocal = obs.toArray();
                    clearChanged();
                }
        
                for (int i = arrLocal.length-1; i>=0; i--)
                    ((Observer)arrLocal[i]).update(this, arg);
            }
        
            public synchronized void deleteObservers() {
                obs.removeAllElements();
            }
        
            protected synchronized void setChanged() {
                changed = true;
            }
        
            protected synchronized void clearChanged() {
                changed = false;
            }
        }
        ```

#### changed 成员变量

-   它用来表明被观察者（Observable）有没有状态更新。
-   当有状态更新时，我们需手动调用 setChanged() 函数，将 changed 变量设置为 true。这样才能在调用 notifyObserver() 函数的时候，真正触发观察者（Observer）执行 update() 函数。

#### notifyObserver() 函数

-   notifyObserver() 函数出于性能考虑。采用了一种折中的方案，这个方案类似于我们之前讲过的让迭代器支持“快照”的解决方案。
-   我们只需要对拷贝创建快照的过程加锁，加锁范围减少了很多，并发性能提高了。

### 单例模式在 Runtime 类中的应用

-   每个 Java 应用在运行时会启动一个 JVM 进程，每个 JVM 进程都只对应一个 Runtime 实例，用于查看 JVM 状态以及控制 JVM 行为。进程内唯一，所以比较适合设计为单例。

-   如果要去实例化一个 Runtime 对象，只能通过 getRumtime() 静态方法来获得。

-   Runtime 类的代码实现如下所示：

    -   ```java
        
        /**
         * Every Java application has a single instance of class
         * <code>Runtime</code> that allows the application to interface with
         * the environment in which the application is running. The current
         * runtime can be obtained from the <code>getRuntime</code> method.
         * <p>
         * An application cannot create its own instance of this class.
         *
         * @author  unascribed
         * @see     java.lang.Runtime#getRuntime()
         * @since   JDK1.0
         */
        public class Runtime {
          private static Runtime currentRuntime = new Runtime();
        
          public static Runtime getRuntime() {
            return currentRuntime;
          }
          
          /** Don't let anyone else instantiate this class */
          private Runtime() {}
          
          //....
          public void addShutdownHook(Thread hook) {
            SecurityManager sm = System.getSecurityManager();
            if (sm != null) {
               sm.checkPermission(new RuntimePermission("shutdownHooks"));
            }
            ApplicationShutdownHooks.add(hook);
          }
          //...
        }
        ```

### 其他模式在 JDK 中的应用汇总

-   **模板模式**，Java Servlet、JUnit TestCase、Java InputStream、Java AbstractList。
-   **享元模式**，Integer 类中 -128～127 之间的整型对象是可以复用的。
-   **职责链模式**，Java Servlet 中的 Filter、Spring 中的 interceptor。拦截器、过滤器这些功能绝大部分都是采用职责链模式来实现的。
-   **迭代器模式**，我们重点剖析了 Java 中的 Iterator 迭代器的实现。

### 重点回顾

-   **在真实的项目开发中， 如何灵活应用设计模式，做到活学活用，能够根据具体的场景、需求，做灵活的设计和实现上的调整**。这也是新手和老手最大的区别。
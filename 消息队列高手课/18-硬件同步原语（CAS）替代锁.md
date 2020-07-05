[toc]

## 18 | 如何用硬件同步原语 （CAS）替代锁？

### 什么是硬件同步原语？

1.  硬件同步原语，**是由计算机硬件提供的一组原子操作，常用的有 CAS 和 FAA**。

2.  CAS（Compare and Swap），先比较，再交换

    -   ```c++
        
        << atomic >>
        function cas(p : pointer to int, old : int, new : int) returns bool {
            if *p ≠ old {
                return false
            }
            *p ← new
            return true
        }
        ```

3.  FAA（Fetch and Add）

    -   ```C++ 
        
        << atomic >>
        function faa(p : pointer to int, inc : int) returns int {
            int value <- *location
            *p <- value + inc
            return value
        }
        ```

4.  **原子操作具有不可分割性，也就不存在并发的问题**。

### CAS 版本的账户服务

1.  看下如何使用 CAS 原语来保证数据的安全性。

2.  使用锁版本，实现转账服务

    -   ```go
        
        package main
        
        import (
          "fmt"
          "sync"
        )
        
        func main() {
          // 账户初始值为0元
          var balance int32
          balance = int32(0)
          done := make(chan bool)
          // 执行10000次转账，每次转入1元
          count := 10000
        
          var lock sync.Mutex
        
          for i := 0; i < count; i++ {
            // 这里模拟异步并发转账
            go transfer(&balance, 1, done, &lock)
          }
          // 等待所有转账都完成
          for i := 0; i < count; i++ {
            <-done
          }
          // 打印账户余额
          fmt.Printf("balance = %d \n", balance)
        }
        // 转账服务
        func transfer(balance *int32, amount int, done chan bool, lock *sync.Mutex) {
          lock.Lock()
          *balance = *balance + int32(amount)
          lock.Unlock()
          done <- true
        }
        ```

3.  CAS 原语版本

    -   ```go
        
        func transferCas(balance *int32, amount int, done chan bool) {
          for {
            old := atomic.LoadInt32(balance)
            new := old + int32(amount)
            if atomic.CompareAndSwapInt32(balance, old, new) {
              break
            }
          }
          done <- true
        }
        ```

4.  FAA 原语版本

    -   ```go
        
        func transferFaa(balance *int32, amount int, done chan bool) {
          atomic.AddInt32(balance, int32(amount))
          done <- true
        }
        ```

5.  场景：

    -   CAS 适用范围更广些。无论计算是什么样的，都可以使用 CAS 原语来保护数据安全。
    -   FAA 只能局限于简单的加减法。

6.  CAS 

    -   反重重试赋值
    -   比较耗费 CPU 资源
        -   缓解方法，使用 Yield()。
    -   适合于线程间碰撞不太频繁

### 小结

1.  原语：由 CPU 提供的原子操作，在并发环境中，单独使用这些原语不用担心数据安全问题。
    -   CAS
    -   FAA
2.  特定场景中，CAS 原语可以替代锁，在保证安全性的同时，提供比锁更好的性能。


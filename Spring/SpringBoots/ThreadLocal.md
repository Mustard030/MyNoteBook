总结：
- 线程并发：一般在多线程并发的场景下用到，单线程也可以用但是没必要
- 传递数据：通过ThreadLocal在同一线程的不同组件中传递数据，如TraceID的实现
- 线程隔离：不同线程中的变量互相独立，不会互相影响（这里有坑：ThreadLocal是线程池实现的，线程复用的时候可能会读取到以前的值，所以每次用完后记得清除）


**用法：**

| 方法                        | 描述              |
| ------------------------- | --------------- |
| ThreadLocal()             | 创建ThreadLocal对象 |
| public void set (T value) | 设置当前线程绑定的局部变量   |
| public T get()            | 获取当前线程绑定的局部变量   |
| public void remove()      | 移除当前线程绑定的局部变量   |

# 底层设计
在JDK8后，每个Thread维护一个ThreadLocalMap，这个Map的key是ThreadLocal实例本身，value才是真正要存储的值Object。
具体过程：
1. 每个Thread线程内部有个Map
2. Map里面存储ThreadLocal对象（key）和线程的变量副本（value）
3. Thread内部的Map是由ThreadLocal维护的，有ThreadLocal负责向map获取和设置线程的变量值
4. 对于不同的线程，每次获取副本值时，别的线程并不能获取到当前线程的副本值，形成了隔离，互不干扰
## JVM的结构
1. 类加载器  
   在JVM启动时或者在类运行时将需要的class加载到JVM中

2. 执行引擎  
   负责执行class文件中包含的字节码指令

3. 运行时数据区  
   a. 方法区(Method Area)：用于存储类结构信息的地方，包括常量池、静态变量、构造函数等。

   b. 堆：存储java实例或者对象的地方。

   c. java栈(Stack)：java栈总是和线程关联在一起，每当创建一个线程时，JVM就会为这个线程创建一个对应的java栈。在这个java栈中又会包含多个栈帧，每运行一个方法就创建一个栈帧，用于存储局部变量表、操作栈、方法返回值等。每一个方法从调用直至执行完成的过程，就对应一个栈帧在java栈中入栈到出栈的过程。所以java栈是现成私有的。

   d. 程序计数器：用于保存当前线程执行的内存地址

   e. 本地方法栈：为Java的本地方法服务

## 内存泄露和内存溢出  
内存泄露：程序在申请内存后，无法释放已申请的空间，内存泄漏累积的后果就是内存溢出  

内存溢出：程序申请内存是，没有足够的内存提供给申请者使用。
## callalbe和runnable的相同点和区别
callable和runnable都是接口，都需要使用thread.start()来启动

实现Callable接口的线程能返回执行结果，而实现Runnable接口的线程不能返回结果；  
Callable接口的call方法允许抛出异常，而Runnable接口的run方法的异常只能内部消化，不能继续向上抛；


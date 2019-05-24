1. 是否保证线程安全  
   ArrayList和LinkedList都是不同步的，无法保证线程安全

2. 底层数据结构  
   ArrayList底层使用的是数组存储，LinkedList底层使用的是双向链表

3. 插入和删除是否受元素位置的影响  
   - ArrayList采用数组存储，所以插入和删除操作的时间复杂度都受元素位置的影响。
   - LinkedList采用链表存储，所以插入和删除不受元素位置的影响

4. 是否支持随机访问  
   ArrayList明显支持随机访问，而LinkedList不支持随机访问

5. 内存占用  
   ArrayList的空间浪费主要体现在会预留一定的容量空间，而LinkedList的空间浪费主要体现在每个元素都需要消耗比ArrayList更多的空间（因为要存放节点的前驱和后继）

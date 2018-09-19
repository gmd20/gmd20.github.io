```text
之前不知道iterator 还可以这样使用啊，搞到我想自己写了类似Linux内核的一个list的实现。

可以把 list<Request*>::iterator   其实就是链表节点来的，整个list的其他元素的添加删除不会影响到你这个iterator 的。所以插入元素到list容器的时候，顺便把:iterator  这个也保存到元素自己一个子变量。然后需要删除的时候，根据iterator来删除就可以了。

vector那些的iterator 类似普通的数组指针，但vector会自动重新分配内存，所以插入元素字号后，以前的iterator 可能无效了，但list应该没有这个问题。 这个 stl 不理解底下实现细节，也不能写出好看的代码来啊。像不同的容器 的iterator  的行为就差别很大，最好写个简单代码，跟踪一下，看看到底都怎么实现的才行。


下面 的示例代码，用list来做queue使用的。
struct Request {
 list<Request*>::iterator  pos;
 int data;
};
----------
  list<Request *> queue;
  for( int i=0; i<10; i++ ) {
  Request * req = new Request();
  req->data = i;
  queue.push_back (req);
  req->pos = --queue.end();
  }

  list<Request*>::iterator  list_it =  queue.begin();
 
  while(list_it != queue.end()) {
  Request * req = *list_it;
 if (req->data  == 8 )
 { 
  queue.erase(req->pos);
  break;
 }  
 list_it ++;
  }
   
  list_it =  queue.begin(); 
  while(list_it != queue.end()) {
  Request * req = *list_it;
  cout << " " << req->data;
  list_it ++;
  }
阅读(928)| 评论(0)
```

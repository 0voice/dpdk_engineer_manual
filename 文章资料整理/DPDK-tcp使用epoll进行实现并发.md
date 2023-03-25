# DPDK-tcp使用epoll进行实现并发

## 1.tcp使用epoll进行实现并发

tcp 服务器编写的步骤都是很熟悉的了。tcp由于每一个新的连接都会新建一个socket和客户端进行通信，但是新建立的连接在很多次之后就会管理就会出现问题，这个时候就可以使用epoll进行管理。
epoll是一种多路转接io，相比selete和poll在管理大量描述符的时候优势很明显。具体的优势我们后面可以慢慢说。
epoll的流程， 创建epoll描述符–> 添加事件–> wait;

```c++
 int epoll_create (int __size)
 __size: 这个值在早期时候用于确定返回列表的预留长度，现在由于返回的内容放置在一个双向链表中，这个		  实际上已经没什么作用了
 return 返回新创建的epoll描述符

```

```c++
extern int epoll_ctl (int __epfd, int __op, int __fd,
		      struct epoll_event *__event) 
__epfd: epoll的文件描述符
_op	:	EPOLL_CTL_ADD 添加监视节点， EPOLL_CTL_DEL 删除监视 ， EPOLL_CTL_MOD 修改监视
__fd： 所关注的文件描述符
__event：关心的事件节点填充
    
typedef union epoll_data
{
  void *ptr; //预留的指针
  int fd;	//一般设置为当前的文件描述符
  uint32_t u32; //一般不用
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;	/* Epoll events */  //在这里设置所关心的事件
  epoll_data_t data;	/* User data variable */
} 

```

```c++
extern int epoll_wait (int __epfd, struct epoll_event *__events,
		       int __maxevents, int __timeout);
参数分别为： epoll文件描述符，返回事件存储的位置，数组的最大长度。超时时间， -1 阻塞等待有事件就返回，0不管有无事件都直接返回 ， 大于0， 等待·超时时间。
return 小于零 出错， 大于0 事件发生的个数

```

## 2.epoll的两种模式：

ET：边沿触发， 只触发一次 ， 只有文件描述符从不可读变为可读的时候才会被触发， EPOLLET 这个宏用于设置ET模式
LT：水平触发， 一直触发
这两种模式都是在调用epol_ctl添加事件之前进行设置的，epoll默认是使用LT模式，在高版本中的系统中没用提供设置LT模式的宏。

### 2.1这个触发是什么意思呢？

假如说使用epoll监控tcp的listen套接字并且监控可读事件（可读事件代表有新的连接建立），如果我们设置LT模式，当新的连接到来的时候epoll的返回的事件数组中肯定会有listen描述符告诉我们可读事件发生了。但是此时如果我们没用获取新的连接，那么返回的文件描述符中还会有listen描述符。
如果是ET模式那么在返回的事件列表中就会没有。

可以看下述代码：

```c++
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <error.h>
#include <stdlib.h>
#include <unistd.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <errno.h>


int main(int argc, char *argv[]){
    int socket_fd  = socket(AF_INET, SOCK_STREAM, 0); //创建流套接字
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(struct sockaddr_in));

    addr.sin_family = AF_INET; //使用IPV4
    addr.sin_port = htons(8888); //绑定端口 8888
    addr.sin_addr.s_addr = INADDR_ANY; //绑定本机的所有网卡

    if(bind(socket_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0){
        exit(-1);
    }

    /*全连接队列为 5 或者 全连接和半连接和为5 和版本有关*/
    if(listen(socket_fd, 5) < 0){
        exit(-1);        
    }

    int epfd = epoll_create(1);//只有0和1的区别，和之前epool实现有关系
    struct epoll_event ev, event[1024];
    ev.events = EPOLLIN; //监控可读事件
    ev.data.fd = socket_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, socket_fd, &ev); //添加事件

    while(1){
        int nready = epoll_wait(epfd, event, 1024, -1); //fd和过期时间, -1 相当于有数据就返回, 
        if(nready < -1){                                // 0那么就直接返回，  返回值为 正数字，
            break;                                      // 那么就是所有的事件
        }

        char buff[1024] = {0};
        for(int i = 0 ; i < nready; i++){ //遍历返回事件
            if(event[i].data.fd == socket_fd){ //如果是监听描述符那么就建立新的连接
                struct sockaddr_in addrs_client;
                int clinet_size = sizeof(struct sockaddr_in);
                int clientfd = accept(socket_fd, (struct sockaddr *)&addrs_client, (socklen_t*)&clinet_size);
                if(clientfd < 0){ //连接建立失败
                    continue;
                }
                    //默认是LT , 没有定义EPOLLT, 设置为LT模式
                ev.events = EPOLLIN;   //EPPOLL 默认是LT， LT是水平触发，有数据就会一直触发。ET是边缘触发，是从没有数据变成有数据才 
               // ev.events = EPOLLIN | EPOLLET; // EPOLLET 设置为ET模式
                ev.data.fd = clientfd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, clientfd, &ev); //将获取到的新连接加入到epoll中
            }else{
                int clientfd = event[i].data.fd; 	//获取一个连接
                int ret = recv(clientfd, buff, 1024, 0); //非阻塞如果没有数据那么就返回-1
                if(ret < 0){//现在是不可能-1的，但是如果一直while读，读空了就会返回-1
                    if(errno == EAGAIN || errno == EWOULDBLOCK){
                        continue; //资源暂不可用， 在尝试一次
                    }else{
                        //出错
                        close(clientfd);
                        ev.events = EPOLLIN;
                        ev.data.fd = clientfd;
                        epoll_ctl(socket_fd, EPOLL_CTL_DEL,clientfd, &ev);
                    }
                }
                else if(ret == 0){ // recv 返回0代表连接已经断开
                    printf("连接被关闭%d\n",clientfd);
                    close(clientfd);
                    ev.events = EPOLLIN;
                    ev.data.fd = clientfd;
                    epoll_ctl(socket_fd, EPOLL_CTL_DEL,clientfd, &ev);                
                }
                else{
                    printf("接收到数据%s\n ", buff);
                    ret = send(clientfd, buff, ret, 0);
                    if(ret < 0){
                        printf("发送失败 %d \n" , ret);
                    }
                }
            }
        }
    }
    
    return 0;
}

```

上述代码流程：创建了一个tcp的套接字，创建了一个epoll，然后将tpc套接字的文件描述符放到了epoll中，通过fd值的不同来区分是否是listen描述符，如果是listen的文件描述符那么就获取新连接并将新连接当道epoll中。如果是普通的文件描述符那么就获取发送的消息，回复消息。
在对端关闭的时候recv会返回0；

客户端可以用下面的代码:

```c++
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <error.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include <string>
#include <iostream>


int main(int argc, char *argv[]){
    int port = 8888; // default
    if(argc == 2){
        port = atoi(argv[1]);
    }

    int socket_fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if(socket_fd < 0){
        printf("socket create err  ret = %d\n", socket_fd);
        exit(-1);
    }

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    
   int ret = connect(socket_fd, (struct sockaddr *)&addr, sizeof(addr));
    if(ret != 0){
        printf("连接失败 %s \n");
        exit(1);
    }
    
    while(1){
        char buff[1024];
        std::string buf;
        std::cin >> buf;
        int sendsize = send(socket_fd, buf.c_str(), buf.size(), 0);
        int ret = recv(socket_fd, buff, 1024, 0);
        printf("recv % s\n", buff);
    }
   

    return 0;
}

```

通过C++ 不用链接任何库就可以编过，有兴趣的可以吧代码沾出来跑一跑

上面这份代码listent的fd和连接的fd每次都要进行判断代码十分的不好看，能不能改一下呢？
我们可以通过回调的函数实现，让整个结构更加好看一些

```C++
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <error.h>
#include <stdlib.h>
#include <unistd.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <errno.h>
int epfd;
// 1. sockaddr 和 sockaddrin 区别
/**
 * 实现一个功能，如果我们收到了一条消息，我们要进行回复如何实现呢？
 * 需要考虑的问题：如果发送缓冲区满了怎么办，这个时候可以通过epoll监控
 * 文件的可写属性
 **/

struct sockitem{
    int sockfd;
    int (*call_back)(int fd, int event, void *arg);
};

int recv_back(int fd, int event, void *arg);
int send_backcall(int fd, int event, void *arg){
    struct sockitem *item = (struct sockitem*)arg;
    const char *sendContent = "nihao\n";
    int ret = send(item->sockfd, "nihao\n", strlen(sendContent), 0);
    printf("send size : %d \n", ret);

    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.ptr = item;
    item->sockfd = fd;
    it->call_back = recv_back;
    epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
    return 0;
}

int recv_back(int fd, int event, void *arg)
{
    char buff[1024] = {0};
    struct sockitem *item = (struct sockitem*)arg;
    int ret = recv(fd, buff, 1024, 0);
    struct epoll_event ev;
    if(ret <= 0){
        if(errno == EAGAIN || errno == EWOULDBLOCK){
            return -1;
        }else{
                // 连接出错了
        }

        ret = epoll_ctl(epfd, EPOLL_CTL_DEL, fd, &ev);
        close(fd);
        free(item);
        printf("epoll_ctl_del ret = %d\n", ret);
        return -2;
    }
    else{
        printf("fd = %d , recv messafe = %s\n", fd, buff);

        item->sockfd = fd;
        item->call_back = send_backcall;
        ev.events = EPOLLOUT | EPOLLET; //设置为ET
        ev.data.ptr = item;
        epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
        return 0;
    }
}

int connect_back(int fd, int event, void *arg){
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(sockaddr_in));
    socklen_t addrlen = sizeof(struct sockaddr_in);
    int clientfd = accept(fd, (struct sockaddr*)&addr, &addrlen);
    if(fd < 0){
        printf("accept err \n");
        return -1;
    }

    struct epoll_event ev;

  //  ev.data.u32 = 1000;
    struct sockitem* it = (struct sockitem*)malloc(sizeof(struct sockitem));
    if(it == NULL){
        return -2;
    }
    memset(it, 0, sizeof(struct sockitem));

    it->call_back = recv_back;
    it->sockfd = clientfd;
    ev.data.ptr = it;
    ev.events = EPOLLIN;
   // ev.data.fd = clientfd;

    epoll_ctl(epfd, EPOLL_CTL_ADD, clientfd, &ev);
    return 0;
}

int main(int argc, char *argv[]){
    int socket_fd  = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(struct sockaddr_in));

    addr.sin_family = AF_INET; //使用IPV4
    addr.sin_port = htons(8888); //绑定端口 8888
    addr.sin_addr.s_addr = INADDR_ANY; //绑定本机的所有网卡

    if(bind(socket_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0){
        exit(-1);
    }

    /*全连接队列为 5 或者 全连接和半连接和为5 和版本有关*/
    if(listen(socket_fd, 5) < 0){
        exit(-1);
    }

    epfd = epoll_create(1);//只有0和1的区别，和之前epool实现有关系
    struct epoll_event ev, event[1024];
    sockitem * it = (sockitem*)malloc(sizeof(struct sockitem));
    memset(it, 0, sizeof(struct sockitem));

    if(it == NULL){

    }

    ev.events = EPOLLIN; //监控可读事件
    it->call_back = connect_back;
    it->sockfd = socket_fd;
    ev.data.ptr = it;
  //  ev.data.u32 = 2000;
    epoll_ctl(epfd, EPOLL_CTL_ADD, socket_fd, &ev);

    while(1){
        int nready = epoll_wait(epfd, event, 1024, -1); //fd和过期时间, -1 相当于有数据就返回,
        if(nready < -1){                                // 0那么就直接返回，  返回值为 正数字，
            break;                                      // 那么就是所有的事件
        }

        for(int i = 0 ; i < nready; i++){
            if(event[i].events & EPOLLIN){
                sockitem *si = (sockitem *)event[i].data.ptr; //
                si->call_back(si->sockfd, EPOLLIN, si);
            }

            if(event[i].events & EPOLLOUT){
                sockitem *si = (sockitem *)event[i].data.ptr; //
                si->call_back(si->sockfd, EPOLLOUT, si);
            }
        }
    }

    return 0;
}



/**
 * 这份代码比之前哪种方式是不是明了很多，不同的fd走不同的回调函数，这样就会明了很多
 * main 函数中的循环明显清晰了不少
*/

```

定义了一个 sockitem 结构用于放置socket的fd和毁掉函数，只要是可写事件那么就掉用这个结构中的sockitem的回调函数。
上述代码的代码可以看到我们不用关系具体fd是什么类型的，只要是事件发送了我们就调用相关的回调函数。当读完之后我们就又可以进行发送数据，但是在实际的情况中 send 不一定每次都成功的，如果内核中的发送缓冲区满了，调用就会返回-1，这个我们可以通过epool_ctl 对epoll进行控制，将对这个文件描述符改成为对可写事件进行监控。

### 2.2相当于说将对文件描述符进行监控，改为了对事件进行监控

但是上述还存在一个问题，如果收到的时候只收到了半条数据，那么解析数据的时候就可能存在问题。这样以来我们还可以定义一个发送和接收的缓冲区，对epfd也可以进行封装，将event数组和它放在一起。
这样一来每一个文件描述符只要有时间触发了就会被epoll带回来，之后使用相关的回调函数就可以进行实现了，这个模型就被称为了reactor模型。 所有的fd只要事件就进行处理，根据自己的回调函数进行处理。

```C++
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <error.h>
#include <stdlib.h>
#include <unistd.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <errno.h>


#define buff_size 1024
struct sockitem{
    int sockfd;
    int (*call_back)(int fd, int event, void *arg);
    char sendBuff[buff_size];
    char recvBuff[buff_size];
    size_t sLength;
    size_t rLength;
};

struct epoll_reactor{ 
    int epollfd;
    struct epoll_event event[512];
};

struct epoll_reactor * reactor_loop; //main loop

int recv_back(int fd, int event, void *arg);
int send_backcall(int fd, int event, void *arg){
    struct sockitem *item = (struct sockitem*)arg;
    const char *sendContent = "nihao\n";
    int ret = send(item->sockfd, "nihao\n", strlen(sendContent), 0);
    printf("send size : %d \n", ret);

    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.ptr = item;
    item->sockfd = fd;
    item->call_back = recv_back;
    epoll_ctl(reactor_loop->epollfd, EPOLL_CTL_MOD, fd, &ev);
    return 0;
}

int recv_back(int fd, int event, void *arg)
{
    char buff[1024] = {0};
    struct sockitem *item = (struct sockitem*)arg;
    int ret = recv(fd, buff, 1024, 0); //Write directly item->recvBuff  Can also be
    struct epoll_event ev;
    if(ret <= 0){
        if(errno == EAGAIN || errno == EWOULDBLOCK){
            return -1;
        }else{
                // 连接出错了
        }

        ret = epoll_ctl(reactor_loop->epollfd, EPOLL_CTL_DEL, fd, &ev);
        close(fd);
        free(item);
        printf("epoll_ctl_del ret = %d\n", ret);
        return -2;
    }
    else{
        printf("fd = %d , recv messafe = %s\n", fd, buff);
        item->sockfd = fd;
        item->call_back = send_backcall;
        ev.events = EPOLLOUT | EPOLLET; //设置为ET，和监控可写
        ev.data.ptr = item;
        epoll_ctl(reactor_loop->epollfd, EPOLL_CTL_MOD, fd, &ev); 
        return 0;
    }
}

int connect_back(int fd, int event, void *arg){
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(sockaddr_in));
    socklen_t addrlen = sizeof(struct sockaddr_in);
    int clientfd = accept(fd, (struct sockaddr*)&addr, &addrlen);
    if(fd < 0){
        printf("accept err \n");
        return -1;
    }

    struct epoll_event ev;

  //  ev.data.u32 = 1000;
    struct sockitem* it = (struct sockitem*)malloc(sizeof(struct sockitem));
    if(it == NULL){
        return -2;
    }
    memset(it, 0, sizeof(struct sockitem));

    it->call_back = recv_back;
    it->sockfd = clientfd;
    ev.data.ptr = it;
    ev.events = EPOLLIN;
   // ev.data.fd = clientfd;

    epoll_ctl(reactor_loop->epollfd, EPOLL_CTL_ADD, clientfd, &ev);
    return 0;
}

int main(int argc, char *argv[]){
    int socket_fd  = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(struct sockaddr_in));

    addr.sin_family = AF_INET; //使用IPV4
    addr.sin_port = htons(8888); //绑定端口 8888
    addr.sin_addr.s_addr = INADDR_ANY; //绑定本机的所有网卡

    if(bind(socket_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0){
        exit(-1);
    }

    /*全连接队列为 5 或者 全连接和半连接和为5 和版本有关*/
    if(listen(socket_fd, 5) < 0){
        exit(-1);
    }

    reactor_loop = (struct epoll_reactor*)malloc(sizeof(struct epoll_reactor));
    memset(reactor_loop->event, 0x00, sizeof(struct epoll_event) * 512);
    reactor_loop->epollfd = epoll_create(1);
    struct epoll_event ev;
    sockitem * it = (sockitem*)malloc(sizeof(struct sockitem));
    memset(it, 0, sizeof(struct sockitem));
    ev.events = EPOLLIN; //监控可读事件
    it->call_back = connect_back;
    it->sockfd = socket_fd;
    ev.data.ptr = it;
  //  ev.data.u32 = 2000;
    epoll_ctl(reactor_loop->epollfd, EPOLL_CTL_ADD, socket_fd, &ev);

    while(1){
        int nready = epoll_wait(reactor_loop->epollfd, reactor_loop->event, 512, -1); //fd和过期时间, -1 相当于有数据就返回,
        if(nready < -1){                                // 0那么就直接返回，  返回值为 正数字，
            break;                                      // 那么就是所有的事件
        }

        for(int i = 0 ; i < nready; i++){
            if(reactor_loop->event[i].events & EPOLLIN){
                sockitem *si = (sockitem *)reactor_loop->event[i].data.ptr; //
                si->call_back(si->sockfd, EPOLLIN, si);
            }

            if(reactor_loop->event[i].events & EPOLLOUT){
                sockitem *si = (sockitem *)reactor_loop->event[i].data.ptr; //
                si->call_back(si->sockfd, EPOLLOUT, si);
            }
        }
    }

    return 0;
}


```

上述的代码中虽然在sockitem中定义了发送缓冲区和接收缓冲区，这样如果要发送数据那么就通过往sockitem中的发送缓冲区进行写入数据，如果接收数据就可以从节点中的接收缓冲区进行读数据。

但是上述的代码还存在问题，就是listent的fd和连接的fd在同一个epoll中，这样就可能导致处理连接不会很快，因为很多时间都用于处理连接fd了。
可以考虑用两个epoll进行处理，一个epoll用于处理连接的fd。 例如Nginx和Redis都是监听fd和连接
进行分离，具体的后面再更新。



原文链接：https://blog.csdn.net/Advsance/article/details/121896749

原文作者：Advsance


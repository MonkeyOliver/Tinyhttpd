# tinyhttpd部分源码剖析（一）

为了简便叙述，先定义以下几个缩写：

《UNIX网络编程 卷1：套接字联网API》（第3版）：《UNPv1》

《UNIX环境高级编程》（第3版）：《APUE》

再来一句名人名言：

    《UNP》是一本好书——鲁迅

没有未来的鲤鱼王最近在宿舍摸鱼把《UNPv1》的重点章节（1-11、12-14、25、26和30章）看完了决定借着tinyhttpd的源码复习整合一下，之后几篇文章的目的就是这个（毕竟我也知道tinyhttpd源码剖析的文章已经满天飞了）。为什么说是“部分源码”呢，就500行的源码还只分析部分么，其实因为：

1. tinyhttpd里面有关于处理cgi的代码，而cgi这东西比较有历史，现在前后端花里胡哨的用cgi的不多了，上知乎搜了搜看有老哥吐槽腾讯貌似还在用，反正我也进不了腾讯就不管它了。
2. 当然是因为我太菜了不太能分析源码里处理cgi部分关于管道进程间通信什么的东西，貌似《UNPv2》里有讲以后有机会学了再说吧，毕竟我主要目的是《UNPv1》的学习总结。
3. 500行代码刨掉cgi处理的部分就更短更少了，非常适合我这种懒人。

接下来是正文，如果不是看热闹想学点知识的话建议有计算机网络、C语言基础、一点Unix编程基础和对HTTP协议的简单了解，当然了你要也读过《UNPv1》这文章就没看的必要了因为你也能写，如果是大牛的话欢迎斧正。

提前说明之后的源码讲解部分对代码有一些魔改，和原始的1999年代码略有不同，魔改部分和原因将在源码剖析部分解释。

## 关于Web服务器

首先要说明的是tinyhttpd是一个Web服务器，服务器有很多类型对吧，比如说有DNS服务器，还有SMTP服务器，还有简单的回显服务器（echo）什么的，但tinyhttpd是个Web服务器，那Web服务器和其他服务器相比有什么特殊的地方呢？简单来说Web服务器就是专门通过http协议来处理请求的服务器，举个简单例子，回显服务器是你输一行文本服务器回你一行同样的文本，而你通过http协议输一行文本服务器也通过http协议回你一行文本，那就变成Web服务器了。当然事实上现实中的Web服务器要复杂得多，牛逼的Web服务器比如Apache，Nginx也有很多精巧的设计值得学习，但对于本文的主角tinyhttpd来说并不需要了解那么多。

## 关于HTTP

那么现在你知道了Web服务器要处理HTTP协议，听起来很抽象，处理一种协议，其实实现上很简单，就是用某种格式把你要传输的数据包装一下，这种格式和数据统称为HTTP报文，**关于HTTP报文的内容不在此赘述。**

## tinyhttpd的工作流程

基于以上概念tinyhttpd的工作流程就很显然了：

客户端向服务器发送HTTP请求->服务器建立TCP连接并接受请求->服务器根据请求返回内容

当然了tinyhttpd还有cgi处理部分，之后不再提。

详细过程将在源码讲解中解释，懒得画图，想看图的建议搜索，大佬的博客图画的又多又好。

## 源码上半部分

编译环境：WSL中的Ubuntu18.04

终于到源码部分了，先来一个概览：

    #include <arpa/inet.h>
    #include <ctype.h>
    #include <netinet/in.h>
    #include <pthread.h>
    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <strings.h>
    #include <sys/socket.h>
    #include <sys/stat.h>
    #include <sys/types.h>
    #include <sys/wait.h>
    #include <unistd.h>

    #define ISspace(x) isspace((int)(x))

    #define SERVER_STRING "Server: jdbhttpd/0.1.0\r\n"

    #define SA struct sockaddr  //模仿《UNPv1》

    typedef unsigned short u_short;

    void *accept_request(void *);
    void cat(int, FILE *);
    void error_die(const char *);
    int get_line(int, char *, int);
    void headers(int, const char *);
    void not_found(int);
    void serve_file(int, const char *);
    int startup(u_short *);
    void unimplemented(int);

    //和cgi相关的函数
    void execute_cgi(int, const char *, const char *, const char *);
    void bad_request(int);
    void cannot_execute(int);

    int main(int argc, char **argv) {
        int server_sock = -1;
        int client_sock = -1;
        u_short port = 0;
        if (argc == 1) port = atoi(argv[1]);
        struct sockaddr_in client_name;
        socklen_t client_name_len = sizeof(client_name);
        pthread_t newthread;

        server_sock = startup(&port);  //在port上创建一个监听套接字
        printf("httpd running on port %d\n", port);

        while (1) {
            client_sock = accept(server_sock, (SA *)&client_name, &client_name_len);
            if (client_sock == -1)
                error_die("accept");
            else
                printf("connection established\n");
            // accept_request(client_sock);
            if (pthread_create(&newthread, NULL, &accept_request,
                            (void *)client_sock) != 0)
                perror("pthread_create");
        }

        close(server_sock);
        return (0);
    }

其中

    void execute_cgi(int, const char *, const char *, const char *);
    void bad_request(int);
    void cannot_execute(int);

是和cgi相关的函数，不讲

error_die是简单的异常处理函数，定义如下：

    void error_die(const char *sc) {
        perror(sc);
        exit(1);
    }

其中perror是根据errno返回sc字符串的系统自带函数，详细建议阅读《APUE》。

### 宏定义部分

判断x是不是空格，parse的时候用

    #define ISspace(x) isspace((int)(x))

报文首部的某一行

    #define SERVER_STRING "Server: jdbhttpd/0.1.0\r\n"

模仿《UNPv1》的一个宏定义，之后某些函数传参时写起来方便

    #define SA struct sockaddr  //模仿《UNPv1》

Ubuntu18.04没有u_short类型，懒得改了就typedef一下

    typedef unsigned short u_short;

### main函数

本文只讲main的前半部分，后半部分下篇文章讲。

    int main(int argc, char **argv) {
        int server_sock = -1;//服务端套接字描述符
        int client_sock = -1;//客户端套接字描述符
        u_short port = 0;//端口，初始化为0，之后根据实现是默认随机
        if (argc == 1) port = atoi(argv[1]);//可以指定端口，此处为源码魔改，注意避开众所周知端口
        struct sockaddr_in client_name;//见下文
        socklen_t client_name_len = sizeof(client_name);//client_name结构的大小
        pthread_t newthread;//线程id

        server_sock = startup(&port);  //在port上创建一个TCP监听套接字
        printf("httpd running on port %d\n", port);

什么是众所周知端口？自行搜索。

什么是sockaddr_in结构？

sockaddr_in是套接字地址结构的一种，套接字地址结构均以sockaddr_开头，in指的是ipv4套接字地址结构，而这个结构包含（部分，完整见《UNPv1》3.2节）：

    struct sockaddr_in {
        sa_family_t sin_family;//套接字协议族（in所对应的应该是ipv4所对应的常量AF_INET）
        in_port_t sin_port;//TCP或UDP端口号
        struct in_addr sin_addr;//ipv4地址
    };

    struct in_addr {
        in_addr_t s_addr;//为什么搞得跟俄罗斯套娃似的是历史原因不在此赘述
    };

### startup函数

可以发现前半部分重要的就只有个startup函数，主要目的是在指定端口建立一个TCP监听套接字，具体定义如下：

    int startup(u_short *port) {
        int httpd = 0;
        struct sockaddr_in name;

        if ((httpd = socket(PF_INET, SOCK_STREAM, 0)) == -1) error_die("socket");

        memset(&name, 0, sizeof(name));
        name.sin_family = AF_INET;
        name.sin_port = htons(*port);
        name.sin_addr.s_addr = htonl(INADDR_ANY);

        if (bind(httpd, (SA *)&name, sizeof(name)) < 0) error_die("bind");
        /* if dynamically allocating a port */
        if (*port == 0) {
            socklen_t namelen = sizeof(name);
            if (getsockname(httpd, (SA *)&name, &namelen) == -1)
                error_die("getsockname");
            *port = ntohs(name.sin_port);
        }

        if (listen(httpd, 5) < 0) error_die("listen");
        return (httpd);
    }

《UNPv1》里有个写得**不知道比这个高到哪里去**的类似函数Tcp_listen()，详细可见《UNPv1》11.13节。

事实上建立一个监听套接字过程很简单，无论你是TCP还是UDP基本都这么几步（UDP一般没监听一说，绑定完就完事儿了）：

1. 用socket()函数初始化一个套接字描述符
2. 初始化一个套接字地址结构
3. 用bind()函数将套接字描述符和套接字地址结构绑定
4. 将绑定好的套接字变成监听状态

接下来详细说明：

初始化套接字描述符：

        if ((httpd = socket(PF_INET, SOCK_STREAM, 0)) == -1) error_die("socket");

用汉语解释就是：建立一个ipv4（PF_INET）的TCP（SOCK_STREAM）套接字描述符并赋给httpd变量，如果返回-1说明出错。其中PF_INET和AF_INET差不多，第三个参数一般为0，关于socket()函数详细可见《UNPv1》4.3节。

初始化套接字地址结构：

     memset(&name, 0, sizeof(name));
     name.sin_family = AF_INET;
     name.sin_port = htons(*port);
     name.sin_addr.s_addr = htonl(INADDR_ANY);

其中htons和htonl函数为字节排序函数，h为host，n为network，s为short，l为long（to？to就是to），用途为将主机字节序转化为网络字节序并赋给套接字地址结构内对应的变量，INADDR_ANY为通配地址，具体以及原因详细可见《UNPv1》3.4节。

绑定：

     if (bind(httpd, (SA *)&name, sizeof(name)) < 0) error_die("bind");
     /* if dynamically allocating a port */
     /* 如果指定port为0，则返回一个内核赋予该连接的本地端口号*/
     if (*port == 0) {
         socklen_t namelen = sizeof(name);
         if (getsockname(httpd, (SA *)&name, &namelen) == -1)
             error_die("getsockname");
         *port = ntohs(name.sin_port);
     }

getsockname()函数详细可见《UNPv1》4.10节，ntohs类似上文的htons和htonl函数，只不过相反，根据网络字节序的值返回主机字节序的值，同样详细可见《UNPv1》3.4节。

监听并返回：

     if (listen(httpd, 5) < 0) error_die("listen");
     return (httpd);

为什么是5？历史原因，《UNPv1》里默认是1024，表示最大可同时监听描述符数量，以下摘录自《UNPv1》源码unp.h里的一个宏定义：

    /* Following could be derived from SOMAXCONN in <sys/socket.h>, but many
    kernels still #define it as 5, while actually supporting many more */
    #define LISTENQ 1024 /* 2nd argument to listen() */

## 结语

前半部分就这么完了，很简单是不是？以上内容其实大多数服务器都会涉及到（就是初始化+监听这一套操作），之后有空了再写后半部分，不过不得不再次吐槽，500行的代码也要分几部分写，我太菜了.jpg。

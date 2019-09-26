# tinyhttpd部分源码剖析（二）

废话就不多说了，关于一些宏定义和缩写的内容看上一篇文章链接，直接进入正文

## 源码下半部分

    while (1) {
        if ((client_sock = accept(server_sock, (SA *)&client_name, &client_name_len)) == -1)
            error_die("accept");
        else
            printf("connection established\n");
        // accept_request(client_sock);
        if (pthread_create(&newthread, NULL, &accept_request, (void *)client_sock) != 0)
            perror("pthread_create");
    }

    close(server_sock);

    return (0);

### accept函数

    if ((client_sock = accept(server_sock, (SA *)&client_name, &client_name_len)) == -1)
        error_die("accept");
    else
        printf("connection established\n");

accept()函数用于从已完成连接队列队头返回下一个已完成连接，具体返回的是一个内核提供的已连接套接字描述符。什么是已完成连接呢，简单来说就是已经完成三次握手的连接，一般是客户端那边调用connect()主动向服务端发起连接并建立连接，然后服务端这边用accept()函数接受已建立连接并使用，通常称accept()的第一参数（server_sock变量）为监听套接字描述符，返回值（client_sock）为已连接套接字描述符，总之，这行代码可以简单理解为建立一个已连接套接字描述符。详细可见《UNPv1》4.6节。

### 创建新线程

    if (pthread_create(&newthread, NULL, &accept_request, (void *)client_sock) != 0)
        perror("pthread_create");

这句代码用汉语翻译一下就是创建一个线程id为newthread的新线程并执行accept_request()函数，同时向这个函数传入client_sock参数。源代码写得不怎么规范用了一些隐式转化的东西，pthread_create()传入的函数其实应该是一个返回(void *)的函数，向函数传入的参数也应该是(void *)的，所以更加规范的应该进行手动强制转化。

事实上关于线程部分如果要展开写那就是大书特书了根本写不完所以在这里就不详细讨论了，简单来说tinyhttpd用的是“每个客户一个线程”的模型，即每来一个客户创一个新线程处理，当然这样可能会遇到一些问题，比如海量客户访问后的性能问题啊加不加锁分离不分离具体就不在这说了，详细可见《UNPv1》26和30章以及各种多线程服务器开发书籍。

### accept_request函数

如果说上一篇最重要的函数是startup()，那很明显这一篇最重要的函数就是这个accept_request()了，先简单说一下他干了什么：接受并处理一个HTTP请求报文再根据报文发送指定内容和响应报文。

    void *accept_request(void *arg) {
        int client = (int)arg;
        int numchars;
        char buf[1024];
        char method[255];
        char url[255];
        char path[512];
        size_t i, j;
        struct stat st;
        int cgi = 0; /* becomes true if server decides this is a CGI program */
        char *query_string = NULL;

        numchars = get_line(client, buf, sizeof(buf));
        i = 0, j = 0;

        while (!ISspace(buf[j]) && (i < sizeof(method) - 1)) method[i++] = buf[j++];
        method[i] = '\0';

        int method_is_get = !strcasecmp(method, "GET");
        int method_is_post = !strcasecmp(method, "POST");

        if (!method_is_get && !method_is_post) {
            unimplemented(client);
            return NULL;
        }

        if (method_is_post) cgi = 1;

        i = 0;
        while (ISspace(buf[j]) && (j < sizeof(buf))) j++;
        while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < sizeof(buf)))
            url[i++] = buf[j++];
        url[i] = '\0';

        if (method_is_get) {
            query_string = url;
            while ((*query_string != '?') && (*query_string != '\0'))
                query_string++;
            if (*query_string == '?') {
                cgi = 1;
                *query_string = '\0';
                query_string++;
            }
        }

        sprintf(path, "htdocs%s", url);
        if (path[strlen(path) - 1] == '/') strcat(path, "index.html");
        printf("parse succeed and path is %s\n", path);
        /*一旦给出pathname，stat函数将返回与此命名文件有关的信息结构struct
        * stat——《APUE》4.2节
        */
        if (stat(path, &st) == -1) {
            while ((numchars > 0) && strcmp("\n", buf)) /* read & discard headers */
                numchars = get_line(client, buf, sizeof(buf));
            not_found(client);
        } else {
            /* The original source code here is:
            *    if ((st.st_mode & S_IFMT) == S_IFDIR) strcat(path, "/index.html");
            */
            if (S_ISDIR(st.st_mode)) strcat(path, "/index.html");
            if ((st.st_mode & S_IXUSR) || (st.st_mode & S_IXGRP) ||
                (st.st_mode & S_IXOTH))
                cgi = 1;
            // if (!cgi)
            //     serve_file(client, path);
            // else
            //     execute_cgi(client, path, method, query_string);
            serve_file(client, path);
        }

        close(client);
    }

### 读取请求报文起始行

    numchars = get_line(client, buf, sizeof(buf));

这里的client其实就是之前说的传进来的已连接套接字，这句话说白了就是将套接字里的东西读到buf里并返回读到的大小，get_line()函数定义如下：

    int get_line(int sock, char *buf, int size) {
        int i = 0;
        char c = '\0';
        int n;
        while ((i < size - 1) && (c != '\n')) {
            n = recv(sock, &c, 1, 0);
            /* DEBUG printf("%02X\n", c); */
            if (n > 0) {
                if (c == '\r') {
                    n = recv(sock, &c, 1, MSG_PEEK);
                    /* DEBUG printf("%02X\n", c); */
                    if ((n > 0) && (c == '\n'))
                        recv(sock, &c, 1, 0);
                    else
                        c = '\n';
                }
                buf[i++] = c;
            } else
                c = '\n';
        }
        buf[i] = '\0';
        return (i);
    }

recv()函数的作用是从sock里读取x（此处为1）大小字节并返回读到的字符（c）和读到的大小（n），至于最后那个参数，会发现有一行是MSG_PEEK其他都是0，简单来说recv()函数是从sock中读，填入buf，然后把sock里读到的丢掉，加个MSG_PEEK意思是不丢掉，《UNPv1》里翻译的是“窥探”，哎意思就是我看一眼是啥就行了，关于recv()和之后那个参数详细可见《UNPv1》14.4和14.7节，最后这个get_line()函数的意思就是从sock中读取到内容（什么内容？请求报文）的开头开始跑，边跑边填buf，读到'\r'时说明一行结束（为什么？复习HTTP报文结构），如果之后有'\n'就再读一个'\n'，没有就补一个'\n'，然后返回。

### 解析请求报文起始行

    while (!ISspace(buf[j]) && (i < sizeof(method) - 1)) method[i++] = buf[j++];
    method[i] = '\0';

    int method_is_get = !strcasecmp(method, "GET");
    int method_is_post = !strcasecmp(method, "POST");

    if (!method_is_get && !method_is_post) {
        unimplemented(client);
        return NULL;
    }

    if (method_is_post) cgi = 1;

    i = 0;
    while (ISspace(buf[j]) && (j < sizeof(buf))) j++;
    while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < sizeof(buf)))
        url[i++] = buf[j++];
    url[i] = '\0';

    if (method_is_get) {
        query_string = url;
        while ((*query_string != '?') && (*query_string != '\0'))
            query_string++;
        if (*query_string == '?') {
            cgi = 1;
            *query_string = '\0';
            query_string++;
        }
    }

    sprintf(path, "htdocs%s", url);
    if (path[strlen(path) - 1] == '/') strcat(path, "index.html");
    printf("parse succeed and path is %s\n", path);

之后这一大段其实都在解析来自客户的请求报文的起始行并分别填入method、url里，然后根据method做出一些选择并将url变成可以访问服务器上文件的具体路径path，这里method只涉及到了GET和POST，因为strcasecmp()是相同返回0，所以定义变量的时候加了个非!，源代码并没用变量替代导致可读性略差，虽然我觉得这一大块完全可以写一个稳健性更强的parse函数来代替但还是不改动太多了。query_string和cgi有关不讲。

### 读取指定文件

    /*一旦给出pathname，stat函数将返回与此命名文件有关的信息结构struct
    * stat——《APUE》4.2节
    */
    if (stat(path, &st) == -1) {
        while ((numchars > 0) && strcmp("\n", buf)) /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));
        not_found(client);
    } else {
        /* The original source code here is:
        *    if ((st.st_mode & S_IFMT) == S_IFDIR) strcat(path, "/index.html");
        */
        if (S_ISDIR(st.st_mode)) strcat(path, "/index.html");
        if ((st.st_mode & S_IXUSR) || (st.st_mode & S_IXGRP) ||
            (st.st_mode & S_IXOTH))
            cgi = 1;
        // if (!cgi)
        //     serve_file(client, path);
        // else
        //     execute_cgi(client, path, method, query_string);
        serve_file(client, path);
    }

    close(client);

关于stat函数和stat结构详细可见《APUE》第4章，也是一个需要大书特书的内容就不展开讲了，简单来说如注释解释，如果没找到与此命名文件（这里的文件也可能是个目录文件）有关的信息结构（说白了就是没找到这文件），把报文剩余部分读掉并返回一个404，找到了如果是个目录文件就补一个index.html将他变成个html文件，根据《APUE》4.3节，注释掉的那部分源代码是比较早期的写法了，现在可以直接用S_ISDIR(mode)宏定义来替代，该宏定义的实现就是那行代码但常数名称略有不同，至于底下那个关于cgi的一系列权限常数详细见《APUE》第4章不在此赘述。最后用serve_file()函数将path所对应的文件送给client已连接套接字描述符。

### 返回404

    void not_found(int client) {
        char buf[1024];

        sprintf(buf, "HTTP/1.0 404 NOT FOUND\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, SERVER_STRING);
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "Content-Type: text/html\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "<HTML><TITLE>Not Found</TITLE>\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "<BODY><P>The server could not fulfill\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "your request because the resource specified\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "is unavailable or nonexistent.\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "</BODY></HTML>\r\n");
        send(client, buf, strlen(buf), 0);
    }

不解释了，就是写了个404格式的响应报文。

### 返回服务器上的一个文件

    void serve_file(int client, const char *filename) {
        FILE *resource = NULL;
        int numchars = 1;
        char buf[1024];

        buf[0] = 'A'; buf[1] = '\0';
        while ((numchars > 0) && strcmp("\n", buf)) /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));

        resource = fopen(filename, "r");
        if (resource == NULL)
            not_found(client);
        else {
            headers(client, filename);
            cat(client, resource);
        }
        fclose(resource);
    }

这个函数其实我挺迷的，感觉进行了一些不怎么优雅的冗余操作，但又不知道该怎么改，事实上如果能进入到这个函数里那么filename必然是存在的（stat函数已经做出了判断），那么resource没有读取到可能是因为权限问题而不是单纯的not found，前面将报文首部读掉的操作感觉也略显笨拙，当然了如果有大佬能解释一下的话欢迎赐教。总之这个函数就是读取了filename所指定的文件，然后用headers函数向套接字写了响应报文的起始行和首部，cat向套接字写了实体部分。

### 填写起始行和首部

    void headers(int client, const char *filename) {
        char buf[1024];
        (void)filename; /* could use filename to determine file type */

        strcpy(buf, "HTTP/1.0 200 OK\r\n");
        send(client, buf, strlen(buf), 0);
        strcpy(buf, SERVER_STRING);
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "Content-Type: text/html\r\n");
        send(client, buf, strlen(buf), 0);
        strcpy(buf, "\r\n");
        send(client, buf, strlen(buf), 0);
    }

将字符串填入buf，再将buf写入套接字。

### 填写实体

    void cat(int client, FILE *resource) {
        char buf[1024];

        fgets(buf, sizeof(buf), resource);
        while (!feof(resource)) {
            send(client, buf, strlen(buf), 0);
            fgets(buf, sizeof(buf), resource);
        }
    }

也很好理解，先在外面写一行的原因是避免resource为空（对于tinyhttpd给的例子来说就是一个啥都没有的index.html）。

### 关闭已连接套接字

    close(client);

是不是已经不知道这一行是哪个函数里的了？accept_request()函数最后一行，因为HTTP是无连接的所以一切都结束后要关闭已连接套接字。

## 真正的结语

事实上很久之前就在知乎上听说了这个源码非常适合新手入门，但其实学的东西多了以后发现这个源码写的也就那样，并不像一些人吹的那样神乎其神，所以想起来知乎上一个老哥说的“上来就推一堆源码的我寻思着你们都读过么？”，说的很有道理，不过瑕不掩瑜，tinyhttpd用来复习一些基本概念还是很好用的，但要真正去学习现代的Web服务器可能还得看看Nginx什么的，以后有机会再说吧。

噢对了最后这个代码编译完是可以跑的，编译完运行后打开浏览器输入127.0.0.0:端口号就可以看到index.html了

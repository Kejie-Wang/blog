---
title: Linux下的基于Pthread的多线程Socket编程
tags: 'linux, pthread, socket'
description: >-
  本文主要基于Linux下的Pthread实现了一个服务器和客户端通过socket进行通信的系统。服务端的程序的设计主要是一个主线程首先调用Socket相关的函数socket，bind，
  listen在建立服务端的Socket之后，等待Accept上面，如果有新的客户端连接上来，则对于每一个客户端新建一个线程。在每一个客户端的线程中，其接收客户端发送的指令然后返回相关的信息，主线程Socket中默认的Accpet默认是阻塞的，这里采用了Linux中函数fctnl将其设置成非阻塞。除了客户端对应的线程和主线程之外，服务端还存在一个服务端的控制线程，其主要是接受服务端的命令，诸如退出，断开连接等等，对服务端的程序进行控制。客户端的程序为一个阻塞的循环，接受用户的输入然后解析命令进行相关的操作，诸如连接服务端，发送数据等等操作。
date: 2016-05-27 03:19:00
---


# **程序设计**

服务端的程序的设计主要是一个主线程首先调用Socket相关的函数socket，bind， listen在建立服务端的Socket之后，等待Accept上面，如果有新的客户端连接上来，则对于每一个客户端新建一个线程。在每一个客户端的线程中，其接收客户端发送的指令然后返回相关的信息，主线程Socket中默认的Accpet默认是阻塞的，这里采用了Linux中函数fctnl将其设置成非阻塞。除了客户端对应的线程和主线程之外，服务端还存在一个服务端的控制线程，其主要是接受服务端的命令，诸如退出，断开连接等等，对服务端的程序进行控制。

客户端的程序为一个阻塞的循环，接受用户的输入然后解析命令进行相关的操作，诸如连接服务端，发送数据等等操作

# 服务端实现细节

服务端的程序主要包括主线程接受客户端请求响应建立客户端对应线程，客户端命令执行，服务端主要控制模块。

## **主线程**

首先建立服务端Socket连接，其过程如下：

- 首先调用socket函数建立一个socket，其中传入的参数AF_INET参数表示其采用TCP协议，并且调用Linux系统提供的fcntl函数设置这个socket为非阻塞。
- 然后初始化服务的地址，保护监听的IP（这里设置的监听服务器端所有的网卡），设置相应的端口以及协议类型（TCP），调用bind函数将其和前面的建立的socket绑定。
- 设置该socket开始监听，这主要对应TCP连接上面的第一次握手，将TCP第一次握手成功的放在队列（SYN队列）中，其中传入的参数BACKLOG为Socket中队列的长度。
- 然后调用Accept函数进行接收客户端的请求，其主要是从TCP上次握手成功的Accept队列中将客户端信息去除进行处理(此部分代码后面解释)。

最后服务端的主要是在一个循环中，其中最前面的serverExit主要是为了交互需要，后面会详细介绍。服务端调用Accept函数进行接收一个客户端的信息，如果存在连接上的客户端，则为其分配一个线程位置，然后调用pthread_create创建一个线程，并且将该客户端的信息参数传入线程。这里采用一个结构体的方式记录了客户端的相关信息，包括客户端IP，端口号，编号，是否连接等等。

这里为了防止服务器的无限的创建线程，设置了最大同时在线的客户端数目（MAXCONN），如果新连接上的客户端发现已经操作上线，那么该主线程就会睡眠，其等待在了一个客户端断开连接的信号量上面，如果有客户端断开连接，则主线程被唤醒，接受该客户端的连接。

需要注意的是这里涉及到很多的多线程的操作，很有可能发生数据冲突，需要很好使用pthread中的互斥锁防止发生数据冲突。

## **客户端服务线程**

其主要是阻塞的接受客户端发送上来的命令，然后解析命令，对应每个命令进行不同的操作，将结果通过send函数发送给对应的客户端。这里主要实现了disconn断开连接，name返回客户端，list返回所有连接的客户端的信息（IP，端口，编号等等），send向别的客户端发送数据。
对于name和time命令只要将数据以字符串的形式发送给对应的客户端即可，而对于disconn命令，需要发送一个断开的信号，使得等待在这个信号量上的客户端能够连接上服务端，对于send命令需要根据发送的地址获取到目的的客户端信息，相对应的客户端发送相关的消息。服务端的程序另一个需要的是需要进行错误处理，诸如客户端的命令错误，格式错误，网络错误等等异常情况，其需要进行相应的报错，并且提供给客户端或者服务端相应的错误信息。
这里主要注意的是发送的数据需要指定数据的长度，而采用C的strlen指定的并不包括字符串最后的‘\0’，因此其不会发送出去。这里有两种解决方案，一种是接收方在数据的最后加上’\0’或者发送的时候将’\0’一同发送出去。 

## **服务端控制线程**

服务端控制程序主要提供了一些命令给服务端可以对于服务端连接的客户端以及自身退出。这里主要实现了exit退出服务端，list列出所有连接的客户端信息，kill关闭某个客户端命令。其中exit命令为退出客户端，其主要是将一个退出退出标志位设置成1，然后主线程判断该标志位为1的时候会关闭所有的客户端，并且取消所有的客户端线程以及服务端程序线程，然后自身退出。对于list命令其和服务端发送的list指令实现相同，kill命令主要是关闭对于对应的客户端的socket，然后取消其对应的线程。

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/time.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <pthread.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

#define PORT  8888
#define BACKLOG 10
#define MAXCONN 100
#define BUFFSIZE 1024

typedef unsigned char BYTE;
typedef struct ClientInfo
{
    struct sockaddr_in addr;
    int clientfd;
    int isConn;
    int index;
} ClientInfo;

pthread_mutex_t activeConnMutex;
pthread_mutex_t clientsMutex[MAXCONN];
pthread_cond_t connDis;

pthread_t threadID[MAXCONN];
pthread_t serverManagerID;

ClientInfo clients[MAXCONN];

int serverExit = 0;

/*@brief Transform the all upper case 
*
*/
void tolowerString(char *s)
{
    int i=0;
    while(i < strlen(s))
    {
        s[i] = tolower(s[i]);
        ++i;
    } 
}

void listAll(char *all)
{
    int i=0, len = 0;
    len += sprintf(all+len, "Index   \t\tIP Address   \t\tPort\n");
    for(;i<MAXCONN;++i)
    {
        pthread_mutex_lock(&clientsMutex[i]);
        if(clients[i].isConn)
            len += sprintf(all+len, "%.8d\t\t%s\t\t%d\n",clients[i].index, inet_ntoa(clients[i].addr.sin_addr), clients[i].addr.sin_port);
        pthread_mutex_unlock(&clientsMutex[i]);
    }
}

void clientManager(void* argv)
{
    ClientInfo *client = (ClientInfo *)(argv);
    
    BYTE buff[BUFFSIZE];
    int recvbytes;
    
    int i=0;
    int clientfd = client->clientfd;
    struct sockaddr_in addr = client->addr;
    int isConn = client->isConn;
    int clientIndex = client->index;
    
    while((recvbytes = recv(clientfd, buff, BUFFSIZE, 0)) != -1)
    {
     //   buff[recvbytes] = '\0';
        tolowerString(buff);    //case-insensitive
        
        char cmd[100];
        if((sscanf(buff, "%s", cmd)) == -1)    //command error
        { 
           char err[100];         
           if(send(clientfd, err, strlen(err)+1, 0) == -1)
           {
               strcpy(err, "Error command and please enter again!\n");
               fprintf(stdout, "%d sends an eroor command\n", clientfd);
               break;
           }
        }
        else    
        {
            char msg[BUFFSIZE]; //The message content
            int dest = clientIndex; //message destination
            int isMsg = 0;              //any message needed to send
            if(strcmp(cmd, "disconn") == 0)
            {   
                pthread_cond_signal(&connDis);  //send a disconnetion signal and the waiting client can get response  
                break;
            }
            else if(strcmp(cmd, "time") == 0)
            {
                time_t now;
                struct tm *timenow;
                time(&now);
                timenow = localtime(&now);
                strcpy(msg, asctime(timenow));
                isMsg = 1;
            }
            else if(strcmp(cmd, "name") == 0)
            {
                strcpy(msg, "MACHINE NAME");
                isMsg = 1;
            }
            else if(strcmp(cmd, "list") == 0)
            {                            
                listAll(msg);
                isMsg = 1;
            }
            else if(strcmp(cmd, "send") == 0)
            {   
                    
                if(sscanf(buff+strlen(cmd)+1, "%d%s", &dest, msg)==-1 || dest >= MAXCONN)
                {
                    char err[100];
                    strcpy(err, "Destination ID error and please use list to check and enter again!\n");
                    fprintf(stderr, "Close %d client eroor: %s(errno: %d)\n", clientfd, strerror(errno), errno);
                    break;
                }
                fprintf(stdout, "%d %s\n", dest, msg);   
                isMsg = 1;
            }
            else
            {
                char err[100];
                strcpy(err, "Unknown command and please enter again!\n");
                fprintf(stderr, "Send to %d message eroor: %s(errno: %d)\n", clientfd, strerror(errno), errno);
                break;
            }
            
            
            if(isMsg)
            {
                pthread_mutex_lock(&clientsMutex[dest]);
                if(clients[dest].isConn == 0)
                {
                    sprintf(msg, "The destination is disconneted!");
                    dest = clientIndex; 
                } 
                            
                if(send(clients[dest].clientfd, msg, strlen(msg)+1, 0) == -1)
                {
                    fprintf(stderr, "Send to %d message eroor: %s(errno: %d)\n", clientfd, strerror(errno), errno);
                    pthread_mutex_unlock(&clientsMutex[dest]); 
                    break;
                }
                printf("send successfully!\n");
                pthread_mutex_unlock(&clientsMutex[dest]); 
            }
        }  //end else       
    }   //end while
    
    pthread_mutex_lock(&clientsMutex[clientIndex]);
    client->isConn = 0;
    pthread_mutex_unlock(&clientsMutex[clientIndex]);
    
    if(close(clientfd) == -1)
        fprintf(stderr, "Close %d client eroor: %s(errno: %d)\n", clientfd, strerror(errno), errno);
    fprintf(stderr, "Client %d connetion is closed\n", clientfd);
    
    pthread_exit(NULL);
}

void serverManager(void* argv)
{
    while(1)
    {
        char cmd[100];
        scanf("%s", cmd);
        tolowerString(cmd);
        if(strcmp(cmd, "exit") == 0)
            serverExit = 1;
        else if(strcmp(cmd, "list") == 0)
        {
            char buff[BUFFSIZE];
            listAll(buff);
            fprintf(stdout, "%s", buff);
        }
        else if(strcmp(cmd, "kill") == 0)
        {
            int clientIndex;
            scanf("%d", &clientIndex);
            if(clientIndex >= MAXCONN)
            {
                fprintf(stderr, "Unkown client!\n");
                continue;
            }
            pthread_mutex_lock(&clientsMutex[clientIndex]);
            if(clients[clientIndex].isConn)
            {
                if(close(clients[clientIndex].clientfd) == -1)
                    fprintf(stderr, "Close %d client eroor: %s(errno: %d)\n", clients[clientIndex].clientfd, strerror(errno), errno);
            }
            else
            {
                fprintf(stderr, "Unknown client!\n");
            }
            pthread_mutex_unlock(&clientsMutex[clientIndex]);
            pthread_cancel(threadID[clientIndex]);
                
        }
        else
        {
            fprintf(stderr, "Unknown command!\n");
        }
    }
}

int main()
{
   int activeConn = 0;
   
   //initialize the mutex 
   pthread_mutex_init(&activeConnMutex, NULL);   
   pthread_cond_init(&connDis, NULL);
   int i=0;
   for(;i<MAXCONN;++i)
       pthread_mutex_init(&clientsMutex[i], NULL); 
   
   for(i=0;i<MAXCONN;++i)
       clients[i].isConn = 0; 
       
   //create the server manager thread
   pthread_create(&serverManagerID, NULL, (void *)(serverManager), NULL);
   
   
   int listenfd;
   struct sockaddr_in  servaddr;
    
   //create a socket
   if((listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
   {
       fprintf(stderr, "Create socket error: %s(errno: %d)\n", strerror(errno), errno);
       exit(0);
   }
   else
       fprintf(stdout, "Create a socket successfully\n");
   
   fcntl(listenfd, F_SETFL, O_NONBLOCK);       //set the socket non-block
   
   //set the server address
   memset(&servaddr, 0, sizeof(servaddr));  //initialize the server address 
   servaddr.sin_family = AF_INET;           //AF_INET means using TCP protocol
   servaddr.sin_addr.s_addr = htonl(INADDR_ANY);    //any in address(there may more than one network card in the server)
   servaddr.sin_port = htons(PORT);            //set the port
   
   //bind the server address with the socket
   if(bind(listenfd, (struct sockaddr*)(&servaddr), sizeof(servaddr)) == -1)
   {
        fprintf(stderr, "Bind socket error: %s(errno: %d)\n", strerror(errno), errno);
        exit(0);
   }
   else
       fprintf(stdout, "Bind socket successfully\n");
   
   //listen
   if(listen(listenfd, BACKLOG) == -1)
   {
       fprintf(stderr, "Listen socket error: %s(errno: %d)\n", strerror(errno), errno);
       exit(0);
   }
   else
       fprintf(stdout, "Listen socket successfully\n");
    
   
   while(1)
   {
       if(serverExit)
       {
           for(i=0;i<MAXCONN;++i)
           {
               if(clients[i].isConn)
               {
                   if(close(clients[i].clientfd) == -1)         //close the client 
                       fprintf(stderr, "Close %d client eroor: %s(errno: %d)\n", clients[i].clientfd, strerror(errno), errno);
                   if(pthread_cancel(threadID[i]) != 0)         //cancel the corresponding client thread
                        fprintf(stderr, "Cancel %d thread eroor: %s(errno: %d)\n", (int)(threadID[i]), strerror(errno), errno);
               }
           }
           return 0;    //main exit;
       }
       
       pthread_mutex_lock(&activeConnMutex);
       if(activeConn >= MAXCONN)
            pthread_cond_wait(&connDis, &activeConnMutex);
       pthread_mutex_unlock(&activeConnMutex);
           
       //find an empty postion for a new connnetion
       int i=0;
       while(i<MAXCONN)
       {
           pthread_mutex_lock(&clientsMutex[i]);
           if(!clients[i].isConn)
           {
               pthread_mutex_unlock(&clientsMutex[i]);
               break;
           }
           pthread_mutex_unlock(&clientsMutex[i]);
           ++i;           
       }   
       
       //accept
       struct sockaddr_in addr;
       int clientfd;
       int sin_size = sizeof(struct sockaddr_in);
       if((clientfd = accept(listenfd, (struct sockaddr*)(&addr), &sin_size)) == -1)
       {   
           sleep(1);        
           //fprintf(stderr, "Accept socket error: %s(errno: %d)\n", strerror(errno), errno);
           continue;
           //exit(0);
       }
       else
           fprintf(stdout, "Accept socket successfully\n");
       
       pthread_mutex_lock(&clientsMutex[i]);
       clients[i].clientfd = clientfd;
       clients[i].addr = addr;
       clients[i].isConn = 1;
       clients[i].index = i;
       pthread_mutex_unlock(&clientsMutex[i]);
       
       //create a thread for a client
       pthread_create(&threadID[i], NULL, (void *)clientManager, &clients[i]);     
       
   }    //end-while
}
```

# **客户端实现细节**

客户端程序主要可以分成客户端主线程控制和服务端消息接受程序。

## **主线程**

客户端的程序较服务端的程序则简单的多，主要是接受用户的指令按照指令执行相关的任务。这里主要实现了conn连接服务器，disconn断开服务器连接，list查询服务器上的所有连接，name查看服务器名字，time查看时间，send先其他的客户端发送消息这些命令。

客户端首先必须通过conn命令连接上对应的服务器，然后可以执行其他的指令。用户通过time，name查看时间和机器名，如果需要发送消息，可以通过list查看所有连接，然后向指定的连接发送消息，quit退出客户端。

## **服务器消息处理线程**

由于主线程为一个阻塞的线程，因此这里创建了一个接受服务器端的消息的线程。当用户通过conn命令连接上服务器之后，主线程就会创建该接受线程，其不断的监听服务器发回的消息，然后将其输入的屏幕上。当用户通过disconn命令取消连接的时候，主线程则就会取消掉这个线程。下面是该线程相关的代码。

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <stdio.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <errno.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>

#define BUFFERSIZE 1024
typedef unsigned char BYTE;
pthread_t receiveID;

void tolowerString(char *s)
{
    int i=0;
    while(i < strlen(s))
    {
        s[i] = tolower(s[i]);
        ++i;
    } 
}

void receive(void *argv)
{
    int sockclient = *(int*)(argv);
    BYTE recvbuff[BUFFERSIZE];
    while(recv(sockclient, recvbuff, sizeof(recvbuff), 0)!=-1) //receive
    {
        fputs(recvbuff, stdout);
        fputs("\n", stdout);      
    }
    fprintf(stderr, "Receive eroor: %s(errno: %d)\n", strerror(errno), errno);
}

int main()
{
    ///define sockfd
    int sockclient = socket(AF_INET,SOCK_STREAM, 0);

    ///definet sockaddr_in
    struct sockaddr_in servaddr;
    memset(&servaddr, 0, sizeof(servaddr));
    
    int isConn = 0;
    
    BYTE buff[BUFFERSIZE];

    
    while (fgets(buff, sizeof(buff), stdin) != NULL)
    {
        tolowerString(buff);
        char cmd[100], ip[100];
        int port;
        if(sscanf(buff, "%s", cmd) == -1)    //command error
        { 
            fprintf(stderr, "Input eroor: %s(errno: %d) And please input again\n", strerror(errno), errno);
            continue;
        }
        if(strcmp(cmd, "conn") == 0)        //connecton command
        {
            char ip[100];
            int port, ipLen=0;
            if(sscanf(buff+strlen(cmd)+1, "%s", ip) == -1)    //command error
            { 
                fprintf(stderr, "Input eroor: %s(errno: %d) And please input again\n", strerror(errno), errno);
                continue;
            }
            if((sscanf(buff+strlen(cmd)+strlen(ip)+2, "%d", &port)) == -1)    //command error
            { 
                fprintf(stderr, "Input eroor: %s(errno: %d) And please input again\n", strerror(errno), errno);
                continue;
            }  
           // fprintf(stdout, "%s %d\n",ip, port);          
            servaddr.sin_family = AF_INET;
            servaddr.sin_port = htons(port);  ///server port
            servaddr.sin_addr.s_addr = inet_addr(ip);  //server ip
            if (connect(sockclient, (struct sockaddr *)&servaddr, sizeof(servaddr)) < 0)
            {
                fprintf(stderr, "Connect eroor: %s(errno: %d)\n", strerror(errno), errno);
                continue;
            } 
            fprintf(stdout, "Connect successfully\n"); 
            isConn = 1;  
            pthread_create(&receiveID, NULL, (void *)(receive), (void *)(&sockclient));            
        }
        else if(strcmp(cmd, "disconn") == 0)
        {
            if(isConn == 0)
            {
                fprintf(stdout, "There is not a connection!\n");
                continue;
            }
            else
            {
                pthread_cancel(receiveID);
                close(sockclient);
            }   
            isConn = 0;
        }
        else if(strcmp(cmd, "quit") == 0)
        {
            if(isConn)
            {
                pthread_cancel(receiveID);
                close(sockclient);
            }
            return 0;
        }
        else
        {
            if(send(sockclient, buff, strlen(buff)+1, 0) == -1) //send
            {
                fprintf(stderr, "Send eroor: %s(errno: %d)\n", strerror(errno), errno);
                continue;
            }
            
            if(isConn == 0)
            {
                fprintf(stdout, "Please use conn <ip> <port> command to build a connnection!\n");
                continue;
            }
            memset(buff, 0, sizeof(buff));
        }      
    }

    close(sockclient);
    return 0;
}
```

# **程序测试结果**

实验测试采用了3个客户端同时连接一个服务端进行测试。同时运行这四个程序，首先3个客户端通过conn命令连接上客户端，我们发现3个客户端都同时连接上，说明conn命令正常工作，socket建立正确。然后在客户端1中我们对一些命令进行测试，输入name发现返回了机器的名字，输入time返回了时间，输入list，方向其显示了3个客户端的连接信息。这说明了name，time和list都正常工作。然后在客户端中通过send命令先客户端2发送一条消息，发现客户端2收到了消息，然后客户端2回复了一条消息，客户端1也收到了消息。先客户端3发送消息，其也正常的收到，其发送消息给客户端2，客户端2也能正常收到。着说明了程序中send命令运行正常，能够进行消息的收发。最后客户端1,2,3都通过quit命令退出了客户端，并且服务器端收到了该退出的消息，并且关闭了该连接和线程，最后服务器端退出了。

## 服务器

![](/images/linux-pthread-socket/server.png)

## 客户端1

![](/images/linux-pthread-socket/client1.png)

## 客户端2

![](/images/linux-pthread-socket/client2.png)

## 客户端3

![](/images/linux-pthread-socket/client3.png)
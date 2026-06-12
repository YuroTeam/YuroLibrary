# 基于C/S套接字编程实现简单聊天程序

# 前言

这是我在大二下学期学完了计算机网络后需要进行的课程设计，这个题目相对而言还算比较简单，于此存档记录。

# 关键词

#### ①C/S模式： Client / Server模式，即客户端\-服务器通信模式

#### ②套接字：即Windows Sockets,是Windows下网络编程的规范，为基于Windows开发平台的、得到广泛应用、开放、支持多种协议的网络编程接口

#### ③套接口：套接字编程接口，即Socket编程接口。套接口是对网络中不同主机上应用进程之间进行双向通信的端点的抽象，从效果上来说，一个套接口就是网络上迸程通信的一端。套接字提供了应用层进程利用网络协议栈交换数据的机制；两个应用进程只要分别连接到自己的套接字，就能方便地通过计算机网络进行通信了，既不用去管网络的复杂结构，也不用去管数据传输的复杂过程。



![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=NTQ3MDBkOGMyYjkwY2VhMWNmZjVhMDQwYzI1NWQ4NjhfMTRjNzNiOGUxNjZkMTE5ZTU2Yzc1OTU3NmIwMWQxZTdfSUQ6NzUxMzU0MjQ5NDI0MDU3MTM5M18xNzgxMTQ0Njk1OjE3ODEyMzEwOTVfVjM)

# 原理

网络进程间面向连接的通信方式基于TCP，因而必须借助流式套接字来编程，应用程序分为服务器端和客户机端，双方是不对称的，需要分别编制。如图2所示为服务器端和客户机端操作流式套接字的基本步骤。

双方都首先要创建并安装套接字，做好准备后，才能进行客户端与服务器端的通信。一个完整的通信过程历经建立连接、发送/接收数据和释放连接3个阶段:建立连接的过程按照TCP三次握手的规范进行；发送接收数据阶段称为客户机与服务器的会话期，会话的内容是有一定格式的，一来一往的数据交换还必须遵守一定的顺序，这些都由应用层协议来规定；最后要释放连接

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=YjBjNzQwYTdjODc5YzJmZTU2ZDcyNDc3MmYwYTgwNGNfMmY4MjMyMzBkYzFhY2U0NzM4MjZkOTVlNzVmMmM4OTJfSUQ6NzUxMzU0ODQ3NDMxMTMxMTM2NF8xNzgxMTQ0Njk1OjE3ODEyMzEwOTVfVjM)

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ODc1N2IwMzViOGU0MTlmMjNkNmEwNDRiZjA0MjllZjJfNDExYTA0MGM2OTFhODc1MDliZjlkMzlhZDg4OTAzOTlfSUQ6NzUxMzU0ODY3MjU5ODkxNzEyM18xNzgxMTQ0Njk1OjE3ODEyMzEwOTVfVjM)

# 实现分析

首先使用C/S套接字编程实现

# 实现过程

## 准备工作

### 开发环境

此处使用MicroSoft VisualStudioCode Community 2022 \+ MinGW\-w64作为课设的设计程序

## 实际操作\(服务器端\)

### 初始化Winsock库（注意：必须在launch\.json先往args内添加 "\-lws2\_32"库）

Winsock编程必须先调用`WSAStartup（）`初始化库，程序结束时调用`WSACleanup（）`释放资源

```C++
#include <iostream>
#include <ws2tcpip.h>
#define OK 0
using namespace std;

int main(){
    cout <<"服务器"<<endl;

*    //1.初始化 Winsock*
    WSADATA wsaData;
    if(WSAStartup(MAKEWORD(2,2),&wsaData) != 0){
        cerr<< "WSAStartup 失败"<<endl;*  // cerr返回错误*
        return 1;*       // 返回 1 表示异常退出*
    }
    cout <<"Winsock 初始化成功!"<<endl;

    WSACleanup();
    return 0;
}
```

这里有几点注意点：
1\.cerr是立即输出无缓冲的一种错误消息提示方式，默认输出到屏幕，不容易被重定向

2\.`WSAStartup()`函数包含如下两项：

```C++
WORD wVersionRequested,  *// 请求的 Winsock 版本*
LPWSADATA lpWSAData  *// 输出参数，接收库的详细信息*
```

`MAKEWORD(2,2）`是一个宏，将两个直接的组合成一个WORD类型，此处用法为`MAKEWORD（主版本，副版本）`，请求的为Winsock 2\.2

`&wsaData`是传入一个WSADATA结构体的指针，函数会自动填充结构体，返回库的实际版本和实现细节

### 创建服务器套接字

套接字是网络通信的核心对象。创建套接字的第一步是利用`socket（）`函数，以下是socket函数的详细内容:

```C++
SOCKET socket(
  int af,        // 地址族（Address Family），如 IPv4 或 IPv6
  int type,      // 套接字类型（流式套接字、数据报套接字等）
  int protocol   // 协议（通常设为 0，自动选择）
);
```

利用此创建套接字：

```C++
*//2.创建套接字 *
    SOCKET serverSocket = socket(AF_INET,SOCK_STREAM,0);*   //AF_INET默认IPv4 , SOCK_STREAM默认TCP，0表示自动适配协议*
    if(serverSocket == INVALID_SOCKET){
        cerr<< "套接字创建失败！错误代码:"<< WSAGetLastError() <<endl;
        WSACleanup();
        return 1;
    }
    cout << "套接字创建成功！"<< endl;
```

在其常用参数中，`AF_INET`默认为IPv4协议，如果要使用v6地址族则参数为`AF_INET6`；`SOCK_STREAM`为面向连接的流式套接字，即TCP，`SOCK_DGRAM`表示UDP即不可靠传输。最后的`protocol`使用0时系统会根据`type`自动选择TCP或UDP协议。

这里使用AF\_INET的原因是因为IPv4协议地址中的地址（如127\.0\.0\.1）最常用，而且如果服务端使用了IPv6，则客户端也得支持IPv6

值得一提的是，套接字可以创建多个，但每个套接字都需要独立的SOCKET变量和错误检查。例如：一个用于监听，另一个用于与客户端通信。

### 绑定套接字到地址和端口

创建了套接字以后，还需要将套接字绑定到一个具体的IP地址和端口，这样客户端才可以通过IP地址
和端口找到并连接到服务器

首先了解一下绑定函数`bind()`:

```C++
int bind(
   SOCKET s,    *// 要绑定的套接字（上一步创建的 serverSocket）*
   const struct sockaddr *addr,    *// 指向地址结构的指针（包含 IP 和端口）*
   int namelen *// 地址结构的大小*
);
```

其中有着一个关键的结构体`sockaddr_in`，用于存储IPv4的地址和端口信息。这个结构体内部的内容为:

```C++
struct sockaddr_in{
    short          sin_family;     *// 地址族（AF_INET）*
    unsigned short sin_port;       *// 端口号（需转换为网络字节序）*
    struct in_addr sin_addr;       *// IP 地址（32 位 IPv4 地址）*
    char           sin_zero[8];    *// 填充字段（通常置零）*
};
```

在调用`bind()`函数前，我们需要根据刚刚所知道的`socketaddr_in`结构体的组成，对结构体中的内容进行填充，完成绑定前的参数设置,随后再根据`bind()`函数的作用，进行套接字与地址的绑定

```C++
*//3.绑定地址*
    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;* //IPv4*
    serverAddr.sin_port = htons(8080);* //端口号(8080),htons 转换字节序*
    serverAddr.sin_addr.s_addr = inet_addr("127.0.0.1");* //本地回环IP*
    memset(serverAddr.sin_zero,0,sizeof(serverAddr.sin_zero));* //填充字段置*
*    //调用bind()*
    if(bind(serverSocket, (sockaddr*)&serverAddr , sizeof(serverAddr)) == SOCKET_ERROR){
        cerr << "绑定地址失败！错误代码:" << WSAGetLastError() <<endl;
        closesocket(serverSocket); 
        WSACleanup();
        return 1;
    }
    cout << "绑定到 127.0.0.1:8080 成功！" << endl;

```

在计算机网络中，我们都学过了大端存储与小端存储的概念，一个利好人类阅读，一个利好机械阅读。在不同的存储设备中，采用的存储顺序可能不同。因此需要使用`htons()`函数来规定网络协议中的存储顺序。

`htons()` = Host To Network Short\(16位端口号\)，与其作用相同的还有`htonl()` = Host To Network Long\(32位IP地址\)。因此使用`htons()`处理端口，可以保证端口解析能够各种客户端上都能正常进行。 

而`inet_addr()`的原理是将IP地址例127\.0\.0\.1字符串按点分的方式拆分为四个字节，即"127","0","0","1",最后组合成32位的整数直接用于`sockaddr_in.sin.addr.s_addr`中。实际上这里有更复合现代的代替函数位`inet_pton()`,这个函数支持IPv6,更安全

### 监听连接

现在套接字已经成功绑定到地址和端口，接下来需要让服务器进入监听状态，时刻等待着客户端的连接。这是通过`listen()`函数实现的。

首先了解`listen()`函数的内容：

```C++
int listen(
  SOCKET s,          // 绑定的套接字（serverSocket）
  int    backlog     // 连接队列的最大长度（通常用 SOMAXCONN）
);
```

这里使用`SOMAXCONN`的原因是因为，在不同的操作系统中默认的最大值不同，比如:在Linux中默认最大值可能是128，但是在Windows中可能为200,`SOMAXCONN`可以使得系统自动选择最大值，避免手动设置成了不合理的值

随后利用`listen（）`函数进行监听代码的实现

```C++
*//4.开始监听*
    if(listen(serverSocket,SOMAXCONN) == SOCKET_ERROR){
        cerr<< "监听失败！错误代码:" << WSAGetLastError() << endl;
        closesocket(serverSocket);
        WSACleanup();
        return 1;
    }
    cout << "服务器正在监听127.0.0.1:8080 ..." <<endl;
```

其注意点与之前的近似相同，因此不再赘述，但需要提醒的是其工作原理和常见错误。监听队列监听到的为已完成三次握手，等待服务器调用`accept（）`函数接收的包，如果队列已满的话，新的连接请求会被拒绝，客户端会受到报错`ECONNREFUSED`

常见错误有：`WSAEINVAL`（套接字未绑定或者已经调用过了listen函数），`WSAENOTSOCK`\(传入的`serverSocket`无效\)

顺带一提的是，如果需要在这一步检查是否有监听\(`cmd`中输入`netstat -ano | findstr :8080 | findstr LISTENING`，随后查看结果\),则需要在`WSACleanup()`前加一个阻塞函数，让listen函数保持非关闭的情况下再去检测端口是否被监听，否则无法得到结果的

```C++
cout << "服务器正在监听127.0.0.1:8080 ..." <<endl;

*    //清理*
    cout << "按回车键退出..." << endl;
    cin.get();* // 阻塞等待用户输入*
    WSACleanup();
    return 0;
```

### 接受客户端连接

服务器成功监听127\.0\.0\.1：8080后，所需要做的就是在监听段有连接请求时接受并处理客户端连接。这里使用的是连接函数`accept()`，首先了解`accept()`的原型:

```C++
SCOKET accept(
   SOCKET s, //监听套接字(serverSocket)
   sockaddr *addr, //输出参数，接收客户端地址信息
   int  *addrlen // 输入输出参数，输出时返回实际写入的地址结构体长度
 );  
```

随后根据`accept()`函数完成接受客户端连接的部分：

```C++
*//5.接受客户端连接*
    cout<< "等待客户端连接中..."<<endl;
    sockaddr_in clientAddr;* //存储客户端地址信息*
    int clientAddrLen = sizeof(clientAddr);*  //地址结构体长度*
*    //调用accept()*
    SOCKET clientSocket = accept(serverSocket,(sockaddr*)&clientAddr , &clientAddrLen);
    if(clientSocket == INVALID_SOCKET){
        cerr << "客户端连接失败！错误代码:"<< WSAGetLastError() <<endl;
        closesocket(serverSocket);
        WSACleanup();
        return 1;
    }
*    //打印客户端连接信息*
    char clientIP[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &clientAddr.sin_addr , clientIP , INET_ADDRSTRLEN);
    cout<< "客户端已成功连接! 客户端IP: " << clientIP << " ,端口：" << ntohs(clientAddr.sin_port) <<endl;  
```



### 简单通信编写

于此给出一个比较简单的通信接收功能，这里请按自己的想法编写，示例中如有不懂的地方请自行查阅。

```C++
*// 6. 简单通信示例*
    char buffer[1024];
    int bytesReceived;
    while (true) {
        *// 接收客户端消息*
        bytesReceived = recv(clientSocket, buffer, sizeof(buffer), 0);
        if (bytesReceived <= 0) {
            cerr << "客户端断开连接或错误发生" << endl;
            break;
        }
        buffer[bytesReceived] = '\0'; *// 确保字符串终止*
        cout << "客户端说: " << buffer << endl;

        *// 发送回复*
        string reply;
        cout << "输入回复: ";
        getline(cin, reply);
        send(clientSocket, reply.c_str(), reply.size(), 0);
    }
```

至此，服务器端的创建告一段落，但是并没有完全结束。等接下来创建好客户端后再来进行进一步的优化协议。



## 实际操作（C\+\+客户端）

### 完整代码

客户端程序的流程与服务器类似，但不需要 `bind()` 和 `listen()`，而是用 `connect()` 连接服务器。所以这里直接给出完整代码

```C++
#include <iostream>
#include <ws2tcpip.h>
using namespace std;

int main() {
    cout << "客户端启动..." << endl;

    // 1. 初始化 Winsock
    WSADATA wsaData;
    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
        cerr << "WSAStartup 失败: " << WSAGetLastError() << endl;
        return 1;
    }

    // 2. 创建套接字
    SOCKET clientSocket = socket(AF_INET, SOCK_STREAM, 0);
    if (clientSocket == INVALID_SOCKET) {
        cerr << "socket() 失败: " << WSAGetLastError() << endl;
        WSACleanup();
        return 1;
    }

    // 3. 设置服务器地址
    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(8080); // 服务器端口
    serverAddr.sin_addr.s_addr = inet_addr("127.0.0.1"); // 服务器IP（本地回环）

    // 4. 连接到服务器
    if (connect(clientSocket, (sockaddr*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR) {
        cerr << "connect() 失败: " << WSAGetLastError() << endl;
        closesocket(clientSocket);
        WSACleanup();
        return 1;
    }
    cout << "已连接到服务器！" << endl;

    // 5. 与服务器通信
    char buffer[1024];
    while (true) {
        // 发送消息
        string message;
        cout << "输入消息: ";
        getline(cin, message);
        send(clientSocket, message.c_str(), message.size(), 0);

        // 接收回复
        int bytesReceived = recv(clientSocket, buffer, sizeof(buffer), 0);
        if (bytesReceived <= 0) {
            cerr << "服务器断开连接或错误发生" << endl;
            break;
        }
        buffer[bytesReceived] = '\0'; // 确保字符串终止
        cout << "服务器回复: " << buffer << endl;
    }

    // 6. 清理
    closesocket(clientSocket);
    WSACleanup();
    return 0;
}
```





# 优化内容

### HTML \+ WebSocket

#### 后端: C\+\+ WebSocket服务器

后端C\+\+WebSocket服务器使用WebSocket\+\+库，是纯粹的头文件，不需要编译依赖。作为后端的载体，我手头上有一台闲置的已穿透的NAS,其使用的系统为ArchLinux。以下是在Linux上操作的配置，如果没有了解需要的可以跳过。

首先确保Linux上已经安装了对应的运行环境

```Plain Text
sudo pacman -S gcc cmake make boost boost-libs openssl
```

安装完成后,进行文件的创建

```Plain Text
mkdir Study
cd Study
sudo touch websocket_server.cpp
```

随后来到了Cpp的编写过程 

首先,我们在创建好了的项目目录中新建过了websocket\_server\.cpp,现在对其编辑。

```C++
// websocket_server.cpp
#include <websocketpp/config/asio_no_tls.hpp> // WebSocket++ 的非加密配置
#include <websocketpp/server.hpp>             // WebSocket 服务器核心
#include <iostream>                         
#include <functional>

namespace ws = websocketpp;

typedef ws::server<ws::config::asio> ws_server;
```

首先是编写三个回调函数（即客户端完成特定事件后被调用执行的函数）：消息处理回调、连接建立回调与连接中断回调。

```Plain Text
void on_message(ws_server* *server* , ws::connection_hdl *hdl*, ws_server::message_ptr *msg*){
    std::cout << "收到消息" << msg->get_payload() << std::endl;
    server->send(hdl,"ECHO:" + msg->get_payload(),msg->get_opcode());
}
```

```Plain Text
void on_open(ws_server* *server* , ws::connection_hdl *hdl*){
    std::cout<< "新客户端连接" << std::endl;
}
```

```Plain Text
void on_close(ws_server* *server* , ws::connection_hdl *hdl*){
    std::cout<< "客户端断开" << std::endl;
}

```

随后是编写main函数,这里的操作和之前服务器端的操作相同，创建\-\>初始化\-\>监听\-\>接收\-\>通信

```C++
int main(){
*    //1.创建服务器实例*
    ws_server server;
    try{
*     //2.设置回调处理器(模板化)*
     server.set_message_handler(std::bind(&on_message,&server,std::placeholders::_1,std::placeholders::_2));
     server.set_open_handler(std::bind(&on_open,&server,std::placeholders::_1));
     server.set_close_handler(std::bind(&on_close,&server,std::placeholders::_1));
*     //3.初始化Asio*
     server.init_asio();
*     //4.监听窗口*
     server.listen(9002);
*     //5.接收连接*
     server.start_accept();
*     //6.启动服务器*
     std::cout<< "Websocket服务器已启动，监听端口：9002" << std::endl;
     server.run();
    }catch (const std::exception& e){
        std::cerr << "服务器错误" << e.what() << std::endl;
        return 1;
    }
        return 0;
}
```

到此为止，WebSocket\_server\.cpp就已经初具雏形了，配置fprc的过程于此不再赘述，这里贴出配置完后的用于测试的html代码

```XML
<!DOCTYPE html>
<html>
<body>
<script>
const ws = new WebSocket('ws://公网IP:公网端口');
ws.onopen = () => console.log("连接成功");
ws.onmessage = e => console.log("收到消息:", e.data);
ws.onerror = e => console.error("错误:", e);
</script>
</body>
</html>
```

配置完后，运行这个HTML，再通过F12打开网页的控制台,即可查看是否连接成功。以下为连接成功后的界面：

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZTNmYzI1MThkNmU5NzVjNzJkMmViZGY1MTdmYzU2MmNfNmFmY2JmOWE4Nzk0ZmE2ODY3OWYwYjE3MDU0MTYwMzdfSUQ6NzUxNzI1Njk3Nzk4NzM3MTAwOV8xNzgxMTQ0Njk1OjE3ODEyMzEwOTVfVjM)

但显然仅仅只是ECHO是远远不够的，到这里只是为了测试我的穿透以及 websocket协议的精简、高封装性。接下来我会着手尝试将服务器的websocket\_server当作中转站，实现两个网页的通信的建立，并且实现类似QQ群聊的聊天方式。




#Window Socket 非阻塞IO模型
--------------------------------------
当进程把一个套接字设置为非阻塞的时候，其实是在通知内核：当前socket所请求的I/O操作要把当前进程投入到睡眠等待的时候，不要进入睡眠状态，而是返回一个明确的错误。

#####socket中常见的会阻塞进程的函数：recv,send。</br>
* 阻塞模式</br>
recv,send 返回值为0时表示，客户端socket主动关闭。返回值为小于0时，表示发生错误，可以根据错误码来判断具体错误。返回值大于0表示接收到数据。
* 非阻塞</br>
recv,send 返回值=0时表示客户端主动断开连接。返回值=-1，如果WSAGetLastError()错误码为WSAEWOULDBLOCK（10035）表示非阻塞未接收到IO操作，立即返回的错误码。 返回值>0表示接收到数据

当一个进程对一个非阻塞socket进行循环调用 recv，send是，我们称之为轮序模式。应用进程持续轮序内核，查看某个操作是否就绪，这样往往会消耗大量的cpu时间，不过在某种简单特定的情况下也会使用到这种IO模型。

```c++
//Windows 下的非阻塞io socket
//采用轮序模式
//注意：windows下 当listen socket为非阻塞时 accept的客户端socket默认为非阻塞（所以不用再主动设置accept的socket为非阻塞模式）与linux不同

#include <winsock2.h>
#include<WS2tcpip.h>
#include <stdio.h>
#include <string.h>
#include <list>

#pragma comment(lib, "ws2_32.lib");

#define GETERROR WSAGetLastError()

struct SSocketClient {
	int nSocket;
	char ip[16];
};

std::list<SSocketClient> mClientList;


int OnRecv(int socket, char* buff, int len)
{
	if (!buff)
		return -1;

	int ret = recv(socket, buff, len, 0);
	if (ret == SOCKET_ERROR)
	{
		if (GETERROR == WSAEWOULDBLOCK) //非阻塞没有接收到数据立即返回
			ret = 0;
		else//根据错误码分析错误原因
			ret = -1;
	}
	else if (ret == 0) //客户端主动断开连接
		ret = -1;

	return ret;
}

int OnSend(int socket, char* buff, int len)
{
	if (!buff)
		return -1;

	int ret = send(socket, buff, len, 0);
	if (ret == SOCKET_ERROR)
	{
		if (GETERROR == WSAEWOULDBLOCK) //非阻塞没有接收到数据立即返回
			ret = 0;
		else//根据错误码分析错误原因
			ret = -1;
	}
	else if (ret == 0) //客户端主动断开连接
		ret = -1;

	return ret;
}

void OnDisconnect(const SSocketClient& client)
{
	closesocket(client.nSocket);
	printf("ip:%s disconnect.\n", client.ip);
}

int main()
{
	WORD vVersion;
	WSADATA wsaData;
	vVersion = MAKEWORD(2, 2);
	if (WSAStartup(vVersion, &wsaData) != 0)
	{
		printf("winsocket startup err!\n");
		return -1;
	}

	if (LOBYTE(wsaData.wVersion) != 2 || HIBYTE(wsaData.wHighVersion) != 2)
	{
		WSACleanup();
		return -1;
	}

	//创建socket
	int nListenSvrSocket = socket(AF_INET, SOCK_STREAM, 0);
	if (nListenSvrSocket == INVALID_SOCKET)
	{
		printf("create svrsocket err!\n");
		return -1;
	}

	//设置为非阻塞模式
	u_long arg = 1;
	if (ioctlsocket(nListenSvrSocket, FIONBIO, &arg) == SOCKET_ERROR)
	{
		printf("set noblock err!\n");
		return -1;
	}

	SOCKADDR_IN svrAddr;
	memset(&svrAddr, 0, sizeof(SOCKADDR_IN));
	svrAddr.sin_family = AF_INET;
	svrAddr.sin_port = htons(9001);
	svrAddr.sin_addr.S_un.S_addr = INADDR_ANY;

	if (bind(nListenSvrSocket, (sockaddr*)&svrAddr, sizeof(svrAddr)) == SOCKET_ERROR)
	{
		printf("bind err!\n");
		return -1;
	}

	if (listen(nListenSvrSocket, 5) == SOCKET_ERROR)
	{
		printf("listen err\n");
		return -1;
	}
	else
		printf("Listen %d begin.\n", 9001);

	bool bRunning = true;
	while (bRunning)
	{
		int ret = 0;
		std::list<std::list<SSocketClient>::iterator> dellist;
		SOCKADDR_IN clientAddr;
		int nClientSocket = accept(nListenSvrSocket, (sockaddr*)&clientAddr, NULL);
		if (nClientSocket != INVALID_SOCKET)
		{
			SSocketClient sClient;
			sClient.nSocket = nClientSocket;
			//char* tempIp = inet_ntoa(clientAddr.sin_addr);
			//if (tempIp)
			//	strcpy_s(sClient.ip, 16, tempIp);
			inet_ntop(AF_INET, (void*)&clientAddr.sin_addr, sClient.ip, 16);
			
			printf("ip:%s connect\n", sClient.ip);
			mClientList.push_back(sClient);
		}

		for (auto it = mClientList.begin(); it != mClientList.end(); ++it)
		{
			char buff[1024] = { 0 };
			const SSocketClient& sClient = *it;
			ret = OnRecv(sClient.nSocket, buff, 1024);
			if (ret > 0)
			{
				printf("ip:%s send [%s]\n", sClient.ip, buff);
				if (strcmp("exit", buff) == 0)
					bRunning = false;
			}
			else if (ret < 0)
			{
				OnDisconnect(sClient);
				dellist.push_back(it);
			}
		}

		for (auto it = dellist.begin(); it != dellist.end(); ++it)
			mClientList.erase(*it);
	}

	for (auto it = mClientList.begin(); it != mClientList.end(); ++it)
		OnDisconnect(*it);
	mClientList.clear();

	return 0;
}
```

接下来会给出轮序模式在linux下的实现代码。
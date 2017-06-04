#Linux下轮询式非阻塞socket

与windows下的类似。但是在设置接收客户端socket的时候要单独设置非阻塞socket。

* 阻塞模式

> recv,send 返回值为0时表示，客户端socket主动关闭。返回值为小于0时，表示发生错误，可以根据错误码来判断具体错误。返回值大于0表示接收到数据。

* 非阻塞

> recv,send 返回值=0时表示客户端主动断开连接。返回值=-1，如果errno
错误码为EWOULDBLOCK 表示非阻塞未接收到IO操作，立即返回的错误码。 返回值>0表示接收到数据

#####Linux创建返回错误
```
#define EAGAIN 11 // Try again
#define EINTR 4 // Interrupted system call
#define EWOULDBLOCK EAGAIN // Operation would block
```
#####例子
----------------------
```c++
#include <stdio.h>
#include <string.h>
#include <list>
#include <sys/time.h>
#include <stddef.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <sys/poll.h>

#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <sys/ioctl.h>
#undef __USE_GNU
#include <netdb.h>
#define __USE_GNU
#include <sys/errno.h>
#include <arpa/inet.h>

typedef int SOCKET;
typedef sockaddr_in			SOCKADDR_IN;
typedef sockaddr			SOCKADDR;
#define INVALID_SOCKET	    -1
#define SOCKET_ERROR        -1
#define GETERROR			errno
#define WSAEWOULDBLOCK		EWOULDBLOCK
#define CLOSESOCKET(s)		close(s)
#define IOCTLSOCKET(s,c,a)  ioctl(s,c,a)
#define CONN_INPRROGRESS	EINPROGRESS
/*
Linux下
#define EAGAIN 11 // Try again
#define EINTR 4 // Interrupted system call
#define EWOULDBLOCK EAGAIN // Operation would block
*/

struct SSocketClient {
	int nSocket;
	char ip[16];
};

std::list<SSocketClient> mClientList;

bool SetNonblock(int socket)
{
	//设置为非阻塞模式
	u_long arg = 1;
	if (IOCTLSOCKET(socket, FIONBIO, &arg) == SOCKET_ERROR)
	{
		printf("set noblock err!\n");
		return false;
	}
	return true;
}
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
	CLOSESOCKET(client.nSocket);
	printf("ip:%s disconnect.\n", client.ip);
}

int main(int narg, char** args)
{
	//创建socket
	int nListenSvrSocket = socket(AF_INET, SOCK_STREAM, 0);
	if (nListenSvrSocket == INVALID_SOCKET)
	{
		printf("create svrsocket err!\n");
		return -1;
	}

	//设置为非阻塞模式
	u_long arg = 1;
	if (IOCTLSOCKET(nListenSvrSocket, FIONBIO, &arg) == SOCKET_ERROR)
	{
		printf("set noblock err!\n");
		return -1;
	}

	SOCKADDR_IN svrAddr;
	memset(&svrAddr, 0, sizeof(SOCKADDR_IN));
	svrAddr.sin_family = AF_INET;
	svrAddr.sin_port = htons(9502);
	svrAddr.sin_addr.s_addr = htonl(INADDR_ANY);

	if (bind(nListenSvrSocket, (sockaddr*)&svrAddr, sizeof(svrAddr)) == SOCKET_ERROR)
	{
		printf("bind err! %d\n", GETERROR);
		return -1;
	}

	if (listen(nListenSvrSocket, 5) == SOCKET_ERROR)
	{
		printf("listen err\n");
		return -1;
	}
	else
		printf("Listen %d begin.\n", 9502);

	bool bRunning = true;
	while (bRunning)
	{
		int ret = 0;
		std::list<std::list<SSocketClient>::iterator> dellist;
		SOCKADDR_IN clientAddr;
		socklen_t nLen = sizeof(clientAddr);
		int nClientSocket = accept(nListenSvrSocket, (sockaddr*)&clientAddr, &nLen);
		if (nClientSocket != INVALID_SOCKET)
		{
			SSocketClient sClient;
			sClient.nSocket = nClientSocket;
			char* tempIp = inet_ntoa(clientAddr.sin_addr);
			if (tempIp)
				strcpy(sClient.ip, tempIp);

			if (!SetNonblock(sClient.nSocket))
				OnDisconnect(sClient);
			else
				mClientList.push_back(sClient);
		}

		for (std::list<SSocketClient>::iterator it = mClientList.begin(); it != mClientList.end(); ++it)
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

		for (std::list<std::list<SSocketClient>::iterator>::iterator it = dellist.begin(); it != dellist.end(); ++it)
			mClientList.erase(*it);
	}

	for (std::list<SSocketClient>::iterator it = mClientList.begin(); it != mClientList.end(); ++it)
		OnDisconnect(*it);
	mClientList.clear();

	return 0;
}
```
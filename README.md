# network_programming
TCP socket编程的难点在于对消息的定界与解读，如何控制缓存的读取。

socket只负责消息按序到达接收端，但不会对消息进行定界，因此需要应用开发者负责消息定界，即在发送的消息头部加入额外的信息，指明本次消息的类型和长度以及其他一些必要的信息。

如何做到，我们自己建立一个消息的缓冲区，同时，在消息的最前面加一个标识用来标识本条消息的长度。

详细请参考：

* [socket数据传输过程中如何准确的接收消息](https://blog.csdn.net/kwsy2008/article/details/49156821)

# 接收端
1. 在接收端会开辟一段缓存，recv()函数负责把接收到的网络数据包存放在缓存中。要保留两个变量readOffset和writeOffset。应用程序要根据readOffset确定数据的边界。判断一个数据包是否接收完的方式是
```
g_uiWriteOffset >= (g_uiReadOffset+sizeof(t_struMsgSendReq))%SOCKET_RECV_UNIT_MAX_LEN
```
2. 一个完整数据有可能在缓存中连续存储，也有可能一部分在缓存的最后，一部分在缓存的起始。因此正确获取一个完整数据包的方式为

		// 如果收到的数据是连续存储的
		if(g_uiReadOffset+sizeof(t_struMsgSendReq)<SOCKET_RECV_UNIT_MAX_LEN)
		{
			t_struMsgSendReq = *(STRU_PC_2_DSP_MSG_SEND_REQ*)(g_aucSocketDlBuf+g_uiReadOffset);
		}
		else
		{
		// 如果收到的数据一部分在数组g_aucSocketDlBuf的后面，一部分在数组g_aucSocketDlBuf的前面
			memcpy((char*)&t_struMsgSendReq,g_aucSocketDlBuf+g_uiReadOffset,SOCKET_RECV_UNIT_MAX_LEN-g_uiReadOffset);
			memcpy((char*)&t_struMsgSendReq+(SOCKET_RECV_UNIT_MAX_LEN-g_uiReadOffset),g_aucSocketDlBuf,(g_uiReadOffset+sizeof(t_struMsgSendReq))%SOCKET_RECV_UNIT_MAX_LEN);
		}

3. 在使用recv函数时，要保证将接收到的网络数据包存放到缓存里，并保持缓存的循环性。recv有一个参数是指本次**最多**读取多少个字节，使用这个参数就可以保持缓存的循环性。代码如下

		cnt = (int)recv( g_soActive, g_aucSocketDlBuf+g_uiWriteOffset, t_uiFreeLenth, 0 );
		//SendDataRequest(PACKET_SIZE);
		if( cnt > 0 )
		{
			g_uiUsedLenth += cnt;
			//这里假定一次接收到的数据不会大于Buffer长度
			if( ((g_uiWriteOffset<g_uiReadOffset) && ((g_uiWriteOffset+cnt)>g_uiReadOffset))	\
					|| ((g_uiWriteOffset>g_uiReadOffset) && ((g_uiWriteOffset+cnt)>(g_uiReadOffset+SOCKET_RECV_UNIT_MAX_LEN))) )
			{
				printf("出错！ Socket写进Driver未读走的部分 \n");
			}

			// 更新写操作起始，下次从更新的写标识开始
			g_uiWriteOffset = (g_uiWriteOffset+cnt) % SOCKET_RECV_UNIT_MAX_LEN;

			// 每recv一次，就唤醒驱动检查是否接收完一个完整意义的包
			Semaphore_post(SEM_DRV_RECV);
		}
		else
		{
			fdClose( s );
			s = INVALID_SOCKET;
		}

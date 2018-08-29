# network_programming

# 接收端
1. 在接收端会开辟一段缓存，recv()函数负责把接收到的网络数据包存放在缓存中。要保留两个变量readOffset和writeOffset。应用程序要根据readOffset确定数据的边界。判断一个数据包是否接收完的方式是
```
g_uiWriteOffset >= (g_uiReadOffset+sizeof(t_struMsgSendReq))%SOCKET_RECV_UNIT_MAX_LEN
```
2. 一个完整数据有可能在缓存中连续存储，也有可能一部分在缓存的最后，一部分在缓存的起始。因此正确获取一个完整数据包的方式为
```
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
```
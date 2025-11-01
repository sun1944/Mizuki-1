---
title: 基于TCP的Socket网络通信的斗地主游戏开发
published: 2025-08-30
description: "基于TCP的Socket网络通信的斗地主游戏开发。"
image: "https://typorytp.oss-cn-hangzhou.aliyuncs.com/login_0614.png"
tags: ["网络", "UGUI", "Socket","C#"]
category: Unity
draft: false
---

# 基于TCP的Socket网络通信的斗地主游戏开发

## 网络框架

本项目采用TCP的方式进行网络通信。

### 数据缓冲区

这个 ByteArray 类是一个字节数组缓冲区的实现，主要用于网络编程中高效地处理字节数据的读写操作，类似于一个环形队列的用法，进行读写操作，**每一个客户端实例在服务器都会对应一个缓冲区**。

~~~cs
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ByteArray 
{
    /// <summary>
    /// 默认大小
    /// </summary>
    const int DEFAULT_SIZE = 1024;
    /// <summary>
    /// 初始大小
    /// </summary>
    private int initSize;
    /// <summary>
    /// 字节数组
    /// </summary>
    public byte[] bytes;
    /// <summary>
    /// 读的位置
    /// </summary>
    public int readIndex;
    /// <summary>
    /// 写的位置
    /// </summary>
    public int writeIndex;
    /// <summary>
    /// 容量
    /// </summary>
    private int capacity;
    /// <summary>
    /// 读写之间的长度
    /// </summary>
    public int Length { get { return writeIndex - readIndex; } }
    /// <summary>
    /// 数组余量
    /// </summary>
    public int Remain { get { return capacity-writeIndex; } }
    /// <summary>
    /// 提供长度的构造函数
    /// </summary>
    /// <param name="size">数组长度</param>
    public ByteArray(int size = DEFAULT_SIZE)
    {
        bytes=new byte[size];
        initSize = size;
        capacity = size;
        readIndex = 0;
        writeIndex = 0;
    }
    /// <summary>
    /// 提供字节数组的构造函数
    /// </summary>
    /// <param name="defaultBytes">字节数组</param>
    public ByteArray(byte[] defaultBytes)
    {
        bytes = defaultBytes;
        initSize = defaultBytes.Length;
        capacity = defaultBytes.Length;
        readIndex=0;
        writeIndex=defaultBytes.Length;
    }
    /// <summary>
    /// 移动数据
    /// </summary>
    public void MoveBytes()
    {
        if (Length > 0)
        {
            Array.Copy(bytes, readIndex, bytes, 0, Length);
        }
        writeIndex = Length;
        readIndex = 0;
    }
    /// <summary>
    /// 重设尺寸
    /// </summary>
    /// <param name="size">新的长度</param>
    public void ReSize(int size)
    {
        if (size < Length)
            return;
        if(size<initSize)
            return;
        capacity = size;
        byte[] newBytes = new byte[capacity];   
        Array.Copy(bytes, readIndex, newBytes, 0, Length);
        bytes = newBytes;
        writeIndex=Length;
        readIndex = 0;
    }
}

~~~



### 协议

协议是客户端和服务端共同遵守的规则，双方的定义应当是一致的。首先为了解决分包粘包问题，我们使用长度信息法来控制。此时协议就定义成了如下所示的结构：

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/58242747417936a7663bd488b5485ded.jpg" alt="58242747417936a7663bd488b5485ded" style="zoom: 15%;" />

这里定义了一个协议基类，有协议名和编码解码的实现方法，用于消息的封装和解析。

~~~cs
using System;
using System.Collections;
using System.Collections.Generic;
using System.Diagnostics;
using System.Text;
using Newtonsoft.Json;
#nullable disable
public class MsgBase
{
    public string protoName = "";
    /// <summary>
    /// 编码
    /// </summary>
    /// <param name="msgBase"></param>
    /// <returns></returns>
    public static byte[] Encode(MsgBase msgBase)
    {
        string s = JsonConvert.SerializeObject(msgBase);
        return Encoding.UTF8.GetBytes(s);
    }
    /// <summary>
    /// 解码
    /// </summary>
    /// <param name="protoName">协议名</param>
    /// <param name="bytes">字节数组</param>
    /// <param name="offset">起始位置</param>
    /// <param name="count">要转码的数量</param>
    /// <returns></returns>
    public static MsgBase Decode(string protoName, byte[] bytes, int offset, int count)
    {
        string s = Encoding.UTF8.GetString(bytes, offset, count);
        return JsonConvert.DeserializeObject(s,Type.GetType(protoName)) as MsgBase;
    }
    /// <summary>
    /// 编码协议名
    /// </summary>
    /// <param name="msgBase"></param>
    /// <returns></returns>
    public static byte[] EncodeName(MsgBase msgBase)
    {
        byte[] nameBytes = Encoding.UTF8.GetBytes(msgBase.protoName);
        short len = (short)nameBytes.Length;
        byte[] bytes = new byte[len + 2];
        bytes[0] = (byte)(len % 256);
        bytes[1] = (byte)(len / 256);
        Array.Copy(nameBytes, 0, bytes, 2, len);
        return bytes;
    }
    /// <summary>
    /// 解码
    /// </summary>
    /// <param name="bytes"></param>
    /// <param name="offset"></param>
    /// <param name="count"></param>
    /// <returns></returns>
    public static string DecodeName(byte[] bytes, int offset, out int count)
    {
        count = 0;
        if (offset + 2 > bytes.Length)
            return "";
        short len = (short)(bytes[offset + 1] * 256 + bytes[offset]);
        if (len <= 0)
            return "";
        count = 2 + len;
        return Encoding.UTF8.GetString(bytes,offset+2, len);
    }
}

~~~

### NetManager

这是网络通信的核心管理类，用于控制网络的连接与数据的收发。

#### 服务端

服务端使用多路复用的方式控制数据的接收和客户端的连接。这里定义了一个客户端实例和检测列表，select对检测列表进行操作，对于新建连接的客户端会先加入客户端实例列表，再加入检测列表用于后续数据的接收。

由于使用的阻塞方法和单线程实现多路复用，在高并发的情况下有较差的表现。

~~~cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Sockets;
using System.Reflection;
using System.Text;
using System.Threading.Tasks;
#nullable disable

public static class NetManager
{
    /// <summary>
    /// 服务端Socket
    /// </summary>
    public static Socket listenfd;
    /// <summary>
    /// 客户端字典
    /// </summary>
    public static Dictionary<Socket,ClientState> clients = new Dictionary<Socket,ClientState>();
    /// <summary>
    /// 用于检测的List
    /// </summary>
    private static List<Socket> sockets =new List<Socket>();
    /// <summary>
    /// 时间间隔
    /// </summary>
    public static long pingInterval = 30;


    /// <summary>
    /// 连接
    /// </summary>
    /// <param name="ip">IP地址</param>
    /// <param name="port">端口号</param>
    public static void Connect(string ip,int port)
    {
        listenfd=new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        IPAddress iPAddress = IPAddress.Parse(ip);
        IPEndPoint iPEndPoint =new IPEndPoint(iPAddress, port);
        listenfd.Bind(iPEndPoint);
        listenfd.Listen(0);

        Console.WriteLine("[服务器]启动成功");
        while (true)
        {
            //填充List
            sockets.Clear();
            sockets.Add(listenfd);
            foreach (ClientState s in clients.Values)
            {
                sockets.Add(s.socket);
            }

            Socket.Select(sockets, null, null, 1000);
            for (int i = 0; i < sockets.Count; i++)
            {
                Socket s = sockets[i];
                if (s == listenfd)
                {
                    //有客户端连接需要Accept
                    Accept(s);
                }
                else
                {
                    //有客户端发过来消息需要Receive
                    Receive(s);
                }
            }
            Timer();
        }
    }
    /// <summary>
    /// 接收客户端Socket
    /// </summary>
    /// <param name="listenfd">服务端Socket</param>
    public static void Accept(Socket listenfd)
    {
        try
        {
            Socket clientfd=listenfd.Accept();
            Console.WriteLine("Accept " + clientfd.RemoteEndPoint.ToString());
            ClientState state = new ClientState();
            state.lastPingTime = GetTimeStamp();
            state.socket=clientfd;
            clients.Add(clientfd, state);
        }
        catch (SocketException ex)
        {
            Console.WriteLine("Accept fail " + ex.ToString());
        }
    }
    /// <summary>
    /// 接收消息
    /// </summary>
    /// <param name="clientfd">发信息的客户端</param>
    public static void Receive(Socket clientfd)
    {
        ClientState state = clients[clientfd];
        ByteArray readBuff=state.readBuff;


        int count = 0;
        if (readBuff.Remain <= 0)
        {
            readBuff.MoveBytes();
        }
        if (readBuff.Remain <= 0)
        {
            Console.WriteLine("Receive fail " + "数组长度不足");
            Close(state);
            return;
        }
        try
        {
            count = clientfd.Receive(readBuff.bytes, readBuff.writeIndex, readBuff.Remain, 0);
        }
        catch (SocketException ex)
        {
            Console.WriteLine("Receive fail " + ex.ToString());
            Close(state);
            return;
        }
        //关闭
        if (count <= 0)
        {
            Console.WriteLine("Socket Close "+clientfd.RemoteEndPoint.ToString());
            Close(state);
            return;
        }
        readBuff.writeIndex += count;
        //解码
        OnReceiveData(state);
        readBuff.MoveBytes();
    }
    /// <summary>
    /// 关闭
    /// </summary>
    /// <param name="state"></param>
    public static void Close(ClientState state)
    {
        //调用OnDisconnect
        MethodInfo mei = typeof(EventHandler).GetMethod("OnDisconnect");
        object[] ob = { state };
        mei.Invoke(null, ob);
        //关闭
        state.socket.Close();
        clients.Remove(state.socket);
    }
    /// <summary>
    /// 处理消息
    /// </summary>
    /// <param name="state"></param>
    public static void OnReceiveData(ClientState state)
    {
        ByteArray readBuff = state.readBuff;
        byte[] bytes=readBuff.bytes;
        if (readBuff.Length <= 2)
            return;
        //解析数字
        short bodyLength = (short)(bytes[readBuff.readIndex + 1] * 256 + bytes[readBuff.readIndex]);
        if (readBuff.Length < bodyLength)
            return;
        readBuff.readIndex += 2;
        //解析协议名
        int nameCount=0;
        string protoName = MsgBase.DecodeName(readBuff.bytes, readBuff.readIndex, out nameCount);
        if (protoName == "")
        {
            Console.WriteLine("OnReceiveData fail", "解析协议名失败");
            Close(state);
            return;
        }
        readBuff.readIndex += nameCount;
        //解析协议体
        int bodyCount = bodyLength - nameCount;
        MsgBase msgBase = MsgBase.Decode(protoName, readBuff.bytes, readBuff.readIndex, bodyCount);
        readBuff.readIndex+=bodyCount;
        readBuff.MoveBytes();
        //分发消息
        MethodInfo mi = typeof(MsgHandler).GetMethod(protoName);
        Console.WriteLine("Receive " + protoName);
        if(mi != null)
        {
            object[] o = { state, msgBase };
            mi.Invoke(null, o);
        }
        else
        {
            Console.WriteLine("OnReceiveData调用Msg函数失败");
        }
        //继续处理
        if (readBuff.Length > 2)
        {
            OnReceiveData(state);
        }
    }
    /// <summary>
    /// 发送数据
    /// </summary>
    /// <param name="cs">发给的客户端</param>
    /// <param name="msgBase">发送的数据</param>
    public static void Send(ClientState cs,MsgBase msgBase)
    {
        if (cs == null || !cs.socket.Connected)
            return;
        //编码
        byte[] nameBytes = MsgBase.EncodeName(msgBase);
        byte[] bodyBytes = MsgBase.Encode(msgBase);
        int len = nameBytes.Length + bodyBytes.Length;
        byte[] sendBytes=new byte[len+2];
        sendBytes[0] = (byte)(len % 256);
        sendBytes[1] = (byte)(len / 256);
        //拷贝到sendBytes
        Array.Copy(nameBytes,0,sendBytes,2,nameBytes.Length);
        Array.Copy(bodyBytes,0,sendBytes,2+nameBytes.Length,bodyBytes.Length);
        try
        {
            cs.socket.Send(sendBytes, 0, sendBytes.Length, 0);
        }
        catch (SocketException ex)
        {
            Console.WriteLine("Send fail " + ex.ToString());
        }

    }
    /// <summary>
    /// 计时器
    /// </summary>
    private static void Timer()
    {
        MethodInfo mei = typeof(EventHandler).GetMethod("OnTimer");
        object[] ob = { };
        mei.Invoke(null, ob);
    }
    /// <summary>
    /// 获取时间戳
    /// </summary>
    public static long GetTimeStamp()
    {
        TimeSpan ts = DateTime.UtcNow - new DateTime(1970, 1, 1, 0, 0, 0, 0);
        return Convert.ToInt64(ts.TotalSeconds);
    }
}


~~~

#### 客户端

客户端不需要维护多个网络连接实例，只需要使用异步的方式用于信息的发送与接收即可。

~~~cs
using System;
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using System.Net.Sockets;
using System.Text;
using UnityEngine;
using UnityEngine.UIElements;

public static class NetManager
{
    /// <summary>
    /// 客户端Socket
    /// </summary>
    private static Socket socket;
    /// <summary>
    /// 字节数组
    /// </summary>
    private static ByteArray byteArray;
    /// <summary>
    /// 写入队列
    /// </summary>
    private static Queue<ByteArray> writeQueue;
    /// <summary>
    /// 是否正在连接
    /// </summary>
    private static bool isConnecting;
    /// <summary>
    /// 是否正在关闭
    /// </summary>
    private static bool isClosing;
    /// <summary>
    /// 消息列表
    /// </summary>
    private static List<MsgBase> msgList = new List<MsgBase>();
    /// <summary>
    /// 消息列表长度
    /// </summary>
    private static int msgCount = 0;
    /// <summary>
    /// 一帧处理消息的最大长度
    /// </summary>
    private static int processMsgCount = 10;

    /// <summary>
    /// 是否启用心跳机制
    /// </summary>
    public static bool isUsePing = true;
    /// <summary>
    /// 心跳间隔
    /// </summary>
    public static int pingInterval = 30;
    /// <summary>
    /// 上一次发送Ping协议的时间
    /// </summary>
    private static float lastPingTime = 0;
    /// <summary>
    /// 上一次收到Pong协议的时间
    /// </summary>
    private static float lastPongTime = 0;


    /// <summary>
    /// 连接状态
    /// </summary>
    public enum NetEvent
    {
        ConnectSucc = 1,
        ConnectFail = 2,
        Close = 3,
    }
    /// <summary>
    /// 事件委托
    /// </summary>
    /// <param name="err"></param>
    public delegate void EventListener(string err);
    /// <summary>
    /// 事件监听列表
    /// </summary>
    public static Dictionary<NetEvent, EventListener> eventListeners = new Dictionary<NetEvent, EventListener>();
    /// <summary>
    /// 添加事件
    /// </summary>
    /// <param name="netEvent"></param>
    /// <param name="listener"></param>
    public static void AddEventListener(NetEvent netEvent, EventListener listener)
    {
        if (eventListeners.ContainsKey(netEvent))
        {
            eventListeners[netEvent] += listener;
        }
        else
        {
            eventListeners[netEvent] = listener;
        }
    }
    /// <summary>
    /// 删除事件
    /// </summary>
    /// <param name="netEvent"></param>
    /// <param name="listener"></param>
    public static void RemoveEventListener(NetEvent netEvent, EventListener listener)
    {
        if (eventListeners.ContainsKey(netEvent))
        {
            eventListeners[netEvent] -= listener;
            if (eventListeners[netEvent] == null)
            {
                eventListeners.Remove(netEvent);
            }
        }
    }
    /// <summary>
    /// 分发事件
    /// </summary>
    /// <param name="netEvent"></param>
    /// <param name="err"></param>
    private static void FireEvent(NetEvent netEvent, string err)
    {
        if (eventListeners.ContainsKey(netEvent))
        {
            eventListeners[netEvent](err);
        }
    }
    /// <summary>
    /// 事件委托
    /// </summary>
    /// <param name="msgBase"></param>
    public delegate void MsgListener(MsgBase msgBase);
    /// <summary>
    /// 消息事件列表
    /// </summary>
    private static Dictionary<string, MsgListener> msgListeners = new Dictionary<string, MsgListener>();
    /// <summary>
    /// 添加消息事件
    /// </summary>
    /// <param name="msgName"></param>
    /// <param name="listener"></param>
    public static void AddMsgListener(string msgName, MsgListener listener)
    {
        if (msgListeners.ContainsKey(msgName))
        {
            msgListeners[msgName] += listener;
        }
        else
        {
            msgListeners[msgName] = listener;
        }
    }
    /// <summary>
    /// 删除消息事件
    /// </summary>
    /// <param name="msgName"></param>
    /// <param name="listener"></param>
    public static void RemoveMsgListener(string msgName, MsgListener listener)
    {
        if (msgListeners.ContainsKey(msgName))
        {
            msgListeners[msgName] -= listener;
            if (msgListeners[msgName] == null)
            {
                msgListeners.Remove(msgName);
            }
        }
    }
    /// <summary>
    /// 分发事件
    /// </summary>
    /// <param name="msgName"></param>
    /// <param name="msgBase"></param>
    private static void FireMsg(string msgName, MsgBase msgBase)
    {
        if (msgListeners.ContainsKey(msgName))
        {
            msgListeners[msgName](msgBase);
        }
    }
    /// <summary>
    /// 连接
    /// </summary>
    /// <param name="ip">ip地址</param>
    /// <param name="port">端口号</param>
    public static void Connect(string ip, int port)
    {
        if (socket != null && socket.Connected)
        {
            Debug.Log("连接失败，已经连接了");
            return;
        }
        if (isConnecting)
        {
            Debug.Log("正在连接");
            return;
        }
        Init();
        isConnecting = true;
        socket.BeginConnect(ip, port, ConnectCallback, socket);
    }
    /// <summary>
    /// 连接回调
    /// </summary>
    /// <param name="ar"></param>
    private static void ConnectCallback(IAsyncResult ar)
    {
        try
        {
            Socket socket = (Socket)ar.AsyncState;
            socket.EndConnect(ar);
            Debug.Log("连接成功");
            isConnecting = false;
            FireEvent(NetEvent.ConnectSucc, "");
            //接收
            socket.BeginReceive(byteArray.bytes, byteArray.writeIndex, byteArray.Remain, 0, ReceiveCallback, socket);
        }
        catch (SocketException ex)
        {
            Debug.Log("连接失败" + ex.ToString());
            FireEvent(NetEvent.ConnectFail, ex.ToString());
            isConnecting = false;
        }
    }
    /// <summary>
    /// 初始化
    /// </summary>
    private static void Init()
    {
        socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        byteArray = new ByteArray();
        writeQueue = new Queue<ByteArray>();
        isConnecting = false;
        isClosing = false;

        msgList = new List<MsgBase>();
        msgCount = 0;

        lastPingTime = Time.time;
        lastPongTime = Time.time;

        if (!msgListeners.ContainsKey("MsgPong"))
        {
            AddMsgListener("MsgPong", OnMsgPong);
        }
    }
    /// <summary>
    /// 关闭
    /// </summary>
    public static void Close()
    {
        if (socket == null || !socket.Connected)
            return;
        if (isConnecting)
            return;
        //还有剩余消息
        if (writeQueue.Count > 0)
        {
            isClosing = true;
        }
        //没有剩余消息，直接关闭
        else
        {
            socket.Close();
            FireEvent(NetEvent.Close, "");
        }
    }
    /// <summary>
    /// 发送消息
    /// </summary>
    /// <param name="msg">消息</param>
    public static void Send(MsgBase msg)
    {
        //检测
        if (socket == null || !socket.Connected)
            return;
        if (isConnecting)
            return;
        if (isClosing)
            return;
        //编码
        byte[] nameBytes = MsgBase.EncodeName(msg);
        byte[] bodyBytes = MsgBase.Encode(msg);
        int len = nameBytes.Length + bodyBytes.Length;
        byte[] sendBytes = new byte[len + 2];
        sendBytes[0] = (byte)(len % 256);
        sendBytes[1] = (byte)(len / 256);
        Array.Copy(nameBytes, 0, sendBytes, 2, nameBytes.Length);
        Array.Copy(bodyBytes, 0, sendBytes, 2 + nameBytes.Length, bodyBytes.Length);

        ByteArray ba = new ByteArray(sendBytes);
        int count = 0;
        lock (writeQueue)
        {
            writeQueue.Enqueue(ba);
            count = writeQueue.Count;
        }
        if (count == 1)
        {
            //发送
            socket.BeginSend(sendBytes, 0, sendBytes.Length, 0, SendCallback, socket);
        }
    }
    /// <summary>
    /// 发送回调
    /// </summary>
    /// <param name="ar"></param>
    private static void SendCallback(IAsyncResult ar)
    {
        Socket socket = ar.AsyncState as Socket;
        if (socket == null || !socket.Connected)
            return;
        int count = socket.EndSend(ar);

        ByteArray ba;
        lock (writeQueue)
        {
            ba = writeQueue.First();
        }

        ba.readIndex += count;
        if (ba.Length == 0)
        {
            lock (writeQueue)
            {
                writeQueue.Dequeue();
                ba = writeQueue.First();
            }
        }
        //继续发送
        if (ba != null)
        {
            socket.BeginSend(ba.bytes, ba.readIndex, ba.Length, 0, SendCallback, socket);
        }
        else if (isClosing)
        {
            socket.Close();
        }
    }
    /// <summary>
    /// 接收回调
    /// </summary>
    /// <param name="ar"></param>
    public static void ReceiveCallback(IAsyncResult ar)
    {
        try
        {
            Socket socket = ar.AsyncState as Socket;
            int count = socket.EndSend(ar);
            //断开
            if (count == 0)
            {
                Close();
                return;
            }
            //成功接收
            byteArray.writeIndex += count;
            OnReceiveData();
            //ByteArray长度过小，扩容
            if (byteArray.Remain < 8)
            {
                byteArray.MoveBytes();
                byteArray.ReSize(byteArray.Length * 2);
            }
            //继续接收消息
            socket.BeginReceive(byteArray.bytes, byteArray.writeIndex, byteArray.Remain, 0, ReceiveCallback, socket);
        }
        catch (SocketException ex)
        {
            Debug.Log("接收失败" + ex.ToString());
        }
    }
    /// <summary>
    /// 处理接收来的消息
    /// </summary>
    public static void OnReceiveData()
    {
        if (byteArray.Length <= 2)
            return;
        int readIndex = byteArray.readIndex;
        byte[] bytes = byteArray.bytes;
        //bytes[readIndex]%256  bytes[readIndex+1]/256
        short bodyLength = (short)(bytes[readIndex + 1] * 256 + bytes[readIndex]);
        if (byteArray.Length < bodyLength + 2)
            return;

        byteArray.readIndex += 2;
        //解析消息名
        int nameCount = 0;
        string protoName = MsgBase.DecodeName(byteArray.bytes, byteArray.readIndex, out nameCount);

        if (protoName == "")
        {
            Debug.Log("解析失败");
            return;
        }
        byteArray.readIndex += nameCount;
        //解析消息体
        int bodyCount = bodyLength - nameCount;
        MsgBase msgBase = MsgBase.Decode(protoName, byteArray.bytes, byteArray.readIndex, bodyCount);
        byteArray.readIndex += bodyCount;
        byteArray.MoveBytes();
        lock (msgList)
        {
            msgList.Add(msgBase);
        }
        msgCount++;
        if (byteArray.Length > 2)
        {
            OnReceiveData();
        }
    }
    /// <summary>
    /// 消息处理，把消息放到msgList当中处理，后续放到Update里面执行
    /// </summary>
    public static void MsgUpdate()
    {
        if (msgCount == 0)
            return;
        for (int i = 0; i < processMsgCount; i++)
        {
            MsgBase msgBase = null;
            lock (msgList)
            {
                if (msgCount > 0)
                {
                    msgBase = msgList[0];
                    msgList.RemoveAt(0);
                    msgCount--;
                }
            }
            if (msgBase != null)
            {
                FireMsg(msgBase.protoName, msgBase);
            }
            else
            {
                break;
            }
        }
    }
    /// <summary>
    /// 发送Ping协议
    /// </summary>
    private static void PingUpdate()
    {
        if (!isUsePing)
            return;
        //发送消息
        if (Time.time - lastPingTime > pingInterval)
        {
            MsgPing msg = new MsgPing();
            Send(msg);
            lastPingTime = Time.time;
        }

        //pingInterval*4时间内接收不到Pong断开
        if(Time.time-lastPongTime>pingInterval*4)
        {
            Close();
        }
    }
    /// <summary>
    /// 这个Update需要在脚本中调用执行
    /// </summary>
    public static void Update()
    {
        MsgUpdate();
        PingUpdate();
    }
    /// <summary>
    /// 接收到Pong消息后
    /// </summary>
    /// <param name="msgBase"></param>
    private static void OnMsgPong(MsgBase msgBase)
    {
        lastPongTime = Time.time;
    }
}

~~~

#### 消息读取与写入

在客户端中定义了一个写入队列和一个消息列表，消息列表用于接收，写入队列用于发送。

~~~cs
	/// <summary>
    /// 写入队列
    /// </summary>
    private static Queue<ByteArray> writeQueue;
	/// <summary>
    /// 消息列表
    /// </summary>
    private static List<MsgBase> msgList = new List<MsgBase>();
~~~

在接收消息时会向列表中写入，在游戏中每帧处理固定大小内数量的消息，而不是直接在每帧中执行，防止阻塞造成的影响。

~~~cs
/// <summary>
    /// 处理接收来的消息
    /// </summary>
    public static void OnReceiveData()
    {
        if (byteArray.Length <= 2)
            return;
        int readIndex = byteArray.readIndex;
        byte[] bytes = byteArray.bytes;
        //bytes[readIndex]%256  bytes[readIndex+1]/256
        short bodyLength = (short)(bytes[readIndex + 1] * 256 + bytes[readIndex]);
        if (byteArray.Length < bodyLength + 2)
            return;

        byteArray.readIndex += 2;
        //解析消息名
        int nameCount = 0;
        string protoName = MsgBase.DecodeName(byteArray.bytes, byteArray.readIndex, out nameCount);

        if (protoName == "")
        {
            Debug.Log("解析失败");
            return;
        }
        byteArray.readIndex += nameCount;
        //解析消息体
        int bodyCount = bodyLength - nameCount;
        MsgBase msgBase = MsgBase.Decode(protoName, byteArray.bytes, byteArray.readIndex, bodyCount);
        byteArray.readIndex += bodyCount;
        byteArray.MoveBytes();
        lock (msgList)
        {
            msgList.Add(msgBase);
        }
        msgCount++;
        if (byteArray.Length > 2)
        {
            OnReceiveData();
        }
    }
 	/// <summary>
    /// 消息处理，把消息放到msgList当中处理，后续放到Update里面执行
    /// </summary>
    public static void MsgUpdate()
    {
        if (msgCount == 0)
            return;
        for (int i = 0; i < processMsgCount; i++)
        {
            MsgBase msgBase = null;
            lock (msgList)
            {
                if (msgCount > 0)
                {
                    msgBase = msgList[0];
                    msgList.RemoveAt(0);
                    msgCount--;
                }
            }
            if (msgBase != null)
            {
                FireMsg(msgBase.protoName, msgBase);
            }
            else
            {
                break;
            }
        }
    }
~~~

我们使用写入队列的目的也是为了保证数据能够完整的发送出去，如果在一次send之后数据没有完全发送，则可以通过队列存储，在发送不完整时取出第一条剩余部分再次发送。

~~~cs
/// <summary>
    /// 发送消息
    /// </summary>
    /// <param name="msg">消息</param>
    public static void Send(MsgBase msg)
    {
        //检测
        if (socket == null || !socket.Connected)
            return;
        if (isConnecting)
            return;
        if (isClosing)
            return;
        //编码
        byte[] nameBytes = MsgBase.EncodeName(msg);
        byte[] bodyBytes = MsgBase.Encode(msg);
        int len = nameBytes.Length + bodyBytes.Length;
        byte[] sendBytes = new byte[len + 2];
        sendBytes[0] = (byte)(len % 256);
        sendBytes[1] = (byte)(len / 256);
        Array.Copy(nameBytes, 0, sendBytes, 2, nameBytes.Length);
        Array.Copy(bodyBytes, 0, sendBytes, 2 + nameBytes.Length, bodyBytes.Length);

        ByteArray ba = new ByteArray(sendBytes);
        int count = 0;
        lock (writeQueue)
        {
            writeQueue.Enqueue(ba);
            count = writeQueue.Count;
        }
        if (count == 1)
        {
            //发送
            socket.BeginSend(sendBytes, 0, sendBytes.Length, 0, SendCallback, socket);
        }
    }
    /// <summary>
    /// 发送回调
    /// </summary>
    /// <param name="ar"></param>
    private static void SendCallback(IAsyncResult ar)
    {
        Socket socket = ar.AsyncState as Socket;
        if (socket == null || !socket.Connected)
            return;
        int count = socket.EndSend(ar);

        ByteArray ba;
        lock (writeQueue)
        {
            ba = writeQueue.First();
        }

        ba.readIndex += count;
        if (ba.Length == 0)
        {
            lock (writeQueue)
            {
                writeQueue.Dequeue();
                ba = writeQueue.First();
            }
        }
        //继续发送
        if (ba != null)
        {
            socket.BeginSend(ba.bytes, ba.readIndex, ba.Length, 0, SendCallback, socket);
        }
        else if (isClosing)
        {
            socket.Close();
        }
    }
~~~

#### 心跳机制

心跳机制是检测客户端连接状态的一种手段，客户端每隔一段事件向服务端发送一条特殊的数据包，服务端收到后也会返回客户端一条接收到的消息，当这个过程中某一方没有收到对方消息时则断开连接。

**客户端**

客户端每帧中执行每隔一段时间发送一次Ping协议，在设定的时间间隔内收不到服务端的Pong协议则自己断开连接。

~~~cs
	/// <summary>
    /// 发送Ping协议
    /// </summary>
    private static void PingUpdate()
    {
        if (!isUsePing)
            return;
        //发送消息
        if (Time.time - lastPingTime > pingInterval)
        {
            MsgPing msg = new MsgPing();
            Send(msg);
            lastPingTime = Time.time;
        }

        //pingInterval*4时间内接收不到Pong断开
        if(Time.time-lastPongTime>pingInterval*4)
        {
            Close();
        }
    }
	/// <summary>
    /// 接收到Pong消息后
    /// </summary>
    /// <param name="msgBase"></param>
    private static void OnMsgPong(MsgBase msgBase)
    {
        lastPongTime = Time.time;
    }
~~~

**服务端**

每次收到Ping协议后都会返回一个Pong协议。

```cs
public static void MsgPing(ClientState c, MsgBase msgBase)
{
    Console.WriteLine("MsgPing");
    c.lastPingTime = NetManager.GetTimeStamp();
    MsgPong msgPong = new MsgPong();
    NetManager.Send(c, msgPong);
}
```

### 委托回调

接受到协议的消息后要执行相关的游戏逻辑，所以很自然就能想到委托。

#### 服务端

服务端创建了一个MsgHandler类定义了若干游戏逻辑的方法，在服务端解析消息的时候，通过C#的反射来动态获取这些方法名。每个方法都有相同的参数，都用send返回服务端处理的数据结果。

~~~cs
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
#nullable disable

public class MsgHandler
{
    #region Heartbeat
    public static void MsgPing(ClientState c, MsgBase msgBase)
    {
        Console.WriteLine("MsgPing");
        c.lastPingTime = NetManager.GetTimeStamp();
        MsgPong msgPong = new MsgPong();
        NetManager.Send(c, msgPong);
    }
    #endregion
    #region Login
    /// <summary>
    /// 注册
    /// </summary>
    /// <param name="c"></param>
    /// <param name="msgBase"></param>
    public static void MsgRegister(ClientState c, MsgBase msgBase)
    {
        MsgRegister msg = msgBase as MsgRegister;
        if (DbManager.Register(msg.id, msg.pw))
        {
            DbManager.CreadtePlayer(msg.id);
            msg.result = true;
        }
        else
        {
            msg.result = false;
        }
        NetManager.Send(c, msg);
    }
    /// <summary>
    /// 登录
    /// </summary>
    /// <param name="c"></param>
    /// <param name="msgBase"></param>
    public static void MsgLogin(ClientState c, MsgBase msgBase)
    {
        MsgLogin msg = msgBase as MsgLogin;
        //检验密码
        if (!DbManager.CheckPassword(msg.id, msg.pw))
        {
            msg.result = false;
            NetManager.Send(c, msg);
            return;
        }
        //检验是否登录
        if (c.player != null)
        {
            msg.result = false;
            NetManager.Send(c, msg);
            return;
        }
        //踢下线
        if (PlayerManager.IsOnline(msg.id))
        {
            Player otherPlayer = PlayerManager.GetPlayer(msg.id);
            MsgKick msgKick = new MsgKick();
            msgKick.isKick = true;
            otherPlayer.Send(msgKick);
            NetManager.Close(otherPlayer.state);
        }

        PlayerData playerData = DbManager.GetPlayerData(msg.id);
        if (playerData == null)
        {
            msg.result = false;
            NetManager.Send(c, msg);
            return;
        }
        //创建玩家
        Player player = new Player(c);
        player.id = msg.id;
        player.data = playerData;
        PlayerManager.AddPlayer(msg.id, player);
        c.player = player;
        msg.result = true;
        player.Send(msg);
    }
    #endregion
    #region Room
    /// <summary>
    /// 获取玩家数据信息
    /// </summary>
    /// <param name="c"></param>
    /// <param name="msgBase"></param>
    public static void MsgGetAchieve(ClientState c, MsgBase msgBase)
    {
        MsgGetAchieve msg = msgBase as MsgGetAchieve;
        Player player = c.player;
        if (player == null)
            return;
        msg.bean = player.data.bean;
        player.Send(msg);
    }
    ......
}
~~~

接收消息时根据消息名称调用相关方法

~~~cs
 		//分发消息
        MethodInfo mi = typeof(MsgHandler).GetMethod(protoName);
        Console.WriteLine("Receive " + protoName);
        if(mi != null)
        {
            object[] o = { state, msgBase };
            mi.Invoke(null, o);
        }
        else
        {
            Console.WriteLine("OnReceiveData调用Msg函数失败");
        }
~~~

#### 客户端

客户端这里使用了委托列表的方式来实现。UI等组件只需要通过AddMsgListener方法就可以绑定相关事件。

~~~cs
/// <summary>
    /// 事件委托
    /// </summary>
    /// <param name="msgBase"></param>
    public delegate void MsgListener(MsgBase msgBase);
    /// <summary>
    /// 消息事件列表
    /// </summary>
    private static Dictionary<string, MsgListener> msgListeners = new Dictionary<string, MsgListener>();
    /// <summary>
    /// 添加消息事件
    /// </summary>
    /// <param name="msgName"></param>
    /// <param name="listener"></param>
    public static void AddMsgListener(string msgName, MsgListener listener)
    {
        if (msgListeners.ContainsKey(msgName))
        {
            msgListeners[msgName] += listener;
        }
        else
        {
            msgListeners[msgName] = listener;
        }
    }
    /// <summary>
    /// 删除消息事件
    /// </summary>
    /// <param name="msgName"></param>
    /// <param name="listener"></param>
    public static void RemoveMsgListener(string msgName, MsgListener listener)
    {
        if (msgListeners.ContainsKey(msgName))
        {
            msgListeners[msgName] -= listener;
            if (msgListeners[msgName] == null)
            {
                msgListeners.Remove(msgName);
            }
        }
    }
    /// <summary>
    /// 分发事件
    /// </summary>
    /// <param name="msgName"></param>
    /// <param name="msgBase"></param>
    private static void FireMsg(string msgName, MsgBase msgBase)
    {
        if (msgListeners.ContainsKey(msgName))
        {
            msgListeners[msgName](msgBase);
        }
    }
~~~

msgList存储从服务端发来的协议数据，每个协议可以在客户端中绑定若干个委托，这样在每帧执行的时候就是取出一个协议数据，执行相关委托。

~~~cs
	/// <summary>
    /// 消息处理，把消息放到msgList当中处理，后续放到Update里面执行
    /// </summary>
    public static void MsgUpdate()
    {
        if (msgCount == 0)
            return;
        for (int i = 0; i < processMsgCount; i++)
        {
            MsgBase msgBase = null;
            lock (msgList)
            {
                if (msgCount > 0)
                {
                    msgBase = msgList[0];
                    msgList.RemoveAt(0);
                    msgCount--;
                }
            }
            if (msgBase != null)
            {
                FireMsg(msgBase.protoName, msgBase);
            }
            else
            {
                break;
            }
        }
    }
~~~

在接收数据时解析数据之后把消息存入消息列表内。

~~~cs
 		//解析消息体
        int bodyCount = bodyLength - nameCount;
        MsgBase msgBase = MsgBase.Decode(protoName, byteArray.bytes, byteArray.readIndex, bodyCount);
        byteArray.readIndex += bodyCount;
        byteArray.MoveBytes();
        lock (msgList)
        {
            msgList.Add(msgBase);
        }
        msgCount++;
~~~

### 数据库连接

这个无需多说，经典的DB操作。。。。

~~~cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Microsoft.Win32;
using MySqlConnector;
using System.Text.RegularExpressions;
using Newtonsoft.Json;
using System.Security.Cryptography;
#nullable disable
public class DbManager
{
    /// <summary>
    /// 数据库对象
    /// </summary>
    public static MySqlConnection mysql;
    /// <summary>
    /// 连接数据库
    /// </summary>
    /// <param name="db">数据表</param>
    /// <param name="ip">IP地址</param>
    /// <param name="port">端口号</param>
    /// <param name="user">用户名</param>
    /// <param name="pw">密码</param>
    /// <returns></returns>
    public static bool Connect(string db, string ip, int port, string user, string pw)
    {
        mysql = new MySqlConnection();
        string s = string.Format("Database={0};Data Source={1};port={2};User Id={3};Password={4}", db, ip, port, user, pw);
        mysql.ConnectionString = s;
        try
        {
            mysql.Open();
            Console.WriteLine("[数据库]启动成功");
            return true;
        }
        catch (Exception e)
        {
            Console.WriteLine("[数据库]启动失败" + e.Message);
            return false;
        }
    }
    /// <summary>
    /// 判断字符串是否安全
    /// </summary>
    /// <param name="str"></param>
    /// <returns></returns>
    private static bool IsSafeString(string str)
    {
        return !Regex.IsMatch(str, @"[-|;|,|\/|\[|\]|\{|\}|%|@|\*|!|\']");
    }
    /// <summary>
    /// 判断账号是否存在
    /// </summary>
    /// <param name="id"></param>
    /// <returns></returns>
    public static bool IsAccountExist(string id)
    {
        if (!IsSafeString(id))
            return true;
        //SQL语句
        //string s = $"select * from account where id={id}";
        string s = string.Format("select * from account where id='{0}'", id);
        try
        {
            //创建执行脚本对象
            MySqlCommand cmd = new MySqlCommand(s, mysql);
            MySqlDataReader dataReader = cmd.ExecuteReader();
            bool result = dataReader.HasRows;
            dataReader.Close();
            return result;
        }
        catch (Exception e)
        {
            Console.WriteLine("[数据库] IsAccountExist fail " + e.Message);
            return true;
        }
    }
    /// <summary>
    /// 注册
    /// </summary>
    /// <param name="id">id</param>
    /// <param name="pw">密码</param>
    /// <returns></returns>
    public static bool Register(string id, string pw)
    {
        //防止SQL注入
        if (!IsSafeString(id))
        {
            Console.WriteLine("[数据库]注册失败，id不安全");
            return false;
        }
        if (!IsSafeString(pw))
        {
            Console.WriteLine("[数据库]注册失败，密码不安全");
            return false;
        }
        if (IsAccountExist(id))
        {
            Console.WriteLine("[数据库]注册失败，账号存在");
            return false;
        }
        pw = GetMD5(pw);
        //SQL语句
        string s=string.Format("insert into account set id='{0}',pw='{1}';",id,pw);
        try
        {
            MySqlCommand cmd = new MySqlCommand(s, mysql);
            cmd.ExecuteNonQuery();
            return true;
        }
        catch (Exception e)
        {
            Console.WriteLine("[数据库]注册失败，"+e.Message);
            return false;
        }
    }
    /// <summary>
    /// 创建玩家
    /// </summary>
    /// <param name="id">id</param>
    /// <returns></returns>
    public static bool CreadtePlayer(string id)
    {
        //防止SQL注入
        if (!IsSafeString(id))
        {
            Console.WriteLine("[数据库]创建角色失败，id不安全");
            return false;
        }
        PlayerData playerData = new PlayerData();
        string data=JsonConvert.SerializeObject(playerData);
        //SQL语句
        string s = string.Format("insert into player set id='{0}',data='{1}';", id, data);
        try
        {
            MySqlCommand cmd = new MySqlCommand(s, mysql);
            cmd.ExecuteNonQuery();
            return true;
        }
        catch (Exception e)
        {
            Console.WriteLine("[数据库]创建失败，" + e.Message);
            return false;
        }
    }
    /// <summary>
    /// 检查密码
    /// </summary>
    /// <param name="id">id</param>
    /// <param name="pw">密码</param>
    /// <returns></returns>
    public static bool CheckPassword(string id,string pw)
    {
        //防止SQL注入
        if (!IsSafeString(id))
        {
            Console.WriteLine("[数据库]检查密码失败，id不安全");
            return false;
        }
        if (!IsSafeString(pw))
        {
            Console.WriteLine("[数据库]检查密码失败，密码不安全");
            return false;
        }
        pw = GetMD5(pw);
        //SQL语句
        string s=string.Format("select * from account where id='{0}' and pw='{1}';",id, pw);
        try
        {
            MySqlCommand cmd = new MySqlCommand(s, mysql);
            MySqlDataReader dataReader = cmd.ExecuteReader();
            bool result = dataReader.HasRows;
            dataReader.Close();
            return result;
        }
        catch (Exception e)
        {
            Console.WriteLine("[数据库]检查密码失败" + e.Message);
            return false;
        }
    }
    /// <summary>
    /// 获取玩家信息
    /// </summary>
    /// <param name="id">id</param>
    /// <returns></returns>
    public static PlayerData GetPlayerData(string id)
    {
        //防止SQL注入
        if (!IsSafeString(id))
        {
            Console.WriteLine("[数据库] 获取角色信息失败，id不安全");
            return null;
        }
        //SQL语句
        string s = string.Format("select * from player where id='{0}';", id);
        try
        {
            MySqlCommand cmd = new MySqlCommand(s, mysql);
            MySqlDataReader dataReader = cmd.ExecuteReader();
            bool result = dataReader.HasRows;
            if (!result)
            {
                dataReader.Close();
                return null;
            }
            //读取信息
            dataReader.Read();
            string data = dataReader.GetString("data");

            //反序列化
            PlayerData playerData=JsonConvert.DeserializeObject<PlayerData>(data);
            dataReader.Close();
            return playerData;
        }
        catch (Exception e)
        {
            Console.WriteLine("[数据库] 获取玩家信息失败" + e.Message);
            return null;
        }
    }
    /// <summary>
    /// 更新玩家信息
    /// </summary>
    /// <param name="id">id</param>
    /// <param name="playerData">玩家信息</param>
    /// <returns></returns>
    public static bool UpdatePlayerData(string id,PlayerData playerData)
    {
        string data=JsonConvert.SerializeObject(playerData);
        //SQL语句
        string s = string.Format("update player set data='{0}' where id='{1}';", data, id);
        try
        {
            MySqlCommand cmd = new MySqlCommand(s, mysql);
            cmd.ExecuteNonQuery();
            return true;
        }
        catch (Exception e)
        {
            Console.WriteLine("[数据库] 更新玩家数据失败" + e.Message);
            return false;
        }
    }
    public static string GetMD5(string input)
    {
        MD5 md5 = MD5.Create();
        byte[] bytes = md5.ComputeHash(Encoding.UTF8.GetBytes(input));
        StringBuilder builder = new StringBuilder();
        for (int i = 0; i < bytes.Length; i++)
        {
            builder.Append(bytes[i].ToString("x2"));
        }
        return builder.ToString();
    }
}


~~~

## 游戏业务逻辑

### 大致实现思路

以玩家刷新房间列表为例，玩家进入大厅或者点击刷新时会获取房间列表信息，房间列表包含房间ID，房间内人数，是否开始三个信息，在房间列表面板中，点击刷新或者进入时向服务端发送MsgGetRoomList协议，在面板初始化时注册该协议方法；服务端接收消息时，**会根据协议名称通过反射机制执行相关协议的方法**，获取房间列表信息，再发送回给相应客户端。客户端受到这个协议数据包后进行解析，将其存入msgList中，每帧处理若干个消息，**并执行绑定的相关委托**。面板中注册好了获取房间列表相关的逻辑操作，于是开始生成item显示在屏幕上。

### UI框架

定义面板基类，定义打开，关闭，初始化的方法。

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class BasePanel : MonoBehaviour
{
    /// <summary>
    /// 加载路径
    /// </summary>
    public string skinPath;
    /// <summary>
    /// 面板
    /// </summary>
    public GameObject skin;
    /// <summary>
    /// 层级
    /// </summary>
    public PanelManager.Layer layer=PanelManager.Layer.Panel;
    /// <summary>
    /// 初始化
    /// </summary>
    public void Init()
    {
        Debug.Log(skinPath);
        skin = Instantiate(Resources.Load<GameObject>(skinPath));
    }
    public virtual void OnInit()
    {

    }
    public virtual void OnShow(params object[] para)
    {

    }
    public virtual void OnClose()
    {

    }
    public void Close()
    {
        string name = GetType().ToString();
        PanelManager.Close(name);
    }
}

~~~

UIManager控制面板的打开，关闭和初始化。

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public static class PanelManager
{
    /// <summary>
    /// 层级
    /// </summary>
    public enum Layer
    {
        Panel,
        Tip
    }
    /// <summary>
    /// 层级列表
    /// </summary>
    private static Dictionary<Layer,Transform> layers=new Dictionary<Layer,Transform>();
    /// <summary>
    /// 面板列表
    /// </summary>
    private static Dictionary<string,BasePanel> panels=new Dictionary<string,BasePanel>();
    /// <summary>
    /// 根目录
    /// </summary>
    private static Transform root;
    /// <summary>
    /// 画布
    /// </summary>
    private static Transform canvas;
    /// <summary>
    /// 初始化
    /// </summary>
    public static void Init()
    {
        root = GameObject.Find("Root").transform;
        canvas = root.Find("Canvas");
        layers.Add(Layer.Panel, canvas.Find("Panel"));
        layers.Add(Layer.Tip, canvas.Find("Tip"));
    }
    /// <summary>
    /// 打开面板
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="para"></param>
    public static void Open<T>(params object[] para)where T:BasePanel
    {
        //是否已经打开
        string name = typeof(T).ToString();
        if (panels.ContainsKey(name))
            return;
        BasePanel panel = root.gameObject.AddComponent<T>();
        panel.OnInit();
        panel.Init();

        Transform layer = layers[panel.layer];
        panel.skin.transform.SetParent(layer, false);
        panels.Add(name, panel);
        panel.OnShow(para);
    }
    /// <summary>
    /// 关闭面板
    /// </summary>
    /// <param name="name"></param>
    public static void Close(string name)
    {
        //是否已经打开
        if (!panels.ContainsKey(name))
            return;
        BasePanel panel=panels[name];
        panel.OnClose();
        panels.Remove(name);
        GameObject.Destroy(panel.skin);
        GameObject.Destroy(panel);
    }
}

~~~

### 登录注册

其实这些面板套路无非就是定义UI组件，寻找组件，绑定UI事件，注册协议委托方法，常规逻辑。

~~~cs
using Newtonsoft.Json.Bson;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class LoginPanel : BasePanel
{
    //组件
    private InputField idInput;
    private InputField pwInput;
    private Button loginButton;
    private Button registerButton;
    public override void OnInit()
    {
        skinPath = "LoginPanel";
        layer=PanelManager.Layer.Panel;
    }
    public override void OnShow(params object[] para)
    {
        //寻找组件
        idInput = skin.transform.Find("IdInput").GetComponent<InputField>();
        pwInput = skin.transform.Find("PwInput").GetComponent<InputField>();
        loginButton = skin.transform.Find("LoginButton").GetComponent<Button>();
        registerButton = skin.transform.Find("RegisterButton").GetComponent<Button>();

        //添加事件
        loginButton.onClick.AddListener(OnLoginClick);
        registerButton.onClick.AddListener(OnRegisterClick);

        NetManager.AddEventListener(NetManager.NetEvent.ConnectSucc, OnConnectSucc);
        NetManager.AddEventListener(NetManager.NetEvent.ConnectFail, OnConnectFail);
        //网络协议监听
        NetManager.AddMsgListener("MsgLogin", OnMsgLogin);
        NetManager.Connect("127.0.0.1",8888);
    }
    public override void OnClose()
    {
        NetManager.RemoveEventListener(NetManager.NetEvent.ConnectSucc, OnConnectSucc);
        NetManager.RemoveEventListener(NetManager.NetEvent.ConnectFail, OnConnectFail);
        
        NetManager.RemoveMsgListener("MsgLogin", OnMsgLogin);
    }
    public void OnLoginClick()
    {
        if(idInput.text==""||pwInput.text=="")
        {
            PanelManager.Open<TipPanel>("用户名的密码不能为空");
            return;
        }
        MsgLogin msgLogin = new MsgLogin();
        msgLogin.id=idInput.text;
        msgLogin.pw = pwInput.text;
        NetManager.Send(msgLogin);
    }
    public void OnRegisterClick()
    {
        //打开注册面板
        PanelManager.Open<RegisterPanel>();
    }
    public void OnMsgLogin(MsgBase msgBase)
    {
        MsgLogin msg = msgBase as MsgLogin;
        if (msg.result)
        {
            //登陆成功
            PanelManager.Open<TipPanel>("登陆成功");
            GameManager.id=msg.id;
            PanelManager.Open<RoomListPanel>();


            Close();
        }
        else
        {
            //登录失败
            PanelManager.Open<TipPanel>("登录失败");
        }
    }
    public void OnConnectSucc(string err)
    {
        Debug.Log("连接成功");
    }
    public void OnConnectFail(string err)
    {
        PanelManager.Open<TipPanel>(err);
    }
}

~~~

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class RegisterPanel : BasePanel
{
    private InputField idInput;
    private InputField pwInput;
    private InputField repInput;
    private Button registerButton;
    private Button closeButton;
    public override void OnInit()
    {
        skinPath = "RegisterPanel";
        layer = PanelManager.Layer.Panel;
    }
    public override void OnShow(params object[] para)
    {
        //寻找组件
        idInput = skin.transform.Find("IdInput").GetComponent<InputField>();
        pwInput = skin.transform.Find("PwInput").GetComponent<InputField>();
        repInput = skin.transform.Find("RepInput").GetComponent<InputField>();
        registerButton = skin.transform.Find("RegisterButton").GetComponent<Button>();
        closeButton = skin.transform.Find("CloseButton").GetComponent<Button>();


        registerButton.onClick.AddListener(OnRegisterClick);
        closeButton.onClick.AddListener(OnCloseClick);

        NetManager.AddMsgListener("MsgRegister", OnMsgRegister);
    }
    public override void OnClose()
    {
        NetManager.RemoveMsgListener("MsgRegister", OnMsgRegister);
    }
    public void OnRegisterClick()
    {
        if (idInput.text == "" || pwInput.text == "" )
        {
            PanelManager.Open<TipPanel>("用户名和密码不能为空");
            return;
        }
        if (pwInput.text != repInput.text)
        {
            PanelManager.Open<TipPanel>("两次密码不一致");
            return;
        }

        //发送注册协议
        MsgRegister msg =new MsgRegister();
        msg.id = idInput.text;
        msg.pw = pwInput.text;
        NetManager.Send(msg);
    }
    public void OnCloseClick()
    {
        Close();
    }
    public void OnMsgRegister(MsgBase msgBase)
    {
        MsgRegister msg=msgBase as MsgRegister;
        if (msg.result)
        {
            //注册成功
            PanelManager.Open<TipPanel>("注册成功");
            Close();
        }
        else
        {
            //注册失败
            PanelManager.Open<TipPanel>("注册失败");
        }
    }
}

~~~

服务端相关协议方法

~~~cs
    #region Login
    /// <summary>
    /// 注册
    /// </summary>
    /// <param name="c"></param>
    /// <param name="msgBase"></param>
    public static void MsgRegister(ClientState c, MsgBase msgBase)
    {
        MsgRegister msg = msgBase as MsgRegister;
        if (DbManager.Register(msg.id, msg.pw))
        {
            DbManager.CreadtePlayer(msg.id);
            msg.result = true;
        }
        else
        {
            msg.result = false;
        }
        NetManager.Send(c, msg);
    }
    /// <summary>
    /// 登录
    /// </summary>
    /// <param name="c"></param>
    /// <param name="msgBase"></param>
    public static void MsgLogin(ClientState c, MsgBase msgBase)
    {
        MsgLogin msg = msgBase as MsgLogin;
        //检验密码
        if (!DbManager.CheckPassword(msg.id, msg.pw))
        {
            msg.result = false;
            NetManager.Send(c, msg);
            return;
        }
        //检验是否登录
        if (c.player != null)
        {
            msg.result = false;
            NetManager.Send(c, msg);
            return;
        }
        //踢下线
        if (PlayerManager.IsOnline(msg.id))
        {
            Player otherPlayer = PlayerManager.GetPlayer(msg.id);
            MsgKick msgKick = new MsgKick();
            msgKick.isKick = true;
            otherPlayer.Send(msgKick);
            NetManager.Close(otherPlayer.state);
        }

        PlayerData playerData = DbManager.GetPlayerData(msg.id);
        if (playerData == null)
        {
            msg.result = false;
            NetManager.Send(c, msg);
            return;
        }
        //创建玩家
        Player player = new Player(c);
        player.id = msg.id;
        player.data = playerData;
        PlayerManager.AddPlayer(msg.id, player);
        c.player = player;
        msg.result = true;
        player.Send(msg);
    }
    #endregion
~~~

### **房间**

由于关于房间的逻辑大部分都是在服务端进行判定的，客户端只需要获取房间列表等信息就足够了，没有什么逻辑运算，所以具体的房间类可以在服务端设定。

游戏是在服务端上去处理关键逻辑运算的，所以游戏是基于**状态同步**来实现的。

对于一个房间，房间需要有ID，需要有一个容量为三的玩家列表来管理状态，同时房间也是由房主发起和开始游戏，所以还需要获取房主ID。

在未开始游戏时玩家可以离开或进入房间，玩家可以准备或不准备，开始游戏后，三个玩家需要决定是否叫地主，房间内有玩家会成为地主，游戏胜负也会因为身份来判定。

房间内的玩家行为应当同步，需要执行广播方法进行同步。

具体逻辑不过多赘述，以下只讲解大致关键实现过程。

### 卡牌

卡牌需要按照特定的形式进行组合才能使用，例如对子、三带一等等，还要比较牌型的大小。为尽可能防止作弊，客户端只发送卡牌序列，这种特殊的运算逻辑在服务端CardManager类中实现。

客户端需要通过点击和按住多选来选择要出的卡牌，通过IPointerDownHandler, IPointerEnterHandler两个接口检测是否按下和指针是否进入游戏对象来实现。

### 战斗

整体流程是从房主开始，选择叫或者不叫地主，然后再传给其他玩家决定是否抢地主，服务端根据是否抢地主的信息随机选择地主，地主确定后开始发牌。当战斗结束后谁不剩牌那方赢。

```
准备 → 叫地主 → [抢地主] → 明牌 → 出牌 → 结束 → 重新准备
```
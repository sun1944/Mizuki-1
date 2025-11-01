---
title: Socket网络通信
published: 2025-07-14
description: "Socket网络通信知识讲解。"
image: "https://typorytp.oss-cn-hangzhou.aliyuncs.com/titlesocket01.png"
tags: ["网络", "Socket","C#"]
category: Unity
draft: false
---

# Socket网络通信

## 计算机网络基础

### Socket简介

  Socket（套接字）是网络通信的底层抽象接口，用于实现不同设备间的数据交换。其本质是操作系统提供的编程接口，通过封装网络协议栈实现应用程序间的通信。

1. **通信端点：**

   - 每个参与网络通信的程序都需要创建一个 Socket。
   - Socket 绑定了一个 **IP 地址**（标识主机）和一个 **端口号**（标识主机上的特定进程/服务）。
   - 一对 Socket（源 IP:Port 和目标 IP:Port）唯一标识了一次网络连接的两端。

2. **传输层协议的接口：**

   - Socket API 是操作系统提供给应用程序使用网络协议（主要是 TCP 和 UDP）的编程接口。
   - 应用程序通过调用 Socket API（如 `socket()`, `bind()`, `listen()`, `accept()`, `connect()`, `send()`, `recv()`, `close()` 等）来建立连接、发送和接收数据。
   - 它屏蔽了底层网络协议栈（IP 路由、数据包封装/解封装、错误处理等）的复杂性。

3. **支持多种协议：**

   - **流式 Socket (SOCK_STREAM)：** 通常使用 **TCP 协议**。提供可靠的、面向连接的、基于字节流的通信。数据按顺序到达，无差错、无丢失、无重复。类似打电话。
   - **数据报 Socket (SOCK_DGRAM)：** 通常使用 **UDP 协议**。提供不可靠的、无连接的、基于数据报（消息边界）的通信。速度快，但不保证顺序、可能丢失或重复。类似寄明信片。
   - **原始 Socket (SOCK_RAW)：** 允许应用程序直接访问更底层的网络协议（如 IP 或 ICMP），通常用于网络诊断或开发新协议。

   ### 三次握手与四次挥手

   **基本符号含义**

   1. `seq` (Sequence Number - 序列号)：用于标识发送方当前数据段的起始字节序号，初始seq是随机的，后续数据段的 seq = 前一个数据段的 seq + 前一个数据段的数据长度。
   2. `ack` (Acknowledgment Number - 确认号)：用于标识接收方期望收到的下一个字节的序号，ack = 收到的 seq + 数据长度（如果收到SYN/FIN，则 ack = 收到的 seq + 1）。
   3. `SYN` (Synchronize - 同步标志)
   4. `ACK` (Acknowledgment - 确认标志)
   5. `FIN` (Finish - 结束标志)

   **三次握手**

   

   <img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/de3ba839a24f475cbdbbd9b9c2f041d7.png" alt="de3ba839a24f475cbdbbd9b9c2f041d7" style="zoom: 33%;" />

   <img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/168c431527c0438080089de11f2dedbc.png" alt="168c431527c0438080089de11f2dedbc"  />

   ### TCP/UDP

   

## 基于TCP的Socket网络连接

### 基本流程

首先绑定服务端的IP地址和端口号，启动进入监听状态，客户端通过**三次握手**与服务端建立连接，服务端接收请求后可以实现双向通信了，传输结束后使用**四次挥手**断开连接。

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/0e49faa75f8058df3cef969315753e3b.jpeg" alt="0e49faa75f8058df3cef969315753e3b" style="zoom: 50%;" />

### 同步通信

举一个同步的Socket的方法如下，程序按照上述流程图进行连接和通信，由于网络通信都是基于字节流来进行传输的，所以需要使用byte[]类型的数据进行转化。

**服务端**

~~~cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading.Tasks;
 
namespace UnityServer
{
    internal class Server
    {
        public static void Main()
        {
            //定义第一个socket，用于监听
            Socket serverSocket=new Socket(AddressFamily.InterNetwork,SocketType.Stream, ProtocolType.Tcp);
            //绑定ip和端口
            IPAddress ip=IPAddress.Parse("127.0.0.1");
            IPEndPoint iPEndPoint = new IPEndPoint(ip, 8888);
            serverSocket.Bind(iPEndPoint);
            //监听
            serverSocket.Listen(0);
            Console.WriteLine("服务端启动成功");
            while (true)
            {
                //定义了第二个Socket，用于表示连上的客户端Socket，每个客户端都有一个，处理数据的发送与接收
                Socket clientSocket = serverSocket.Accept();
                Console.WriteLine("服务端Accept");
                //Receive
                byte[] readBuffer=new byte[1024];
                int count = clientSocket.Receive(readBuffer);
                string readStr = Encoding.UTF8.GetString(readBuffer, 0, count);
                Console.WriteLine("接收到的消息"+readStr);
 
                string sendStr = "我是服务端";
                byte[] sendBuffer = Encoding.UTF8.GetBytes(sendStr);
                clientSocket.Send(sendBuffer);
            }
        }
    }
}
~~~

**客户端**

~~~cs
using System.Collections;
using System.Collections.Generic;
using System.Net.Sockets;
using System.Text;
using UnityEngine;
using UnityEngine.UI;

public class Client : MonoBehaviour
{
    
    public InputField InputField;
    public Text text;
    private Socket socket;
    
    public void Connect()
    {
        //连接
        socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        socket.Connect("127.0.0.1", 8888);
    }

    public void Send()
    {
        //发送
        string  sendStr = InputField.text;
        byte[] sendBytes = Encoding.UTF8.GetBytes(sendStr);
        socket.Send(sendBytes);
        //接收
        byte[] readBuffer = new byte[1024];
        int count = socket.Receive(readBuffer);
        string receiveStr = Encoding.UTF8.GetString(readBuffer, 0, count);
        text.text = receiveStr;

    }
     
}

~~~

这个例子使用的是TCP协议，是有连接、保证数据按序到达、可靠的传输协议，当有数据包丢失时会使用重传机制保证可靠性（虽然性能会降低），接收的字节流需要一个缓冲区来接收，Receive方法有数据就返回（至少1字节），没有数据则会一直阻塞，不会等待所有数据包接收后才返回。

虽然TCP是可靠的传输协议，但是**Receive方法属于应用层的方法，TCP是传输层的协议**，所以虽然传输层保证数据可靠，但是应用层方法不当也会导致接收的数据有问题。在上述代码中，当客户端发送一个hello world后，服务端Receive（）方法就有可能接收成hell，导致一次接收数据不完整而造成潜在问题。

由于Accept()`、`Connect()`、`Receive()`、`Send()方法都会导致线程阻塞，调用方法会一直等待操作完成，因为没有处理连接中断情况造成卡死的问题，所以需要使用异步的方式来进行连接。

### 异步通信

**服务端**

~~~cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Sockets;
using System.Text;
using System.Threading.Tasks;
#nullable disable
namespace UnityServer
{
    internal class ClientState
    {
        /// <summary>
        /// 客户端socket
        /// </summary>
        public Socket socket;
        /// <summary>
        /// 接收缓冲区
        /// </summary>
        public byte[] readBuff=new byte[1024];
    }
}


using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading.Tasks;
#nullable disable
namespace UnityServer
{
    internal class Server
    {
        static string receiveStr = "";
        /// <summary>
        /// 服务端socke
        /// </summary>
        static Socket serverSocket;
        /// <summary>
        /// 客户端socket以及客户端信息的字典
        /// </summary>
        static Dictionary<Socket, ClientState> clients = new Dictionary<Socket, ClientState>();
        public static void Main()
        {
            //定义socket
            serverSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
            //绑定ip和端口
            IPAddress ip = IPAddress.Parse("127.0.0.1");
            IPEndPoint iPEndPoint = new IPEndPoint(ip, 8888);
            serverSocket.Bind(iPEndPoint);
            //监听
            serverSocket.Listen(0);
            Console.WriteLine("服务端启动成功");
            serverSocket.BeginAccept(AcceptCallback, serverSocket);
            Console.ReadLine();
        }
        /// <summary>
        /// 应答回调
        /// </summary>
        /// <param name="ar"></param>
        private static void AcceptCallback(IAsyncResult ar)
        {
            try
            {
                Socket serverSocket = ar.AsyncState as Socket;
                Socket clientSocket = serverSocket.EndAccept(ar);
                //创建客户端的ClientState以及添加字典
                ClientState state = new ClientState();
                state.socket = clientSocket;
                clients.Add(clientSocket, state);
 
                Console.WriteLine("有一个客户端Accept");
                clientSocket.BeginReceive(state.readBuff, 0, 1024, 0, ReceiveCallback, state);
                serverSocket.BeginAccept(AcceptCallback, serverSocket);
            }
            catch (SocketException ex)
            {
                Console.WriteLine("应答失败" + ex.ToString());
            }
        }
        /// <summary>
        /// 接收回调
        /// </summary>
        /// <param name="ar"></param>
        private static void ReceiveCallback(IAsyncResult ar)
        {
            try
            {
                ClientState state = ar.AsyncState as ClientState;
                Socket clientSocket = state.socket;
                int count = clientSocket.EndReceive(ar);
                //客户端的关闭
                if (count == 0)
                {
                    clientSocket.Close();
                    clients.Remove(clientSocket);
                    Console.WriteLine("客户端断开连接");
                    return;
                }
                receiveStr = Encoding.UTF8.GetString(state.readBuff, 0, count);
                Console.WriteLine(receiveStr);
 
                foreach (ClientState s in clients.Values)
                {
                    s.socket.BeginSend(Encoding.UTF8.GetBytes(receiveStr), 0, receiveStr.Length, 0, SendCallback, clientSocket);
                }
                clientSocket.BeginReceive(state.readBuff, 0, 1024, 0, ReceiveCallback, state);
            }
            catch (SocketException ex)
            {
                Console.WriteLine("接收失败" + ex.ToString());
            }
        }
        /// <summary>
        /// 发送回调
        /// </summary>
        /// <param name="ar"></param>
        private static void SendCallback(IAsyncResult ar)
        {
            try
            {
                Socket clientSocket = ar.AsyncState as Socket;
                clientSocket.EndSend(ar);
            }
            catch (SocketException ex)
            {
                Console.WriteLine("发送失败" + ex.ToString());
            }
        }
        
    }
}
~~~



**客户端**

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using System.Net.Sockets;
using UnityEngine.UI;
using System.Text;
using System;
 
public class Client : MonoBehaviour
{
    Socket socket;
    public InputField InputField;
    public Text text;
    /// <summary>
    /// 接收缓冲区
    /// </summary>
    byte[] readBuff=new byte[1024];
    string receiveStr = "";
    /// <summary>
    /// 连接
    /// </summary>
    public void Connect()
    {
        socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        socket.BeginConnect("127.0.0.1", 8888, ConnectCallback, socket);
    }
    /// <summary>
    /// 连接回调
    /// </summary>
    /// <param name="ar"></param>
    private void ConnectCallback(IAsyncResult ar)
    {
        try
        {
            Socket socket = (Socket)ar.AsyncState;
            socket.EndConnect(ar);
            Debug.Log("连接成功");
            socket.BeginReceive(readBuff, 0, 1024, 0, ReceiveCallback, socket);
        }
        catch (SocketException ex)
        {
            Debug.Log("客户端连接失败" + ex.ToString());
        }
    }
    /// <summary>
    /// 接收回调
    /// </summary>
    /// <param name="ar"></param>
    private void ReceiveCallback(IAsyncResult ar)
    {
        try
        {
            Socket socket = ar.AsyncState as Socket;
            int count = socket.EndReceive(ar);
            string s=Encoding.UTF8.GetString(readBuff,0,count);
            receiveStr = s + "\n" + receiveStr;
            
            socket.BeginReceive(readBuff, 0, 1024, 0, ReceiveCallback, socket);
        }
        catch (SocketException ex)
        {
            Debug.Log("客户端接收失败" + ex.ToString());
        }
    }
 
    /// <summary>
    /// 发送
    /// </summary>
    public void Send()
    {
        //发送
        string sendStr = InputField.text;
        byte[] sendBytes = Encoding.UTF8.GetBytes(sendStr);
        socket.BeginSend(sendBytes, 0, sendBytes.Length, 0, SendCallback, socket);
 
        
    }
    /// <summary>
    /// 发送回调
    /// </summary>
    /// <param name="ar"></param>
    private void SendCallback(IAsyncResult ar)
    {
        try
        {
            Socket socket =ar.AsyncState as Socket;
            int count=socket.EndSend(ar);
            Debug.Log("发送成功" + count);
        }
        catch (SocketException ex)
        {
            Debug.Log("发送失败" + ex.ToString());
        }
    }
    private void Update()
    {
        text.text = receiveStr;
    }
}
~~~

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/deepseek_mermaid_20250715_f7f595.png" alt="deepseek_mermaid_20250715_f7f595" style="zoom: 25%;" />

阻塞（Blocking）：在阻塞操作中，如果数据还没有准备好（例如，等待数据从磁盘读取或从网络接收），则调用者（通常是一个线程或进程）会被阻塞，直到数据准备好为止。在此期间，调用者无法执行其他任务，只能等待I/O操作完成。阻塞I/O操作的典型例子是普通的文件读写。
非阻塞（Non-blocking）：在非阻塞操作中，如果数据还没有准备好，调用者不会被阻塞，而是立即返回一个错误码（例如，表示资源不可用）。调用者可以继续执行其他任务，然后在适当的时间点再次尝试I/O操作。非阻塞I/O操作的典型例子是使用select，poll或epoll等I/O多路复用技术处理的网络通信。

使用异步的方式不会造成线程阻塞，以客户端接收为例，启动BeginReceive之后，在操作系统接收到消息后，需要先EndReceive接收完消息，然后才能进行数据的处理，如果在EndReceive之前进行操作可能出现异常等问题。当然，采用异步编程可以在多个线程中并发处理，

**原因：**Begin启动异步操作，End完成它。在Begin之后，操作系统开始接收数据，但数据存储在内核缓冲区中。当数据完全到达后，回调被触发，此时调用EndReceive将数据从内核缓冲区复制到用户提供的缓冲区（如readBuff）。只有EndReceive完成后，用户才能确定有多少数据被正确接收，从而保证数据的完整性。

IAsyncResult ar 是 .NET 异步编程模型中的核心接口，用于跟踪异步操作的状态和结果。在 Socket 编程中，每个回调函数都必须通过 ar 参数获取异步操作的上下文信息和最终结果。

Socket socket = ar.AsyncState as Socket;对于同一个连接，每次获取的socket是一样的，不论是哪条消息。

### I/O同步多路复用Select

[你管这破玩意叫 IO 多路复用？-CSDN博客](https://bcysz.blog.csdn.net/article/details/121646426?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2~default~CTRLIST~PaidSort-1-121646426-blog-123452472.235^v43^pc_blog_bottom_relevance_base2&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2~default~CTRLIST~PaidSort-1-121646426-blog-123452472.235^v43^pc_blog_bottom_relevance_base2&utm_relevant_index=1)

使用上述的异步编程也有许多缺点。每个异步操作（接收/发送）完成时，回调会在I/O线程池线程中执行。如果有多个客户端则会产生大量线程池任务，在高并发下可能耗尽线程池。**也就是说，执行Begin方法不会占用太多资源，但在执行回调函数时，每一个回调函数就会占用一个线程资源。**

所以我们需要使用单线程轮询所有连接，使用Select实现。

~~~cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Net.Sockets;
using System.Net;
/// <summary>
/// 服务器总管理
/// </summary>
public class NetManager
{
    public static NetManager Instance;
    private Socket listenedSocket;
    //客户端Socket对象以及信息
    public Dictionary<Socket, ClientObject> clientObjectsDict = new Dictionary<Socket, ClientObject>();
    private List<Socket> checkReadSocketsList = new List<Socket>();
    /// <summary>
    /// 开始连接并监听消息
    /// </summary>
    /// <param name="listenedPort">端口号</param>
    public void StartServer(int listenedPort)
    {
        listenedSocket = new Socket(AddressFamily.InterNetwork,SocketType.Stream,ProtocolType.Tcp);
        IPAddress iPAddress = IPAddress.Parse("127.0.0.1");
        IPEndPoint iPEndPoint = new IPEndPoint(iPAddress, listenedPort);
        listenedSocket.Bind(iPEndPoint);
        listenedSocket.Listen(0);
        DbManager.Instance = new DbManager();
        DbManager.Instance.Connect("wow", "127.0.0.1", 3306, "root", "");
        Console.WriteLine("Wow服务器启动成功");
        //异步方法
        //listenedSocket.BeginAccept(AcceptCallback, listenedSocket);
        while (true)
        {
            ResetCheckReadList();
            //多路复用
            Socket.Select(checkReadSocketsList, null,null,1000);
            for (int i = 0; i < checkReadSocketsList.Count; i++)
            {
                Socket socket= checkReadSocketsList[i];
                if (socket==listenedSocket)
                {
                    HandleListenedSocket();
                }
                else
                {
                    HandleClientSocket(socket);
                }
            }
            CheckPingTime();
        }        
    }

    /// <summary>
    /// 更新可读Socket检测列表
    /// </summary>
    private void ResetCheckReadList()
    {
        checkReadSocketsList.Clear();
        checkReadSocketsList.Add(listenedSocket);
        foreach (ClientObject co in clientObjectsDict.Values)
        {
            checkReadSocketsList.Add(co.socket);
        }
    }
    /// <summary>
    /// 处理服务器负责监听的套接字
    /// </summary>
    private void HandleListenedSocket()
    {
        try
        {
            Socket clientSocket = listenedSocket.Accept();
            ClientObject co = new ClientObject() { socket = clientSocket,lastPingTime=GetTimeStamp() };
            clientObjectsDict.Add(clientSocket, co);
        }
        catch (SocketException se)
        {
            Console.WriteLine("客户端连接失败：" + se);
        }
    }
    /// <summary>
    /// 处理客户端套接字
    /// </summary>
    private void HandleClientSocket(Socket socket)
    {
        ClientObject co = clientObjectsDict[socket];
        //缓冲区满了，解析消息
        if (co.bo.remainLength<=0)
        {
            HandleReceiveData(co);
        }
        if (co.bo.remainLength <= 0)
        {
            Console.WriteLine("接收消息失败，协议解析不成功或单条协议超过缓冲区长度");
            CloseClientSocket(co);
            return;
        }
        int length = 0;
        try
        {         
            length = co.socket.Receive(co.bo.bytes,co.bo.writeIndex,co.bo.remainLength,SocketFlags.None);
        }
        catch (SocketException se)
        {
            Console.WriteLine("接收信息失败：" + se);
            CloseClientSocket(co);
            return;
        }
        //客户端正常关闭
        if (length<=0)
        {
            Console.WriteLine("客户端断开连接");
            CloseClientSocket(co);
        }
        co.bo.writeIndex += length;
        HandleReceiveData(co);
        co.bo.CheckAndMoveBytes();
    }
    /// <summary>
    /// 关闭客户端
    /// </summary>
    /// <param name="co"></param>
    private void CloseClientSocket(ClientObject co)
    {
        Console.WriteLine("客户端:" + co.socket.RemoteEndPoint + "关闭");
        PlayerManager.Instance.RemovePlayer(co.player.id);
        co.socket.Close();
        clientObjectsDict.Remove(co.socket);
    }

    public void Send(ClientObject co,PTBase pt)
    {
        byte[] ptBytes = PT.EncodeName(pt).Concat(PT.EncodeBody(pt)).ToArray();
        Int16 length = (Int16)ptBytes.Length;
        byte[] lengthBytes = BitConverter.GetBytes(length);
        if (!BitConverter.IsLittleEndian)
        {
            lengthBytes.Reverse();
        }
        byte[] sendBytes = lengthBytes.Concat(ptBytes).ToArray();
        ByteObject bo = new ByteObject(sendBytes);
        co.socket.Send(bo.bytes,0,bo.bytes.Length,SocketFlags.None);
    }

    private void HandleReceiveData(ClientObject co)
    {
        if (co.bo.dataLength <= 2)
        {
            return;
        }
        Int16 bodyLength = (Int16)(co.bo.bytes[co.bo.readIndex] | co.bo.bytes[co.bo.readIndex + 1] << 8);
        if (co.bo.dataLength < bodyLength + 2)
        {
            return;
        }
        co.bo.readIndex += 2;
        //解析协议名
        int nameCount = 0;
        string protoName = PT.DecodeName(co.bo.bytes, co.bo.readIndex, out nameCount);
        if (protoName == "")
        {
            Console.WriteLine("协议解析失败");
            return;
        }
        co.bo.readIndex += nameCount;
        //解析协议体
        int bodyCount = bodyLength - nameCount;
        PTBase ptBase = PT.DecodeBody(protoName, co.bo.bytes, co.bo.readIndex, bodyCount);
        co.bo.readIndex += bodyCount;
        co.bo.CheckAndMoveBytes();
        PTManager.Instance.SendPTEvent(protoName, new MsgPT() { clientObject = co, pt = ptBase }) ;
        HandleReceiveData(co);
    }
}

~~~



### 分包、粘包问题

分包是指一个数据包被分成多个小包发送，而粘包则是多个小包被合并成一个大包接收。这在TCP通信中常见，因为TCP是流式传输，没有消息边界。

为了解决这个问题，有固定长度法（浪费带宽、灵活性差）、分隔符法（只适合文本通信，如HTTP）、长度前缀法（需要缓冲区管理）。以下说明长度前缀法的实现方式。

**长度前缀法**：利用消息头存储消息体的长度，发送端先发送2字节长度字段，再发送实际数据；接收端先读取2字节获取消息长度，根据长度读取后续字节作为完整消息。

使用PT类定义协议的封装与解析，在发送（Send）和解析消息（HandleReceiveData）的时候使用其中的方法就可以实现。

~~~cs
using System.Collections;
using System.Collections.Generic;
using System.Text;
using System;
using System.Linq;
using System.Web.Script.Serialization;
//*****************************************
//功能说明：协议封装与解析工具类
//***************************************** 
public class PT
{
    //json解码编码器
    private static JavaScriptSerializer jss = new JavaScriptSerializer();

    /// <summary>
    /// 编码协议体
    /// </summary>
    /// <param name="ptBase"></param>
    /// <returns></returns>
    public static byte[] EncodeBody(PTBase ptBase)
    {
        return Encoding.UTF8.GetBytes(jss.Serialize(ptBase));
    }
    /// <summary>
    /// 解码协议体
    /// </summary>
    /// <param name="protoName"></param>
    /// <param name="bytes"></param>
    /// <param name="startIndex"></param>
    /// <param name="count"></param>
    /// <returns></returns>
    public static PTBase DecodeBody(string protoName,byte[] bytes,
        int startIndex,int count)
    {
        return (PTBase)jss.Deserialize(Encoding.UTF8.GetString(bytes, startIndex, count),
            Type.GetType(protoName));
    }
    /// <summary>
    /// 编码协议名
    /// </summary>
    /// <param name="ptBase"></param>
    /// <returns></returns>
    public static byte[] EncodeName(PTBase ptBase)
    {
        byte[] nameBytes = Encoding.UTF8.GetBytes(ptBase.protoName);
        Int16 length = (Int16)nameBytes.Length;
        byte[] lengthBytes = BitConverter.GetBytes(length);
        if (!BitConverter.IsLittleEndian)
        {
            lengthBytes.Reverse();
        }
        byte[] bytes = lengthBytes.Concat(nameBytes).ToArray();
        return bytes;
    }
    /// <summary>
    /// 解码协议名
    /// </summary>
    /// <param name="bytes"></param>
    /// <param name="startIndex"></param>
    /// <returns></returns>
    public static string DecodeName(byte[] bytes,int startIndex,out int count)
    {
        count = 0;
        if (bytes.Length<2+startIndex)
        {
            return "";
        }
        Int16 length = (Int16)(bytes[startIndex] | bytes[startIndex + 1] << 8);  
        if (startIndex+2+length>bytes.Length)
        {
            return "";
        }
        count = length+2;
        return Encoding.UTF8.GetString(bytes,startIndex+2,length);
    }
}

~~~



### 大端小端

大小端是指多字节数据在内存中存储的字节顺序，核心区别在于高位字节和低位字节的排列方式。

| 类型 | 字节排列     | 示例（0x12345678） |
| ---- | ------------ | ------------------ |
| 大端 | 高位字节在前 | 12 34 56 78        |
| 小端 | 底位字节在前 | 78 56 34 12        |

例如上述提到的长度前缀法消息头存储的消息长度就需要考虑大小端的使用模式。

### 心跳机制

虽然TCP是可靠连接，但它底层主要保证数据包的可靠传输，并不能实时感知应用层的连接状态。一个Socket连接可能表面上还是“连接中”，但实际上物理链路早已断开（比如网线被拔、Wi-Fi突然断开、客户端程序崩溃）。心跳是应用层检测连接健康度的最有效手段。

心跳机制的原理非常简单，通常包含以下两个部分：

1. **发送心跳包**：客户端和服务器（通常是客户端主动发送，服务器回应）会以固定的时间间隔（例如每5秒、每30秒）向对方发送一个非常小的、特定的数据包。
2. **接收与回应**：
   - 服务器收到客户端的心跳包后，会立即回复一个应答包（ACK），表示“我收到了，我还活着”。
   - 同样，客户端收到服务器主动发来的（或是对它发送的）心跳应答包，也认为连接正常。

## 基于TCP的Socket网络连接

### 基本流程

相比于TCP，UDP的基本通信流程十分简单，并且客户端和服务端相同。首先需要创建自己的Socket对象并绑定本地地址，然后就可以使用接收和发送方法来进行通信。

![image-20251024145730173](https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20251024145730173.png)

### 分包、粘包问题

UDP相比于TCP，是无连接，不可靠的传输协议。TCP在发送与接收时，都会使用一个缓冲区来控制数据，保持数据的流式传输，因为数据是流式的，就会产生数据的分割和粘连。

但UDP不同，UDP的每一个数据包就是最小单位，要么整个包到达，要么整个包丢弃，因此不存在所谓的粘包现象。在发送数据时，应该保证数据包大小在合适的范围之内，因此需要定义最大传输单元（MTU）。

### 同步通信

~~~cs

using System.Net;
using System.Net.Sockets;
using System.Text;

public class UDPTest
{
    public static void Main(string[] args)
    {
        //创建套接字
        Socket socket = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp);
        //绑定地址和端口
        IPEndPoint endPoint = new IPEndPoint(IPAddress.Parse("127.0.0.1"), 5000);
        socket.Bind(endPoint);
        //发送消息
        IPEndPoint sendEndPoint = new IPEndPoint(IPAddress.Parse("127.0.0.1"), 5001);
        socket.SendTo(Encoding.UTF8.GetBytes("Hello world"), sendEndPoint);
        //接收消息
        byte[] buffer = new byte[1024];
        EndPoint remoteEndPoint = new IPEndPoint(IPAddress.Any, 0);
        int length = socket.ReceiveFrom(buffer, ref remoteEndPoint);
        Console.WriteLine(Encoding.UTF8.GetString(buffer, 0, length));  
        //关闭套接字的发送和接收功能
        socket.Shutdown(SocketShutdown.Both);
        //关闭套接字
        socket.Close();
    }
}
~~~

### 异步通信


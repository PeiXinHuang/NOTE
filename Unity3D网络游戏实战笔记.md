# Unity3D网络游戏实战笔记

## 一、网络游戏的开端

> - Socket —— 网络上通讯的两个程序的端点，Socket包含了5种信息（连接的协议，本地IP，本地端口，远程IP,远程端口）
>
> - Socket通信流程
>
>   - 客户端创建Socket对象，绑定IP和Port
>   - 服务端开启监听 (Listener)
>   - 客户端连接服务端（Connect）
>   - 服务端接受连接（Accept）连接完成
>   - 客户端和服务端开始发送接受数据（Send/Receive）
>   - 某一方关闭连接（Close）
>
> - 简单的客户端服务端通讯程序（Echo程序）
>
>  ```c#
> // 客户端
> // 连接函数
> public void Connetion()
> {
>     //创建Socket
>     socket = new Socket(AddressFamily.InterNetwork,
>                         SocketType.Stream, ProtocolType.Tcp);
>     //连接远程服务器
>     socket.Connect("127.0.0.1", 8888);
> }
> 
> //发送数据
> public void Send()
> {
>     // 发送内容
>     string sendStr = "hello,this is client";
>     byte[] sendBytes = System.Text.Encoding.Default.GetBytes(sendStr);
>     // 发送
>     socket.Send(sendBytes);
>     // 接收内容
>     byte[] readBuff = new byte[1024]; 
>     int count = socket.Receive(readBuff);
>     // 接收数据
>     string recvStr = System.Text.Encoding.UTF8.GetString(readBuff, 0, count);
>     // 关闭连接
>     socket.Close();
> }
>  ```
>
> ``` c#
> // 服务端程序
> public static void Main (string[] args)
> {
>     // 创建监听Socket
>     Socket listenfd = new Socket(AddressFamily.InterNetwork,
>                                  SocketType.Stream, ProtocolType.Tcp);
>     // 监听Socket绑定地址和端口
>     IPAddress ipAdr = IPAddress.Parse("127.0.0.1");
>     IPEndPoint ipEp = new IPEndPoint(ipAdr, 8888);
>     listenfd.Bind(ipEp);
> 
>     // 监听Socket开始监听
>     listenfd.Listen(0);
> 
>     Console.WriteLine("[服务器]启动成功");
> 
>     while (true) {
> 
>         // 监听Socket被连接,返回一个连接Socket用来处理客户端数据
>         Socket connfd = listenfd.Accept();
> 
>         byte[] readBuff = new byte[1024];
> 
>         // 连接Socket接受客户端数据
>         int count = connfd.Receive (readBuff);
> 
>         string readStr = System.Text.Encoding.UTF8.GetString (readBuff, 0, count);
>         Console.WriteLine ("[服务器接收]" + readStr);
> 
> 
>         string sendStr = System.DateTime.Now.ToString();
>         byte[] sendBytes = System.Text.Encoding.Default.GetBytes (sendStr);
> 
>         // 连接Socket发送数据到客户端
>         connfd.Send (sendBytes);
>     }
> }
> ```
> ```C#
> 以上代码都使用了阻塞API(Connect Send Receive等)，存在程序容易卡住，照成一直等待的缺点
> ```


----

## 二、异步和多路复用（解决程序卡住的三种方法）

> 1. 异步  —— 执行异步函数，程序不需要继续等待，而是继续执行下去，如果满足条件就调用回调函数，提高执行的效率
> 2. 状态检测Poll —— 使用Poll方法检测Socket的状态，如果Socket可读，那么调用 Socket Receive，否则继续循环（这样就防止了一直Receive等待接收数据而照成的卡住现象），同理，如果socket可写，才调用Send
> 3. 多路复用Select —— 服务端存在多个连接Socket，使用Select函数，当有一个或者多个Socket客户的时候，返回这些Socket，调用Receive，这样就没有了Receive照成的阻塞现象，同时还降低了CPU占用率
>
> 客户端程序
>
> ```c#
> 	//定义套接字
> 	Socket socket;
> 
> 	//接收缓冲区
> 	byte[] readBuff = new byte[1024]; 
> 	string recvStr = "";
> 
> 	//checkRead列表
> 	List<Socket> checkRead = new List<Socket>();
> 
> 	//点击连接按钮
> 	public void Connetion()
> 	{
> 		//Socket
> 		socket = new Socket(AddressFamily.InterNetwork,
> 			SocketType.Stream, ProtocolType.Tcp);
> 		//Connect
> 		socket.BeginConnect("127.0.0.1", 8888, ConnectCallback, socket);
> 	}
> 
> 	//Connect回调
> 	public void ConnectCallback(IAsyncResult ar){
> 		try{
> 			Socket socket = (Socket) ar.AsyncState;
> 			socket.EndConnect(ar);
> 			Debug.Log("Socket Connect Succ ");
> 		}
> 		catch (SocketException ex){
> 			Debug.Log("Socket Connect fail" + ex.ToString());
> 		}
> 	}
> 
> 	// Send Poll
> 	public void Send()
> 	{
> 		string sendStr = InputFeld.text;
> 		byte[] sendBytes = System.Text.Encoding.Default.GetBytes(sendStr);
> 		if(socket == null) return;
>         if(socket.Poll(0,SelectMode.SelectWrite))
>             socket.Send(sendBytes); 
> 	}	
> 
> 	public void Update(){
> 		if(socket == null) {
> 			return;
> 		}
> 		//填充checkRead列表
> 		checkRead.Clear();
> 		checkRead.Add(socket); //这里是客户端，只有一个读取Socket
>         
> 		// Receive Select 
> 		Socket.Select(checkRead, null, null, 1000);
> 		foreach (Socket s in checkRead){
> 			readBuff = new byte[1024];
> 			int count = socket.Receive(readBuff);
> 			recvStr = System.Text.Encoding.Default.GetString(readBuff, 0, count);
> 		}
> 	}
> ```
>
> 服务端程序(使用Poll)
>
> ```c#
> // 连接客户端管理类
> class ClientState
> {
> 	public Socke socket;  //连接客户端 Socket
> 	public byte[] readBuff = new byte[1024];  //连接客户端缓存数据
> }
> 
> class MainClass
> {
> 	//监听Socket
> 	static Socket listenfd;
> 	//客户端Socket及状态信息
> 	static Dictionary<Socket, ClientState> clients = new Dictionary<Socket, ClientState>();
> 
> 	public static void Main (string[] args)
> 	{
> 
> 		listenfd = new Socket(AddressFamily.InterNetwork,
> 			SocketType.Stream, ProtocolType.Tcp);
> 		IPAddress ipAdr = IPAddress.Parse("127.0.0.1");
> 		IPEndPoint ipEp = new IPEndPoint(ipAdr, 8888);
> 		listenfd.Bind(ipEp);
> 		listenfd.Listen(0);
> 		Console.WriteLine("[服务器]启动成功");
>         
> 		while(true){
> 			//检查listenfd
> 			if(listenfd.Poll(0, SelectMode.SelectRead)){
> 				ReadListenfd(listenfd);
> 			}
> 			//检查clientfd
> 			foreach (ClientState s in clients.Values){
> 				Socket clientfd = s.socket;
> 				if(clientfd.Poll(0, SelectMode.SelectRead)){
> 					if(!ReadClientfd(clientfd)){
> 						break;
> 					}
> 				}
> 			}
> 			//防止cpu占用过高
> 			System.Threading.Thread.Sleep(1);
> 		}
> 
> 	}
> 
> 	//读取Listenfd
> 	public static void ReadListenfd(Socket listenfd){
> 		Console.WriteLine("Accept");
> 		Socket clientfd = listenfd.Accept();
> 		ClientState state = new ClientState();
> 		state.socket = clientfd;
> 		clients.Add(clientfd, state);
> 	}
> 
> 	//读取Clientfd
> 	public static bool ReadClientfd(Socket clientfd){
> 		ClientState state = clients[clientfd];
> 		int count = clientfd.Receive(state.readBuff);
> 		//客户端关闭
> 		if(count == 0){
> 			clientfd.Close();
> 			clients.Remove(clientfd);
> 			Console.WriteLine("Socket Close");
> 			return false;
> 		}
> 		//广播
> 		string recvStr = System.Text.Encoding.Default.GetString(state.readBuff, 0, count);
> 		Console.WriteLine("Receive " + recvStr);
> 		string sendStr = clientfd.RemoteEndPoint.ToString() + ":" + recvStr;
> 		byte[] sendBytes = System.Text.Encoding.Default.GetBytes(sendStr);
> 		foreach (ClientState cs in clients.Values){
> 			cs.socket.Send(sendBytes);
> 		}
> 		return true;
> 	}
> }
> ```





---

## 三、大乱斗游戏(网络模块)

> - 通信协议 通信双方达成的数据传送的一种约定 如: 消息名|参数1 参数2 参数3
> - 内容知识点(见后面通用客户端 服务端模块)



---

## 四、正确收发数据流

> - 粘包和半包现象
>   - 粘包:发送端快速发送多条数据,接收到没有及时调用Receive,之后接收数据的时候同时读到多条数据
>   - 半包:系统只接受到一条数据的一半部分,之后才接受到另外一半
> - 解决粘包和半包方法
>   - 长度信息法: 在美国数据包前面加上长度信息,每次接收读取该长度的数据,如果数据长度小于要接收的数据长度,等待下一次数据接收
>   - 固定长度法: 规定每条数据的长度相等,用"."表示填充字符
>   - 结束符号法:使用特定符号来分隔信息里的每一条数据
> - 大端小端问题
>   - 对于长度信息的数据,大端的计算方法是第一个数据×256+第二个数据,小端的计算方法是第二个数据×256+第一个数据,在不同的设备上,大小端模式的选用都可能不同,于是会照成数据读取错误
> - 大小端问题解决方法
>   - 判断是否为大端编码,如果是,调用Reverse()方法,将大端编码转换为小端编码
>   - 手动还原数值,由于系统大小端采用不同的编码格式,对大端编码改为小端编码的读取格式:((readBuff[1]<<8)|readBuff[0])
> - 完整发送数据



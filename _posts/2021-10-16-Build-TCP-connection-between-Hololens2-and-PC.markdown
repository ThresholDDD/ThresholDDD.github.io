---
layout: post
title: "Build TCP connection between Hololens 2 and PC"
data: 2021-10-16
---

# Abstract
It is not a pleasant experience to build the TCP connection between Hololens and PC especially when you are not familiar with both Hololens development and communication. I was struggling with it over one week and finally make it to work. Following I will show how I tackle the problem. I mainly referenced a [blog](https://foxypanda.me/tcp-client-in-a-uwp-unity-app-on-hololens/) , a [github code](https://gist.github.com/danielbierwirth/0636650b005834204cb19ef5ae6ccedb) and a [stackoverflow question](https://stackoverflow.com/questions/51411832/hololens-tcp-sockets-python-client-to-hololens-server). I think those three will be a good point to start. For my experince, since I am not an expert, my basic thinking is to try each sample code and see if it works. I have to say I still do not fully understand this. Therefore, in this blog, I will mainly show the problems of the sample code, some sources and the final version code I used.

# History of solving problems 

1. The first problem I met is I don't know how to start. 
	* **Recommendation:** I recommend to start with [code](https://stackoverflow.com/questions/51411832/hololens-tcp-sockets-python-client-to-hololens-server) in stackoverflow. In this code, Hololens 2 acts as server and python code on PC is a client. Notice: The server app is developed in unity. Since unity c# is different from UWP c#, therefore, they use keyword like #if !UNITY_EDITOR.
	* **Reason:** It works. It can give you some confidence. In addition, it gives you a hint about pipeline in server. You can check the each class or event in windows document.
	* **Following Problem:** Since my goal is to create app on PC to communicate with Hololens 2. Therefore, I create a [unity client](https://gist.github.com/danielbierwirth/0636650b005834204cb19ef5ae6ccedb). After test, I found it can work only one time. For the second time, an error will pop up "SocketException: An existing connection was forcibly closed by the remote host" in unity and "A device disconnection with the id 20000000068 has been reported but no device with that id was connect" in Hololens 2. The python server only execute once and then connection will be closed. That is why whenever I tried python server it can work.
	* **Why?:** Actually I did not find out why otherwise I will stop here :). I referenced [windows official sample](https://github.com/microsoft/Windows-universal-samples/tree/main/Samples/StreamSocket) to modify bit of above code, but still not work. I have one guess -> for continuously reading the data, there is a while lop in async void function, but actually while loop is run in main thread, it may cause some problem.
	
2. Then I directly try the sample code found in [github](https://gist.github.com/danielbierwirth/0636650b005834204cb19ef5ae6ccedb). I tried it on two unity instances, one for server another for client. That works good. Then when I applied to Hololens 2 (server), nothing happened. 
	* **WHY:** Since the github code is all unity c#, it is different from UWP c#. For instance, in unity c#, you cannot use async function but need to create new thread.
	
3. I modified code in [blog](https://foxypanda.me/tcp-client-in-a-uwp-unity-app-on-hololens/) (This blog is definitly worthy reading, it clearly explained the problem and solution. The Code is very clear and makes me understand more about multithread.) and make hololens 2 to be client. Server code is still from [github](https://gist.github.com/danielbierwirth/0636650b005834204cb19ef5ae6ccedb). But I found the hololens 2 could not connect to PC.
	* **WHY:** Windows firewall plays a role in this case. Because previous experience tells me there is no problem in connection when server is hololens. However this time, it cannot connect at all. So I guess it is because firewall.
	* **Solution:** Solution is simple, just close windows firewall.
	
	
# Code 

**server**

{% highlight ruby linenos%}
using System;
using System.Collections;
using System.Collections.Generic;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading;
using UnityEngine;
using TMPro;
using System.Text.RegularExpressions;
public class server:MonoBehaviour
{

	/// <summary> 	
	/// TCPListener to listen for incomming TCP connection 	
	/// requests. 	
	/// </summary> 	
	private TcpListener tcpListener;
	/// <summary> 
	/// Background thread for TcpServer workload. 	
	/// </summary> 	
	private Thread tcpListenerThread;
	/// <summary> 	
	/// Create handle to connected tcp client. 	
	/// </summary> 	
	private TcpClient connectedTcpClient;

	public string mes;


		// Use this for initialization
		void Start()
	{
		// Start TcpServer background thread 		
		tcpListenerThread = new Thread(new ThreadStart(ListenForIncommingRequests));
		tcpListenerThread.IsBackground = true;
		tcpListenerThread.Start();
	}

	// Update is called once per frame
	void Update()
	{
		if (Input.GetKeyDown(KeyCode.Space))
		{
			SendMessage();
		}

	}

	/// <summary> 	
	/// Runs in background TcpServerThread; Handles incomming TcpClient requests 	
	/// </summary> 	
	private void ListenForIncommingRequests()
	{
		try
		{	
			//tcpListener = new TcpListener(IPAddress.Any, 9090);
			tcpListener = new TcpListener(IPAddress.Parse("your computer's IP"), 9090);
			tcpListener.Start();
			Debug.Log("Server is listening");
			Byte[] bytes = new Byte[1024];
			while (true)
			{
				using (connectedTcpClient = tcpListener.AcceptTcpClient())
				{
					// Get a stream object for reading 					
					using (NetworkStream stream = connectedTcpClient.GetStream())
					{
						int length;
						// Read incomming stream into byte arrary. 						
						while ((length = stream.Read(bytes, 0, bytes.Length)) != 0)
						{
							var incommingData = new byte[length];
							Array.Copy(bytes, 0, incommingData, 0, length);
							// Convert byte array to string message. 							
							string clientMessage = Encoding.ASCII.GetString(incommingData);
							mes = clientMessage;
							Debug.Log("client message received as: " + clientMessage);

						}
					}
				}
			}
		}
		catch (SocketException socketException)
		{
			Debug.Log("SocketException " + socketException.ToString());
		}
	}
	/// <summary> 	
	/// Send message to client using socket connection. 	
	/// </summary> 	
	private void SendMessage()
	{
		if (connectedTcpClient == null)
		{
			return;
		}

		try
		{
			// Get a stream object for writing. 			
			NetworkStream stream = connectedTcpClient.GetStream();
			if (stream.CanWrite)
			{
				string serverMessage = mes + "\n";
				// Convert string message to byte array.                 
				byte[] serverMessageAsByteArray = Encoding.ASCII.GetBytes(serverMessage);
				// Write byte array to socketConnection stream.               
				stream.Write(serverMessageAsByteArray, 0, serverMessageAsByteArray.Length);
				Debug.Log("Server sent his message - should be received by client");
			}
		}
		catch (SocketException socketException)
		{
			Debug.Log("Socket exception: " + socketException);
		}
	}
}

{% endhighlight %}


**Client**

{%highlight ruby linenos%}
using System;
using System.Collections.Generic;
using System.IO;
using System.Text;
using System.Threading;
using UnityEngine;
using TMPro;
#if !UNITY_EDITOR
using System.Threading.Tasks;
#endif

public class clientFoxy : MonoBehaviour
{

    public GameObject text;
    private string host= "host IP";
    private string port="9090";
    public StateManager stateManager;
#if !UNITY_EDITOR
    private bool _useUWP = true;
    private Windows.Networking.Sockets.StreamSocket socket;
    private Task exchangeTask;
#endif

#if UNITY_EDITOR
    private bool _useUWP = false;
    System.Net.Sockets.TcpClient client;
    System.Net.Sockets.NetworkStream stream;
    private Thread exchangeThread;
#endif

    private Byte[] bytes = new Byte[256];
    private StreamWriter writer;
    private StreamReader reader;

    private void Start()
    {
        if (_useUWP)
        {
            ConnectUWP(host, port);
        }
        else
        {
            ConnectUnity(host, port);
        }
    }

#if UNITY_EDITOR
    private void ConnectUWP(string host, string port)
#else
    private async void ConnectUWP(string host, string port)
#endif
    {
#if UNITY_EDITOR
        errorStatus = "UWP TCP client used in Unity!";
#else
        try
        {
            if (exchangeTask != null) StopExchange();
        
            socket = new Windows.Networking.Sockets.StreamSocket();
            Windows.Networking.HostName serverHost = new Windows.Networking.HostName(host);
            await socket.ConnectAsync(serverHost, port);
        
            Stream streamOut = socket.OutputStream.AsStreamForWrite();
            writer = new StreamWriter(streamOut) { AutoFlush = true };
        
            Stream streamIn = socket.InputStream.AsStreamForRead();
            reader = new StreamReader(streamIn);

            RestartExchange();
            successStatus = "Connected!";
        }
        catch (Exception e)
        {
            errorStatus = e.ToString();
        }
#endif
    }

    private void ConnectUnity(string host, string port)
    {
#if !UNITY_EDITOR
        errorStatus = "Unity TCP client used in UWP!";
#else
        try
        {
            if (exchangeThread != null) StopExchange();
            
            client = new System.Net.Sockets.TcpClient(host, Int32.Parse(port));
            stream = client.GetStream();
            reader = new StreamReader(stream);
            writer = new StreamWriter(stream) { AutoFlush = true };

            RestartExchange();
            successStatus = "Connected!";
            
        }
        catch (Exception e)
        {
            errorStatus = e.ToString();
        }
#endif
    }

    private bool exchanging = false;
    private bool exchangeStopRequested = false;
    private string lastPacket = null;

    private string errorStatus = null;
    private string warningStatus = null;
    private string successStatus = null;
    private string unknownStatus = null;

    public void RestartExchange()
    {
#if UNITY_EDITOR
        if (exchangeThread != null) StopExchange();
        exchangeStopRequested = false;
        exchangeThread = new System.Threading.Thread(ExchangePackets);
        exchangeThread.Start();
#else
        if (exchangeTask != null) StopExchange();
        exchangeStopRequested = false;
        exchangeTask = Task.Run(() => ExchangePackets());
#endif
    }

    public void Update()
    {

        if (lastPacket != null)
        {
            text.GetComponent<TextMeshProUGUI>().text = lastPacket;
        }

        if (errorStatus != null)
        {
            stateManager.SetError(errorStatus);
            errorStatus = null;
        }
        if (warningStatus != null)
        {
            stateManager.SetWarning(warningStatus);
            warningStatus = null;
        }
        if (successStatus != null)
        {
            stateManager.SetSuccess(successStatus);
            successStatus = null;
        }
        if (unknownStatus != null)
        {
            stateManager.SetUnknown(unknownStatus);
            unknownStatus = null;
        }
    }

    public void ExchangePackets()
    {
        while (!exchangeStopRequested)
        {
            if (writer == null || reader == null) continue;
            exchanging = true;

            writer.Write("X\n");
            Debug.Log("Sent data!");
            string received = null;

#if UNITY_EDITOR
            byte[] bytes = new byte[client.SendBufferSize];
            int recv = 0;
            while (true)
            {
                recv = stream.Read(bytes, 0, client.SendBufferSize);
                received += Encoding.UTF8.GetString(bytes, 0, recv);
                if (received.EndsWith(".")) break;
            }
#else
            received = reader.ReadLine();
#endif

            lastPacket = received;
            Debug.Log("Read data: " + received);

            exchanging = false;
        }
    }

    
    public void StopExchange()
    {
        exchangeStopRequested = true;

#if UNITY_EDITOR
        if (exchangeThread != null)
        {
            exchangeThread.Abort();
            stream.Close();
            client.Close();
            writer.Close();
            reader.Close();

            stream = null;
            exchangeThread = null;
        }
#else
        if (exchangeTask != null) {
            exchangeTask.Wait();
            socket.Dispose();
            writer.Dispose();
            reader.Dispose();

            socket = null;
            exchangeTask = null;
        }
#endif
        writer = null;
        reader = null;
    }

    public void OnDestroy()
    {
        StopExchange();
    }

}
{%endhighlight%}
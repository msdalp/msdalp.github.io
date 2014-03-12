---
layout: post
title:  "Android Advanced Udp Client Example"
date:   2014-03-09 19:46:32
categories:
---
UDP (User Datagram Protocol) is anther commonly used protocol on the Internet. However, UDP is never used to send important data such as webpages,
database information, etc; UDP is commonly used for streaming audio and video. Streaming media such as Windows Media audio files (.WMA) , Real Player (.RM),
and others use UDP because it offers speed! The reason UDP is faster than TCP is because there is no form of flow control or error correction. 
The data sent over the Internet is affected by collisions, and errors will be present. Remember that UDP is only concerned with speed.
This is the main reason why streaming media is not high quality.<br>
Besides UDP a is connectionless protocol. Communication is datagram oriented. The integrity is guaranteed only on the single datagram.
Datagrams reach destination and can arrive out of order or don't arrive at all. Is more efficient than TCP because it uses non ack.
It's generally used for real time communication, where a little percentage of packet loss rate is preferable to the overhead of a TCP connection.

These are the general details of the UDP you can find anywhere.I assume you decided that it's better to use UDP on you Android application here some ways to implement 
client for UDP.A client can send data to server, can listen for coming data from a port or send a data to server and wait for answer.<br>

First we start with sending data to server.You need datagramsocket and datagrampacket to send data to server.DatagramSocket is the port we will send and DatagramPacket is the data
we will send represented as bytes.It may throw SocketException when you try to create a new DatagramSocket object or IOException when you try to send data.

{% highlight java %}
public class ClientSend implements Runnable {
        @Override
        public void run() {
            try {
                DatagramSocket udpSocket = new DatagramSocket(port);
                InetAddress serverAddr = InetAddress.getByName(ip);
                byte[] buf = ("The String to Send").getBytes();
                DatagramPacket packet = new DatagramPacket(buf, buf.length,serverAddr, port);
                udpSocket.send(packet);
            } catch (SocketException e) {
                Log.e("Udp:", "Socket Error:", e);
            } catch (IOException e) {
                Log.e("Udp Send:", "IO Error:", e);
            }
        }
}
{% endhighlight %}
Just sending data may not be enough.You can listen a port as well.Listening a port is done by infinite loops since you don't know when to stop listening.
We should prefer run flags for this than can close the loop when the job is done.If you just use while(true) you can see on your logcat it is still listening even the application
is not open on the main screen.The data size will be defined first as byte here so use a reasonable number.If you choose a small number it will give unexpected errors.

{% highlight java %}
public class ClientListen implements Runnable {
  @Override
  public void run() {
  boolean run = true;
	while (run) {
	  try {
	    DatagramSocket udpSocket = new DatagramSocket(port);
	    byte[] message = new byte[8000];
	    DatagramPacket packet = new DatagramPacket(message,message.length);
	    Log.i("UDP client: ", "about to wait to receive");
	    udpSocket.receive(packet);
	    String text = new String(message, 0, packet.getLength());
	    Log.d("Received data", text);
	  }catch (IOException e) {
	    Log.e("UDP client has IOException", "error: ", e);
	    run = false;
	  }
	}
  }
}	
{% endhighlight %}
There is also another way to write listener by giving it listening time.How can we use such an implementmation?
It may work first you send a data to server like "FILES" asking for files on the server.And set the listening time with setSoTimeout() method and start listening.
You should catch the exception which is SocketTimeoutException if there is no answer.
{% highlight java %}
public class ClientSendAndListen implements Runnable {
        @Override
        public void run() {
            boolean run = true;
            try {
                DatagramSocket udpSocket = new DatagramSocket(portVal);
		InetAddress serverAddr = InetAddress.getByName(ipVal);
                byte[] buf = ("FILES").getBytes();
                DatagramPacket packet = new DatagramPacket(buf, buf.length,serverAddr, port);
                udpSocket.send(packet);
                while (run) {
                    try {
                        byte[] message = new byte[8000];
                        DatagramPacket packet = new DatagramPacket(message,message.length);
                        Log.i("UDP client: ", "about to wait to receive");
                        udpSocket.setSoTimeout(10000);
                        udpSocket.receive(packet);
                        String text = new String(message, 0, p.getLength());
                        Log.d("Received text", text);                        
                    } catch (IOException e) {
                        Log.e(" UDP client has IOException", "error: ", e);
                        run = false;
                        udpSocket.close();
                    } catch (SocketTimeoutException e) {
                        Log.e("Timeout Exception","UDP Connection:",e);
                        run = false;
                        udpSocket.close();
                    }
                }
            } catch (SocketException e) {
                Log.e("Socket Open:", "Error:", e);
            }
        }
}
{% endhighlight %}
The last thing is how to call these Clients from you main activity.Remember these were Runnable so you should start these with Thread.
{% highlight java %}
package msd.udp.test;

import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.EditText;
import java.io.IOException;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.net.SocketException;
import java.net.SocketTimeoutException;
public class MainActivity extends Activity {
    private String ip;
    private int port;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        udpConnect = new Thread(new ClientSendAndListen()).start();
    }
}
{% endhighlight %}

---
layout: post
title:  "Android Advanced Udp Client Example"
date:   2014-03-09 19:46:32
categories:
---

Code example for now:

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

    private DatagramSocket UDP_SOCKET;
    private EditText ip, port;
    private Thread udpConnect;
    private String ipVal;
    private int portVal;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        ip = (EditText) findViewById(R.id.serverIp);
        port = (EditText) findViewById(R.id.serverPort);
    }

    public void login(boolean force) {

    }

    public void tryLogin(View view) {
        ipVal = ip.getText().toString();
        portVal = Integer.valueOf(port.getText().toString());
        ClientV1 connectionRequest = new ClientV1();
        udpConnect = new Thread(connectionRequest);
        udpConnect.start();
    }

    public class ClientV1 implements Runnable {

        @Override
        public void run() {
            boolean run = true;
            try {
                UDP_SOCKET = new DatagramSocket(portVal);
            } catch (SocketException e) {
                Log.e("Socket Open:", "Error:", e);
            }
            try {
                InetAddress serverAddr = InetAddress.getByName(ipVal);
                byte[] buf = ("connection").getBytes();
                DatagramPacket packet = new DatagramPacket(buf, buf.length,
                        serverAddr, Integer.valueOf(portVal));
                UDP_SOCKET.send(packet);
                while (run) {
                    try {
                        byte[] message = new byte[30000];
                        DatagramPacket p = new DatagramPacket(message,
                                message.length);
                        Log.i("***** UDP client: ", "about to wait to receive");
                        UDP_SOCKET.setSoTimeout(10000);
                        try {
                            UDP_SOCKET.receive(p);
                        } catch (SocketTimeoutException e) {
                            Log.d("Timeout Exception", "for UDP Connection");
                            run = false;
                            UDP_SOCKET.close();
                        }
                        String text = new String(message, 0, p.getLength());
                        Log.d("Received text", text);
                    } catch (IOException e) {
                        Log.e("***** UDP client has IOException", "error: ", e);
                        run = false;
                    }
                }
            } catch (Exception e) {
                Log.e("UDP", "Client Error", e);
            }
        }

    }

}
{% endhighlight %}

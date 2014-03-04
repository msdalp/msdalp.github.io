---
layout: post
title:  "Jetty Embedded Server With Post, Get, and Gson"
date:   2014-03-04 16:40:24
categories: jekyll update
---
Jetty is used in a wide variety of projects and products, both in development and production. Jetty can be easily embedded in devices, tools, frameworks, application servers, and clusters. See the <a href="http://www.eclipse.org/jetty/powered">Jetty Powered page</a> for more uses of Jetty.Here we want to use it as an embedded web server that runs with our application.
You should first download the required jetty jar files and import them. Basically you can just down <a href="http://repo1.maven.org/maven2/org/eclipse/jetty/aggregate/jetty-all/9.1.0.v20131115/jetty-all-9.1.0.v20131115.jar">jetty all</a> and use it there will no problem. For now the latest version is the 9.1 and it is the one we will use.Also Gson will be used and you should download it from <a href="https://code.google.com/p/google-gson/downloads/detail?name=google-gson-2.2.4-release.zip&can=2&q=">this</a> link.
First create a class that extends Server and by giving port to its constructor we will be able to start it easily.
{% highlight java %}
package com.msd.jetty;
 
import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.servlet.ServletContextHandler;
import org.eclipse.jetty.servlet.ServletHolder;
 
public class Jetty extends Server {
 
    private static final Gson gson = new GsonBuilder().
    setDateFormat("yyyy-MM-dd HH:mm:ss").create();
 
    public Jetty(int port) {
        super(port);
 
        ServletContextHandler context = new ServletContextHandler(ServletContextHandler
        .NO_SESSIONS);
        context.setContextPath("/test");
        context.addServlet(new ServletHolder(new GetName(gson)), "/GetName/*");
        context.addServlet(new ServletHolder(new SaveName()), "/SaveName/*");
        this.setHandler(context);
        this.setStopAtShutdown(true);
    }
 
}
{% endhighlight %}
By using context.addServlet() method you can add servlets easily.addServlet() takes two parameters and the first one is the class name that will be called when this servlet called and the second one is the name how you will call it.
{% highlight java %}
context.addServlet(new ServletHolder(new SaveName()), "/SaveName/*");
{% endhighlight %}
  will be called as localhost:portNumber/test/SaveName 
GET methods are defined using doGet and POST methods are defined with doPost.SaveName and GetName classes are

{% highlight java %}
package com.msd.jetty;
 
 
import java.io.IOException;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
 
@SuppressWarnings("serial")
public class SaveName extends HttpServlet {
 
    public SaveName() {
    }
 
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
                                             throws ServletException, IOException {
        try {
            final long timeStamp = System.currentTimeMillis();
            final String name = request.getParameter("name");
            final String surName = request.getParameter("surName");
            System.out.println("name:"+name+" surname:"+surName);
            response.setStatus(HttpServletResponse.SC_OK);
 
        } catch (Exception ex) {
            response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
 
        } finally {
            response.setContentType("text/html;charset=UTF-8");
            response.getWriter().println("");
            response.getWriter().close();
        }
    }
 
}
{% endhighlight %}
And the get method we will use:
{% highlight java %}
package com.msd.jetty;
 
import com.google.gson.Gson;
import com.google.gson.JsonObject;
import java.io.IOException;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
 
@SuppressWarnings("serial")
public class GetName extends HttpServlet {
 
    private final Gson gson;
 
    public GetTeamDictionary(Gson gson) {
        this.gson = gson;
    }
 
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) 
                                            throws ServletException, IOException {
        String output = "";
        try {
            final JsonObject retVal = new JsonObject();
            
            retVal.add("name", "jetty");
            retVal.add("surname", "is awasome");
            output = retVal.toString();
            response.setStatus(HttpServletResponse.SC_OK);
 
        } catch (Exception ex) {
            response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
 
        } finally {
            response.setContentType("text/html;charset=UTF-8");
            response.getWriter().println(output);
            response.getWriter().close();
        }
    }
 
}
{% endhighlight %}


It's all that simple and now you only need to start to jetty server from your main class.
{% highlight java %}

package com.msd.jetty;
 
public class App {
 
    public static void main(String[] args) {
            final Jetty jetty = new Jetty(8080);
            jetty.start();
            Thread.sleep(500);
            if (false == jetty.isStarted()) {
                throw new Exception("Cannot start jetty server");
            }
    }
}
{% endhighlight %}
When your application is running you can call localhost:8080/test/GetName and it will return to you a json object like this
{% highlight json %}
{
  "name": "jetty",
  "surname": "is awesome"
}
{% endhighlight %}
You can use HTTP:POST to send data to server by using localhost:8080/test/SaveName and your post parameters should be named as "name" and "surName".It is the simplest thing you can do with jetty embedded server.I use it to get data from database using parameters and posting data into database and it's really fast and lightweight which makes it really feasible.



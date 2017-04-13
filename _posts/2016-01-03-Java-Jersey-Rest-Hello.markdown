---
layout: post
title:  "Rest API Example with Java Jersey and Tomcat "
date:   2016-01-03 10:43:32
categories:
---


The definition of the Jersey framework from their website:

Jersey framework is more than the JAX-RS Reference Implementation. Jersey provides it’s own API that extend the JAX-RS toolkit with additional features and utilities to further simplify RESTful service and client development. Jersey also exposes numerous extension SPIs so that developers may extend Jersey to best suit their needs.

You can read details from <a href="https://jersey.java.net">https://jersey.java.net</a>

What I will try to show is creating a project from beginning and put more in time to make it a whole complete structure in this order:

 - Create a project by using HTTP methods.
 - Add database connection to the project and Authentication (tokens, roles)
 - Add error mappings, logging requests and responses
 - Jackson specific settings

I will be using Netbeans 8, Tomcat 7, Jersey 2.9.1, and Java 7.

Firstly create a new project by selecting Maven Web project.

![new project ss](/assets/img/new_maven_project.png)<br>

After creating your project add dependencies as given below and we will be adding other dependencies when it is needed.
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.jersey.test</groupId>
    <artifactId>JerseyTest</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>

    <name>JerseyTest</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>
        <dependency>
            <groupId>org.glassfish.jersey.containers</groupId>
            <artifactId>jersey-container-servlet</artifactId>
            <version>2.9.1</version>
        </dependency>
        <dependency>
            <groupId>org.glassfish.jersey.media</groupId>
            <artifactId>jersey-media-json-jackson</artifactId>
            <version>2.22.1</version>
        </dependency>

    </dependencies>
    <build>
        <finalName>JerseyTest</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>2.6</version>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.5.1</version>
                <configuration>
                    <source>1.7</source>
                    <target>1.7</target>
                </configuration>
            </plugin>

        </plugins>
    </build>

</project>
{% endhighlight %}

Jersey provides an abstract class Application declaring root resource and provider classes, and root resource and provider singleton instances. A Web service may extend this class to declare root resource and provider classes.

Alternatively it is possible to reuse ResourceConfig - Jersey's own implementations of Application class that we will be using.

Simply you give the packages that you want to include in the project for deployment and register the properties like JSON and GZIP. Also you can register filters, singletons and other things as well.


{% highlight java %}
package com.jersey.test.app;

import javax.ws.rs.ApplicationPath;
import org.glassfish.jersey.jackson.JacksonFeature;
import org.glassfish.jersey.server.ResourceConfig;

/**
 *
 * @author Mustafa Alparslan <msdalp.github.io>
 */
@ApplicationPath("")
public class RestApp extends ResourceConfig {
    public RestApp() {
        packages(new String[]{
            "com.jersey.test.data"
        });
        register(JacksonFeature.class);
    }
}
{% endhighlight %}

At this stage if your IDE is giving error about web.xml or context.xml then simply put these files and will be used in the future posts.

web.xml
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="3.1" xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd">

</web-app>
{% endhighlight %}
context.xml
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<Context antiJARLocking="true" path="/JerseyTest"/>
{% endhighlight %}

The package I included in the main class was "com.jersey.test.data" and for the sake of simplicity I will add User model and the Users service to show methods.

These two classes have no specific property (user model and a map to put users inside while we have no database connection).
{% highlight java %}
package com.jersey.test.data;

/**
 *
 * @author Mustafa Alparslan <msdalp.github.io>
 */
public class User {
    public long id;
    public String name;
    public String password;

    public User() {
    }

    public User(long id, String name, String password) {
        this.id = id;
        this.name = name;
        this.password = password;
    }
}
{% endhighlight %}

{% highlight java %}
package com.jersey.test.app;

import com.jersey.test.data.User;
import java.util.HashMap;
import java.util.Map;

/**
 *
 * @author Mustafa Alparslan <msdalp.github.io>
 */
public class Data {

    public static Map<Long,User> users = new HashMap<>();

}
{% endhighlight %}




{% highlight java %}
package com.jersey.test.data;

import com.jersey.test.app.Data;
import javax.servlet.ServletContext;
import javax.servlet.http.HttpServletRequest;
import javax.ws.rs.Consumes;
import javax.ws.rs.DELETE;
import javax.ws.rs.GET;
import javax.ws.rs.POST;
import javax.ws.rs.PUT;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;
import javax.ws.rs.Produces;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;

/**
 *
 * @author Mustafa Alparslan <msdalp.github.io>
 */
@Path("/users")
public class Users {

    @Context
    ServletContext servletContext;
    @Context
    HttpServletRequest servletRequest;

    @Produces({MediaType.APPLICATION_JSON})
    @Path("{userId}")
    @GET
    public Response processGet(@PathParam("userId") Long userId) {
        if (Data.users.containsKey(userId)) {
            return Response.status(Response.Status.OK)
                    .entity(Data.users.get(userId)).type("application/json").build();
        } else {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(new Msg(Msg.Cond.error, "Not Found")).type("application/json").build();
        }
    }

    @Produces({MediaType.APPLICATION_JSON})
    @Path("{userId}")
    @DELETE
    public Response processDel(@PathParam("userId") long userId) {
        if (Data.users.containsKey(userId)) {
            Data.users.remove(userId);
            return Response.status(Response.Status.OK)
                    .entity(new Msg(Msg.Cond.success, "Operation successful")).type("application/json").build();
        } else {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(new Msg(Msg.Cond.error, "Not Found")).type("application/json").build();
        }
    }

    @Consumes(MediaType.APPLICATION_JSON)
    @Produces({MediaType.APPLICATION_JSON})
    @Path("")
    @POST
    public Response processAdd(User user) {
        Data.users.put(user.id, user);
        return Response.status(Response.Status.OK)
                .entity(Data.users.values()).type("application/json").build();
    }

    @Consumes(MediaType.APPLICATION_JSON)
    @Produces({MediaType.APPLICATION_JSON})
    @Path("")
    @PUT
    public Response processUpdate(User user) {
        Data.users.put(user.id, user);
        return Response.status(Response.Status.OK)
                .entity(Data.users.values()).type("application/json").build();
    }

}
{% endhighlight %}

Your service definition starts with a "@Path("/users")" annotation which defines the root path of the class. You can add more lower levels before the methods like "@Path("{userId}/orders")" which will take user id as parameter and return the orders of that user. However be aware that adding conflicting classes will broke the URL schema and calling these URLs will give some runtime Exceptions.

You define the name of HTTP method with annotations  as well (by using @GET, @POST, @PUT, @DELETE).

For the requests with @GET and @DELETE methods PathParam (directly in url like /users/14) or QueryParam (by using ? and values behind like /users?id=14) can be used (together or only one of them). However I highly recommend that if it is a REST-API then use PathParams to make it more readable.

Also you should read about REST a bit from
<a href="http://stackoverflow.com/questions/671118/what-exactly-is-restful-programming"> this stackoverflow link</a>. Honestly I don't care about the talk 'your api should match 100% to the rules otherwise your api is not a rest' crap. The main idea is creating a standard that allows developers (for both server and client side) to make their jobs with ease. At the end call your api 'like REST' and then everyone will be happy.

Even though I said that is not important but at least you should be able to provide these properties.

- Proper url naming: It is quite easy and important thing since creating urls properly would make much easier to use you API.

Please do not use sentences at the urls and at least it should match the form of

{% highlight bash %}
/users -> to add or update the user
/users/{userId} -> to get or delete the user
/users/{userId}/orders -> to get the orders of the user
{% endhighlight %}

- Proper usage of HTTP methods:

It should use @GET to retrieve data, @POST-@PUT to add and update data, and @DELETE to delete data. It may change with the specific methods but using this in your main models should be quite easy.

- Proper usage of HTTP Codes:

It is quite important to send proper HTTP Codes in your responses. It would be much easier to manage to responses and take action according to these codes (like if the code is 401 and can not refresh token then redirect to the login page). Again there are many codes but:

{% highlight bash %}
200 -> successful request
400 -> bad request
401 -> auth is not valid or expired
403 -> forbidden
500 -> internal error
{% endhighlight %}

- Security with Auth Token and others if more needed (like JWT)
- Proper request responses and errors  


I will continue the REST specifications in another post and return to the JERSEY.

For the requests with @POST and @PUT methods FormParam (like usual form parameter with username=msd&password=alp) or directly the object representation of request as it is used in the example (User user). However you must define the '    @Consumes(MediaType.APPLICATION_JSON)' annotation for the method.

Also you may notice that some methods use the same path. Separation of unique methods are done by classical java method name and parameters on the compile time,  path and http method on the runtime. Therefore you can define two methods with the same path and parameter but different HTTP method (GET or DELETE).

@Produces({MediaType.APPLICATION_JSON}) annotation takes a list of MediaType (or string like 'application/json') that defines the possible return types. It is possible to use two different response type in one method like
{% highlight java %}
     @Produces({MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML})
{% endhighlight %}

This is all the things you need for a simple project. I will be adding Redis, PostgreSQL, and Mysql for data access and caching in the next posts.

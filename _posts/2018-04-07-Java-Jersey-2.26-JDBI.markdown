---
layout: post
title:  "Java Jersey Database with Jersey 2.26 and JDBI"
date:   2018-04-07 21:32:52
categories:
---
In the previous blog we created an empty Jersey project to have a hello world service but it is quite useless as you expect. To make it more realistic I will add database connection and implement some methods with JDBI. You can use other options like MyBatis or Hibernate but I really don't like giving all control to the ORM. If you haven't used JDBI before it is the default JDBC Api on Dropwizard and actually it is quite similar to the Spring JDBC Template. Since we will be using simple stuff you will get the idea very quickly so don't worry about that. 

How does this really work? We don't have a database connection at this stage and how do we start the db connection with application start and close it when the application is closed? 

Answer is `ServletContextListener` for the web app which enables us to have methods when context initialized and destroyed. It basically gives access to the start and end of your application. Also we could be using `@PostConstruct` and `@PreDestroy` for `RestApplication` class and as expected it will run after app is initialized and before getting destroyed. Additionally if your app is not deployed as war but a jar then you can start your modules at start and close them on `Shutdown Hook` properly. 

{% highlight xml %}
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>
{% endhighlight %}

When you try to implement `ServletContextListener` you will notice it is not in our project scope yet. Add `javax.servlet-api` to your pom to be able to use `ContextListener` properly. Next will be defining a custom listener and adding it to our `web.xml` file. 

{% highlight java %}
public class MyListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent) {
        System.out.println("Context started");
    }

    @Override
    public void contextDestroyed(ServletContextEvent servletContextEvent) {
        System.out.println("Context dying");
    }
}
{% endhighlight %}
 
`ServletContextListener` has two methods as contextInitialized and contextDestroyed. You can check the logs to see where these methods run. Also after defining the custom listener we need to add it to `web.xml`.
{% highlight xml %}
<listener>
    <description>Context Listener</description>
    <listener-class>com.blog.api.MyListener</listener-class>
</listener>
{% endhighlight %}
 
Your web.xml file only had a simple webapp definition before and now it includes the context listener. 

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>

<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1"
         id="blog-api">
    <listener>
        <description>Context Listener</description>
        <listener-class>com.blog.api.MyListener</listener-class>
    </listener>
</web-app>
{% endhighlight %}

You can try running the app and see where these methods print out the messages we put for now temporarily. 

We can't use a single db connection for real user applications. It would require us to have a proper connection pool which will be done by HikariCP. Details for HikariCP can be found at [https://github.com/brettwooldridge/HikariCP](https://github.com/brettwooldridge/HikariCP). It is also possible to use Tomcat Pool, c3p0, or dbcp2 but especially for production hikari has the best implementation with proper Thread safe methods. Also as I mentioned before, JDBI is the JDBC wrapper we will be using. Add both of these dependencies to your pom file. 

{% highlight xml %}
<dependency>
    <groupId>org.jdbi</groupId>
    <artifactId>jdbi3-core</artifactId>
    <version>3.1.1</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.jdbi/jdbi3-sqlobject -->
<dependency>
    <groupId>org.jdbi</groupId>
    <artifactId>jdbi3-sqlobject</artifactId>
    <version>3.1.1</version>
</dependency>
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>3.0.0</version>
</dependency> 
{% endhighlight %}

Now I want to have a database manager class for just storing the `HikariDataSource` and `Jdbi` object for database access. 

{% highlight java %}
public class DBIManager {
    private static DBIManager instance;

    private final HikariDataSource hikariDataSource;
    private final Jdbi jdbi;

    private DBIManager() throws NamingException {
        this.hikariDataSource = (HikariDataSource) new InitialContext().lookup("java:comp/env/jdbc/hikariSrc");
        this.jdbi = Jdbi.create(hikariDataSource);
    }

    public static DBIManager getInstance() {
        return instance;
    }

    public static synchronized void start() throws ServerErrorException {
        if (instance == null) {
            try {
                instance = new DBIManager();
            } catch (NamingException ex) {
                throw new ServerErrorException(Response.Status.INTERNAL_SERVER_ERROR, ex);
            }
        }
    }

    public static void shutdown() {
        instance.hikariDataSource.close();
    }

    public static Jdbi getJdbi() {
        return instance.jdbi;
    }
}
{% endhighlight %}

With this manager class, we will start the connection on `contextInitialized` and close it with `contextDestroyed`, and access to database connection with `getJdbi()` method. The connection information will be stored in context xml or you can just set in start method with `HikariConfig` object. Finally we can update `context` methods to start and stop database connection pool.

{% highlight java %}
public class MyListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent) {
        DBIManager.start();
    }

    @Override
    public void contextDestroyed(ServletContextEvent servletContextEvent) {
        DBIManager.shutdown();
    }
}
{% endhighlight %}

As you might notice connection details are retrieved from the context lookup but haven't done anything about it. First create a folder for the `context.xml` just next to `WEB-INF` as `META-INF` and put the `context.xml` in it. 

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<Context path="/">

    <Resource name="jdbc/hikariSrc"
              auth="Container"
              factory="com.zaxxer.hikari.HikariJNDIFactory"
              type="javax.sql.DataSource"
              driverClassName="com.mysql.jdbc.Driver"
              jdbcUrl="jdbc:mysql://localhost:3306/jersey?useUnicode=true&amp;characterEncoding=utf8&amp;useSSL=false&amp;serverTimezone=UTC"
              autoCommit="true"
              characterEncoding="utf8"
              useUnicode="true"
              encoding="UTF-8"
              leakDetectionThreshold="10000"
              poolName="BlogJeysey"
              password="mypassword"
              username="root"
              minimumIdle="5" maximumPoolSize="20" connectionTimeout="20000" maxLifetime="1800000"/>
</Context>
{% endhighlight %}

These are specific HikariConfig details which you can update as you want but it should be enough for now to run our application. Also need to define this context in `web.xml`. After adding it to `web.xml` it should be something like below.
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>

<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1"
         id="blog-api">
    <listener>
        <description>Context Listener</description>
        <listener-class>com.blog.api.MyListener</listener-class>
    </listener>

    <resource-ref>
        <description>MySQL Jersey</description>
        <res-ref-name>jdbc/hikariSrc</res-ref-name>
        <res-type>javax.sql.DataSource</res-type>
        <res-auth>Container</res-auth>
    </resource-ref>

</web-app>
{% endhighlight %}

If you run without a proper database connection you will get the error `Caused by: java.net.ConnectException: Connection refused (Connection refused)`. 

I will run mysql on my local with docker. Also my default database name will be `jersey` so make sure you created a schema as `create database jersey;`. 

`docker run -d --name=jersey-mysql -e MYSQL_ROOT_PASSWORD=mypassword -v mysql:/var/lib/mysql mysql` 

Create a db package under `com.blog.api` and have two classes in it as `User` and `UserDao`. `User` is our resource model and make sure you implement empty constructor and getters-setters. Otherwise JDBI will fail to initialize the model while mapping it. 
{% highlight java %}
public class User {
    private int id;
    private String name;
}
{% endhighlight %}

`UserDao` is the JDBI interface where you define your database logic. You can read JDBI specific details from [http://jdbi.org](http://jdbi.org). 
{% highlight java %}
public interface UserDao{

    @SqlQuery("SELECT * FROM user;")
    @RegisterBeanMapper(User.class)
    List<User> getUsers();

    @SqlQuery("SELECT * FROM user WHERE id=:id")
    @RegisterBeanMapper(User.class)
    User getUser(@Bind("id") int id );
}
{% endhighlight %}

Last thing will be removing the Test resource we had created in previous post and create a `UserResource` class in rest package.
{% highlight java %}
@Path("users")
public class UserResource {

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    @Path("")
    public Response getUsers(){
        return Response.ok().entity(DBIManager.getJdbi().onDemand(UserDao.class).getUsers()).build();
    }


    @GET
    @Produces(MediaType.APPLICATION_JSON)
    @Path("{userId}")
    public Response getUser(@PathParam("userId") int userId){
        return Response.ok().entity(DBIManager.getJdbi().onDemand(UserDao.class).getUser(userId)).build();
    }
}
{% endhighlight %}

I will implement two rest calls for `User` as get all users and get a specific user. To make sure you have all the data required put the default models as below in the `RestApp` class. It will keep inserting new users as we re-run again but doesn't really matter. You access the JDBI calls `jdbi.onDemand(Dao.class)` and have your method call on it. It will close the connection automatically and will use the pool properly with Hikari. 
{% highlight java %}
@PostConstruct
public void postCreate(){
    DBIManager.getJdbi().withHandle(handle -> {
        handle.execute("CREATE TABLE IF NOT EXISTS `user` (\n" +
                "  `id` int(11) NOT NULL AUTO_INCREMENT,\n" +
                "  `name` varchar(45) NOT NULL,\n" +
                "  PRIMARY KEY (`id`)\n" +
                ") ENGINE=InnoDB DEFAULT CHARSET=utf8;\n");
        handle.execute("INSERT INTO user (name) VALUES (?)",  "Alice");
        handle.execute("INSERT INTO user (name) VALUES (?)",  "John");
        handle.execute("INSERT INTO user (name) VALUES (?)",  "Msd");
        return null;
    });
}
{% endhighlight %}

These should be all we need for a proper representation of user table as a resource. First try a specific user retrieval as `/users/{userId}` and it should return a single user object.


`http://localhost:8080/users/2`
{% highlight json %}
{"id":2,"name":"Repa"}
{% endhighlight %}

Next you can check all the users with `/users/` call and see the list of users.

`http://localhost:8080/users/`
{% highlight json %}
[{"id":1,"name":"John"},{"id":2,"name":"Repa"},{"id":3,"name":"Smith"}]
{% endhighlight %}

You can check the project from [https://github.com/msdalp/jersey-jdbi-sample](https://github.com/msdalp/jersey-jdbi-sample). Next I will convert Dao classes to annotation and inject it to resources automatically. 
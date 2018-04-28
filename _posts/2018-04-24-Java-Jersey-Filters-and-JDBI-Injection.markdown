---
layout: post
title:  "Java Jersey Auth Filter and JDBI Injection"
date:   2018-04-24 12:00:00
categories:
---

I started with a dummy jersey project and later added jdbi to that. You can see both posts below 
[http://msdalp.github.io/2018/03/28/Java-Jersey-2.26-Tutorial/](http://msdalp.github.io/2018/03/28/Java-Jersey-2.26-Tutorial/)
[http://msdalp.github.io/2018/04/07/Java-Jersey-2.26-JDBI/](http://msdalp.github.io/2018/04/07/Java-Jersey-2.26-JDBI/)

I will continue from the last part which you can pull from [https://github.com/msdalp/jersey-jdbi-sample](https://github.com/msdalp/jersey-jdbi-sample) the `master` branch. All work done on this post will go to `auth_filter-dao` branch. 

Now I want add these in order:

- Add a proper user class with password and write jdbi implementation for that. 
- Add a login method with token you users can login.
- Users should have different roles so later we can utilize Jersey's in role and have proper role based service calls. 
- Add a filter to check tokens to see if user is authorized or not.
- Implement user role details
- Write a Dao annotation so we can inject Jdbi interfaces easily.

For the simplest case I need a User model with unique username, a bcrypt hashed password, and a role selected from 'admin', 'mod', or 'user' which will be defaulted as 'user'.

{% highlight sql %}
CREATE TABLE auth_user (
  id int(11)  unsigned NOT NULL AUTO_INCREMENT,
  username varchar(45) NOT NULL,
  password binary(60) NOT NULL,
  role enum('admin','mod','user') NOT NULL DEFAULT 'user',
  PRIMARY KEY (id),
  UNIQUE KEY 'username_UNIQUE' (username)
) DEFAULT CHARSET=utf8;
{% endhighlight %}

With this we create an `auth_user` table. I don't like using `user` table because it is possible to mixing up with mysql user table. id's are auto generated and `username` is unique. `password` is defined as binary(60) since we are storing the result of bcrypt hashed password but not the text itself. Lastly an enum for user roles as admin,mod,and default user itself.

{% highlight java %}
public class AuthUser {
    private enum Role{
        admin,mod,user
    }

    private int id;
    private String username;
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private String password;
    private Role role;
    
    @JsonIgnore
    public boolean isValid(){
        return !(this.username == null || this.password == null);
    }
}
{% endhighlight %}

Representation of that model in java would be like above and make sure you have proper getters and setters, otherwise JSON conversion will fail. Also do not forget to add `@JsonProperty` for Jackson and it can ignore the password part while returning the model. JDBI interface for this class is given below and it is pretty straightforward.
{% highlight java %}
public interface UserDao{

    @SqlQuery("SELECT * FROM auth_user")
    @RegisterBeanMapper(AuthUser.class)
    List<AuthUser> getUsers();

    @SqlQuery("SELECT * FROM auth_user WHERE id=:id")
    @RegisterBeanMapper(AuthUser.class)
    AuthUser getUser(@Bind("id") int id );

    @SqlUpdate("INSERT into auth_user(username, password, role) values (:username, :password, :role)")
    @GetGeneratedKeys
    int addUser(@BindBean AuthUser user);

    @SqlUpdate("UPDATE auth_user set username=:username, password=:password, role=:role where id=:id")
    void updateUser(@BindBean AuthUser user);

    @SqlUpdate("DELETE from auth_user where id=:id")
    void deleteUser(@Bind("id") int id);
}
{% endhighlight %}


We also want to have a token table so we can store tokens assigned to users after login. `token` itself will be varchar(32) since we will using `BigInteger(130, secureRandom).toString(32)` for generating tokens. `expire_at` will be given as one day by default and `user_id` is the user which is token is assigned.

{% highlight sql %}
CREATE TABLE token (
  id int(11) unsigned NOT NULL AUTO_INCREMENT,
  token varchar(32) NOT NULL,
  expire_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  user_id int(11) unsigned NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY 'token_UNIQUE' (token),
  KEY token_user_id_idx (user_id),
  CONSTRAINT token_user_id FOREIGN KEY (user_id) REFERENCES auth_user (id) ON DELETE NO ACTION ON UPDATE NO ACTION
) DEFAULT CHARSET=utf8;
{% endhighlight %}

{% highlight java %}
public class Token {

    @JsonIgnore
    private int id;
    private String token;
    private int userId;
    private Timestamp expireAt;
}
{% endhighlight %}

For tokens we will implement methods for retrieving and inserting one.`@GetGeneratedKeys` returns the insert id of token or user which will be put on response for the client.  

{% highlight java %}
public interface TokenDao {

    @SqlQuery("SELECT * FROM token;")
    @RegisterBeanMapper(Token.class)
    List<Token> getTokens();

    @SqlQuery("SELECT * FROM token WHERE token=:token and expire_at > now()")
    @RegisterBeanMapper(Token.class)
    Token getToken(@Bind("token") String token);

    @SqlUpdate("INSERT into token(token,user_id,expire_at) values (:token, :userId, DATE_ADD(NOW(), INTERVAL 1 DAY))")
    @GetGeneratedKeys
    int addToken(@BindBean Token token);
}
{% endhighlight %}

Now let's move one the User resource class and implement methods for add,update, and delete. But first we need a Bcrypt class for hashing user passwords. There are few implementations or you can have a class for that which are quite easy to find on Github. I am going to add JBcrypt to my dependencies and use it from there. 

{% highlight xml %}
<!-- https://mvnrepository.com/artifact/org.mindrot/jbcrypt -->
<dependency>
    <groupId>org.mindrot</groupId>
    <artifactId>jbcrypt</artifactId>
    <version>0.4</version>
</dependency> 
{% endhighlight %}

For the User resource I will follow the REST style I explained in the first post. 
```
GET users = HTTP 200 get all users
GET users/{userId} = HTTP 200 get a specific user
POST users = HTTP 201 creates a new user and accept user body as json
PUT users/{userId} = HTTP 201 update the user specified on user id and  and accept user body as json
DELETE users/{userId} = HTTP 204 delete a specific user
```

Check the methods below and read the details through the comments I left on those. 
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

    @POST
    @Produces(MediaType.APPLICATION_JSON)
    // we can set consume type in here as json directly
    @Consumes(MediaType.APPLICATION_JSON)
    @Path("")
    // Jackson will convert the posted json to model directly.
    // you can just consume the object.
    public Response addUser(AuthUser user){
        // generate a new salt for user with 10 rounds
        final String userSalt = BCrypt.gensalt(10, new SecureRandom());
        // TODO validate password
        // hash the password with the salt created above.
        // I just assumed password is proper but you should have
        // password validations before doing any process
        user.setPassword(BCrypt.hashpw(user.getPassword(),userSalt));
        // addUser method will return the created user id and
        // we set that id to user body
        user.setId(DBIManager.getJdbi().onDemand(UserDao.class).addUser(user));
        // return the user with HTTP 201 status
        return Response.status(Response.Status.CREATED).entity(user).build();
    }

    @DELETE
    @Produces(MediaType.APPLICATION_JSON)
    @Path("{userId}")
    public Response delUser(@PathParam("userId") int userId){
        DBIManager.getJdbi().onDemand(UserDao.class).deleteUser(userId);
        // return no content 204 on delete. there is no
        // point to return any other message and transfer data
        return Response.noContent().build();
    }

    @PUT
    @Produces(MediaType.APPLICATION_JSON)
    @Consumes(MediaType.APPLICATION_JSON)
    @Path("{userId}")
    public Response updateUser(@PathParam("userId") int userId, AuthUser user){
        // set the user id so we can just use user bean on jdbi
        user.setId(userId);
        // update user with jdbi method
        DBIManager.getJdbi().onDemand(UserDao.class).updateUser(user);
        // return HTTP 201 with the body
        return Response.status(Response.Status.CREATED).entity(user).build();
    }
}
{% endhighlight %}

Use Postman or any similar tool to test these services. Create your user with POST  and the json below and use GET 
to check the user you have created.
{% highlight json %}
{
  "username": "richard",
  "role": "user",
  "password": "fornow"
}
{% endhighlight %}

It should return a result similar to this and later you can try updating and deleting user service calls. 
{% highlight json %}
[
    {
        "id": 1,
        "username": "msdalp",
        "role": "admin"
    },
    {
        "id": 9,
        "username": "richard",
        "role": "user"
    }
]
{% endhighlight %}

After creating user we have all tools we need to have a proper login method. Now create an `AuthResource` class and have a `POST` method for login.
Simply check if user input is valid and if not throw error. Later check if user exists and if it is validate password. A `POST` method with consuming user model as json or `form-url-encoded` formatted username and password fields possible. Both works quite well so pick one that you are comfortable with. I will be using JSON consuming one. Again you can go through the comments and understand the logic easily. 

{% highlight java %}

@Path("auth")
public class AuthResource {
    private static final Random secureRandom = new SecureRandom();

    @POST
    @Produces(MediaType.APPLICATION_JSON)
    // we can set consume type in here as form url directly
    @Consumes(MediaType.APPLICATION_JSON)
    @Path("")
    public Response login(AuthUser userLogin) {

        if (!userLogin.isValid()) {
            return Response.status(401).entity(new FailedLogin()).build();
        }

        final AuthUser user = DBIManager.getJdbi().onDemand(UserDao.class).getUserByName(userLogin.getUsername());

        // make sure user or password is available
        // check if password matches as well
        // otherwise throw not authorized
        if (user == null || !BCrypt.checkpw(userLogin.getPassword(), user.getPassword())) {
            return Response.status(401).entity(new FailedLogin()).build();
        }

        final Token token = new Token();
        token.setUserId(user.getId());
        // you can use any kind of token generation method
        token.setToken(new BigInteger(130, secureRandom).toString(32));
        // expire after one day
        token.setExpireAt(Timestamp.from(Instant.now().plus(1,ChronoUnit.DAYS)));
        // get the inserted id
        token.setId(DBIManager.getJdbi().onDemand(TokenDao.class).addToken(token));
        return Response.status(Response.Status.CREATED).entity(token).build();
    }

}
{% endhighlight %}

It is obviously useless right now without a proper filter to check if user has authorization. You can just login and it will return a token but we are using it anywhere. Jersey provides `ContainerRequestFilter` and by implementing this interface you can have a filter method. It enables you to have chance check or validate before processing all request. What we will do is check the `Authorization` header and get token from it. If there is no token or the token we got is invalid and abort the request with `HTTP 401` not authorized. Also we will allow this `auth` path so people can login without a 
token. 

{% highlight java %}
@Provider
@PreMatching
@Priority(Priorities.AUTHORIZATION)
public class AuthFilter implements ContainerRequestFilter {

    @Context
    HttpServletRequest webRequest;

    @Context
    HttpServletResponse servletResponse;

    @Override
    public void filter(ContainerRequestContext requestContext) {
        // get token from auth header.
        final String tokenHeader = webRequest.getHeader(HttpHeaders.AUTHORIZATION);
        // get the top path from url so we can check if is is authorization call
        final String ws = requestContext.getUriInfo().getPath().split("/")[0];

        final Token token = DBIManager.getJdbi().onDemand(TokenDao.class).getToken(tokenHeader);
        // if user is not trying to login and token is not available then throw not authorized
        if (!ws.equals("auth") && token == null){
            requestContext.abortWith(Response.status(401).entity(new NoAuth()).build());
        }
    }
}
{% endhighlight %}

{% highlight java %}
public class NoAuth {
    public String message  = "Auth Token Error";
    public String detail = "Auth token is not valid or expired";

    public NoAuth() {}
}
{% endhighlight %}

You can inject resources with `@Context` annotation. There some resources on default which are provided by Jersey like `HttpServletRequest`, `HttpServletResponse` or `UriInfo`. Also you can define your custom resources to inject like database source or `SecurityContext`. By utilizing these two we will get `Authorization` header and the top request path. If you just check token return error if it is not valid then there will no way for users to login properly. Therefore you need to exclude the `auth` resource from the token check. Also I am defining errors as just classes so I can show it quickly but a proper way is having a `GenericExceptionMapper` and catch errors there for logging properly and returning generic errors. I will probably show it later in a post.

Now call `users` service again without a token and it will return an `401` error to you saying `Auth token is not valid`. After adding the tokens we got from earlier calls it will work again properly. 

{% highlight json %}
{
    "message": "Auth Token Error",
    "detail": "Auth token is not valid or expired"
}
{% endhighlight %}

We have last two parts which implementing user roles so normal users can not add or delete users. It will be possible thanks to `SecurityContext` that Jersey provides we will use the roles we defined earlier. 

Start with implementing `Principal` interface as `AuthUser implements Principal`. You will implement two methods, return a name and check if implies. Ignore the second and just return username as name for now. Also make sure you made `Role` enum public so we can use it in `SecurityContext`.

{% highlight java %}
@Override
public String getName() {
    return this.username;
}

@Override
public boolean implies(Subject subject) {
    return false;
}

public enum Role{
    admin,mod,user
}
{% endhighlight %}

Now we can create a `SecurityUser` which implements `SecurityContext`. It will ask you to add `isSecure`, `authenticationScheme`, `isUserInRole`, and `isSecure`. We implemented `Principal` on `AuthUser` earlier so we can return that user in here as principal.

{% highlight java %}
public class SecurityUser implements SecurityContext {

    private final AuthUser user;

    public SecurityUser(AuthUser user) {
        this.user = user;
    }
    
    @Override
    public Principal getUserPrincipal() {
        return user;
    }

    @Override
    public boolean isUserInRole(String s) {
        return user.getRole().name().equals(s);
    }

    @Override
    public boolean isSecure() {
        return true;
    }

    @Override
    public String getAuthenticationScheme() {
        return "TOKEN";
    }
}
{% endhighlight %}
We added user into this object and it will our principal. Authentication scheme is "token" or any other name you prefer. Set isSecure as true and compare the role given to us with the user's role and return true if they are equal.
This is just a class though. How do we set this into our `Context`? As expected it will require us to register `RolesAllowed` as `register (RolesAllowedDynamicFeature.class);` in our `RestApp` class. After this in the filter method if it is actually a valid token then get the user and 
set as `SecurityContext`. I updated filter method a bit so we can just return if it is a `auth` request. 

{% highlight java %}
@Override
public void filter(ContainerRequestContext requestContext) {
    final String tokenHeader = webRequest.getHeader(HttpHeaders.AUTHORIZATION);
    final String ws = requestContext.getUriInfo().getPath().split("/")[0];

    if (ws.equals("auth")) {
        return;
    }else if(tokenHeader == null){
        requestContext.abortWith(Response.status(401).entity(new NoAuth()).build());
    }

    final Token token = DBIManager.getJdbi().onDemand(TokenDao.class).getToken(tokenHeader);
    if (token == null) {
        requestContext.abortWith(Response.status(401).entity(new NoAuth()).build());
    } else {
        final AuthUser authUser = DBIManager.getJdbi().onDemand(UserDao.class).getUser(token.getUserId());
        requestContext.setSecurityContext(new SecurityUser(authUser));
    }
}
{% endhighlight %}

With this filter method when it is an `auth` request we just continue. If token header is empty then directly throw no auth error. Otherwise check token and if it is not valid throw no auth error. At last this is a valid token with a valid user so get the user info and set it as security context. Whenever you call `@Context SecurityContext securityContext;` in any resource you would be able to access to the current user. 

Update the `UserResource` and add `@RolesAllowed({"admin","mod"})` on top of it. With this, path `users/*` will be only accessible to mods and admins. When you try to access it with user role it will throw `403 Forbidden`.  

{% highlight java %}
@RolesAllowed({"admin","mod"})
@Path("users")
public class UserResource { ...
{% endhighlight %}

Also you can allow a sub path to user role just putting `@RolesAllowed({"user"})` on top of the method. Let's say we want to give user role the access of retrieving users. Adding this will only affect this specific method and now our default user can access to `/users` call directly. 

{% highlight java %}
@RolesAllowed("user")
@GET
@Produces(MediaType.APPLICATION_JSON)
@Path("")
public Response getUsers(){
    return Response.ok().entity(DBIManager.getJdbi().onDemand(UserDao.class).getUsers()).build();
}
{% endhighlight %}

At last I wanted to have a JDBI annotation so we can just inject it by using `@Dao UserDao dao` on methods or constructors rather than bunch of DbiManager etc. every time.
However Jersey 2.26 broke some api calls and apparently there is no `AbstractValueFactoryProvider` class anymore. Also no clue what is the replacement and can't ask on Github at this stage because project is archived and will be transferred to Eclipse Jakarta.

You can check all codes on [https://github.com/msdalp/jersey-jdbi-sample/tree/auth_filter-dao](https://github.com/msdalp/jersey-jdbi-sample/tree/auth_filter-dao)
I think I will have one or two more posts about Jersey and it will cover:

    * Using Redis injection for token validations and cache
    * Implement GenericExceptionMapper for catching exceptions and errors properly 
    * And maybe lastly implement web sockets on jersey. 

However I am not really keen about this and I might just move on to Spring Boot and have some examples on it. 
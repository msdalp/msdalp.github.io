---
layout: post
title:  "Rest Api with Spring Boot"
date:   2019-12-20 12:00:00
categories:
---

a real application?:
    - movie or tv show tracking 
    - user specific
    - recommendations
    - new episode mails? 
    - watch lists?
    - sharing lists? 


Step by step Spring Boot How to Rest:
* Have proper urls, responses, and http status codes 

## Follow Rest (Restful) Standards
* Use HTTP methods (verbs) `GET, POST, PUT, PATCH and DELETE`.
    * Do not use `GET: /articles/get or GET: /getArticles/`, use `GET: /articles/`  to get all articles
    * Do not use `POST: /articles/add or POST: /addArticles/`, use  `POST: /articles/`  to add a new article
    * Do not use `GET: /articles/get?id=12`, use  `GET: /articles/12`  to get a specific article
    * Do not use `DELETE: /articles/del?id=12`, use  `DELETE: /articles/12` to delete a specific article
    * Do not use `PUT: /articles/update`, use  `PUT: /articles/12`  to update a specific article

Why this is so important to follow? Isn't `/article/get` is more readable? For the second question it is unnecessary. Imagine  implementing functions for a string class and every function ends with string like `string.concatString`, `string.splitString` etc. For the first question this simple structure saves a lot of time of a client or the dev who is implementing the API. Syncing on the same page with other devs really hard and this makes the task a lot easier. 

How about the resources or tasks thats doesn't fit into this structure? Most of the services will fit into this style but only few will not when performing specific actions. Then just use a self explaining url with `POST`. Like `POST /users/forgotpassword/` or `POST /users/token/refresh`. I prefer using `POST` for these type of actions unless it can be specified with another method. 

This is called resource-oriented and each of these either represent a resource or a collection. Let's check gmail api as an example:

API service: gmail.googleapis.com (too much nesting though)
* A collection of users: users/*. Each user has the following resources.
    * A collection of messages: users/*/messages/*.
    * A collection of threads: users/*/threads/*.
    * A collection of labels: users/*/labels/*.
    * A collection of change history: users/*/history/*.
    * A resource representing the user profile: users/*/profile.
    * A resource representing user settings: users/*/settings.

## HTTP status codes 
Check on https://developer.mozilla.org/en-US/docs/Web/HTTP/Status

* Informational responses (100–199),
* Successful responses (200–299),
* Redirects (300–399),
* Client errors (400–499),
* and Server errors (500–599) 

Most common ones used for an API:
* 200 : Generic succesful response
* 201 : When a resource is updated or created return the new one
* 204 : Deleted a resource

* 400 : Input user provided is not valid or usable
* 401 : Need Authentication. Client must authenticate itself to get the requested response. 
* 403 : Forbidden. We know the client but client doesn't have access to this resource. 
* 404 : In API context either endpoint is not valid or  endpoint is valid but the resource itself does not exist. 
* 405 : Method is forbidden and can not be used for this resource. 

* 500 : Internal Server Error, specific server error or server doesn't know how to handle this error. 

| URL | GET | POST | DELETE | PUT |
|:---:|:---:|:---:|:---:|:---:|
| /articles | Status 200 OK with list of articles | Status 201 Created with the new article (add the actual id returned from db)  | x | x |
| /articles/12 | Status 200 OK with the article id = 12  |x |Status 204 No Content with no message when it is deleted  | Status 201 Created with the article updated |

## Response types
As it was stated before keep the responses like either a resource or a collection of resources. 
DO NOT wrap your responses into other models. 
`GET :/articles` -> `{ "articles" : [ {article1}, {article2} ] }`
`GET :/articles/12` -> `{ "data" : {article12}  }`
Most of the clients use libraries that parses the response to the actual object or models. This is counterintuitive generates unnecessary steps on the client to deal with sometimes causing errors. Prefer returning:
`GET :/articles` -> `[ {article1}, {article2} ] `
`GET :/articles/12` -> `{article12}`

Do not use merged responses for your errors and resources.
```
{
  "status" : 200,
  "error" : null,
  "data" : {article12}
}
```
```
{
  "status" : 400,
  "error" : "Mail can not be empty",
  "data" : null
}
```
This happens a lot when frameworks fail to provide returning generic response types or developer thinks it is easier to have a generic response that covers everything. Another horrible version of this returning status 200 to every call and make the client check if error is null for successful requests. Overall it has problems defined as above for wrapping responses and adds more complexity to the clients, mostly resulting in bad or faulty implementations.  Pick a generic error response for 400 and 500 responses with simple info and detailed message and always return that in case of error. 
```
{
  "error" : "Not Found",
  "message" : "User is not found"
}
```


[] rest standard
[] response
[] database
[] jwt-token
[] auto doc
[] migration
[] cache
[] ci
[] testing



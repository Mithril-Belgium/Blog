+++
title = "What is CORS and why it annoys us?"
date = "2020-12-04"
description = "CORS introduction"
tags = [
    "Web",
    "CORS"
]
+++

If you're a web developper, you probably encountered CORS issues a couple of times and, even if it's a well known issue in the web environment, it's not always deeply understood how and why it occurs.

Understanding what's CORS and the issues it can bring to your development is nonetheless important if you do not want to waste a lot of time in some silly situations...

## When can I encounter CORS issues?
Well everything is in the name... CORS: Cross Origin Ressource Sharing.  
In short: it's when you try to pick up some ressources from a different origin than the one you are asking the ressource!

It's easy to reproduce. Let's say I have an API exposing some data at the following URL: "mysuperapi.mithril.be/data". Obviously, this web API is not configured to handle the CORS...

Now, I'm going on any website (Google for example): I open the console and try to get these data using the fetch API:

> fetch("https://mysuperapi.mithril.be/data")

The request is failing and, in the network tab, we can see this: 

{{< image
src="/static/cors-introduction/request-failed.png"
alt="Request failed" >}}


Which is a little bit strange, because we are expecting an HTTP code as a status... the failed sound like something like a network issue... 

And we get this message in the console:
>Access to fetch at 'https://mysuperapi.mithril.be/data' from origin 'https://www.google.com' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource. If an opaque response serves your needs, set the request's mode to 'no-cors' to fetch the resource with CORS disabled.

It's because the request origin (www.google.com) is different from the response origin (mysuperapi.mithril.be).  
The origins differ when the domains are different or even when the ports are differents.

## Why is this happening?

To understand what's happening, let's check the raw request :
>GET https://mysuperapi.mithril.be/data HTTP/1.1  
>Host: mysuperapi.mithril.be   
[...]  
Origin: https://www.google.com  
Sec-Fetch-Site: cross-site  
Sec-Fetch-Mode: cors  
Sec-Fetch-Dest: empty  
[...] 


As you can see, the "Origin" header has been added to your request (whether you want it or not) by the browser. 

And here is the response:

>HTTP/1.1 200 OK  
Date: Thu, 03 Dec 2020 16:38:31 GMT  
Content-Type: application/json; charset=utf-8  
Server: Kestrel  
Transfer-Encoding: chunked  
>
>My Data

As you can see, the server is responding correctly with a 200 status and some data.   
And that's maybe the most frustrating part when you encounter this error for the first time and you don't understand it... The request seems correct, the API is reached (you could put a breakpoint if you have it locally), maybe a database is queried and some logs are generated... And the correct response is sent!

The important thing is to remember that between your web application and your API there are a few middlemen...

{{< image
src="/static/cors-introduction/middlemen.png"
alt="Middlemen" >}}

Each middleman (browser, server, proxy, ...) can add headers to your request and that's exactly what the browser is doing: it adds a header to your request stating the origin of it.

The API response does not contain anything to authorize the access to that specific origin so when the browser receive the response... it just drops it.
From the website point of view, the answer is never received and the request is considered as failed.

So, if you're trying to consume the same endpoint from a console application or Postman, it will work like a charm: **It's purely a browser mechanism**.

## How to prevent that ?

As the browser itself is rejecting the response, there is nothing you can do on the client side...  

>Well, actually, that's not totally true, there are some alternatives... For example, you can specify an option in your browser to ignore that or use an extension that will add what's missing to the response... But, since it will only work "on your machine", it's not a good idea... Except, maybe, if you're not responsible of the server but you already want to use these endpoints...

By deduction, we have to do something on the server/API side and the solution can be found in the error message we got from the console earlier:

>[...] No 'Access-Control-Allow-Origin' header is present on the requested resource. [...]

So, we will add such a header for all outcoming responses. 
You can either allow all origins:
>Access-Control-Allow-Origin: *

Or a specific one:
>Access-Control-Allow-Origin: https://www.google.com

When the browser checks the response and sees the header authorizing the origin, it will let the request reach the website.

In order to add this header to your response, you can either add it "manually". The following example shows how to add an anonymous middleware in ASP.NET Core (3.1) to achieve that: 

```csharp
app.Use((context, func) =>
{
    context.Response.Headers.Add("Access-Control-Allow-Origin", "*");
    return func();
});
```

Or you can also use the built-in middleware provided in the framework to handle that. We'd rather use this solution because it will handle more cases we didn't tackle in this post. 

```csharp
app.UseCors(builder => builder.AllowAnyOrigin());
```

But don't forget: since it's a browser mechanism, it can totally be avoided using a console application or a tool like Postman... Or even by configuring your browser... So the origin control is not a proper way to protect your API!

## A CORS issue can hide another one

In most cases, if you configure correctly your CORS middleware, you should get rid of most issues you could encounter.

But there can be another error in your application/server/middleman that could bring back a CORS issue... and thus, hiding the real error.

### Middleman (Proxy, API Manager, ...)

A first case that happened recently in a mission: Some of our API calls went through an API Manager (Azure API Manager in our case) but its behavior was to cut every request which took more than 20 seconds to execute and to send a "504 gateway timeout" response.

{{< image
src="/static/cors-introduction/middleman-issue.png"
alt="Middleman issue" >}}

Since it's not our API that is answering the request but the manager (which is a middleman), obviously, our CORS configuration in our API was not helpful and we got a CORS issue for some requests... (the API manager itself had some CORS mechanism but it was not enabled...).

In reality the real issue was (1) our request was lasting way too long and (2) the middleman timeout was shorter than our API timeout...  
But since the first visible error was a CORS issue, the real issues were not detected as soon as possible.

### Middlewares order

Your middlewares order is really important. If you put the middleware responsible for the authentication after the one that will give data to consumer, it will be useless.

The same issue can occur with CORS.  
If the authentication middleware is put before the CORS middleware and the authentication is failing your web API will answer with a "401" (which is totally legit) but since your request/response doesn't reach the CORS middleware, the expected headers won't be added.

The same happens if you put a middleware (let's say, for log purposes) that crashes before the CORS middleware... A "500" will be sent as a response but without the expected headers...

{{< image
src="/static/cors-introduction/middleware.png"
alt="Middlewares" >}}

So, once again, you will spend some time wondering why you get a CORS issue and checking where the configuration must be corrected, when, in the end, it was a good old 401/500 error you could have solved in just a few minutes...

## Summary
* CORS issues happen when the request origin is different from response origin.
* CORS is a browser mechanism : meaning, you will not have this issue with console or tool (Postman, Fiddler, CURL, ...).
* Since this is a client mechanism, it could easily be bypassed and should not be used as a way to protect your API.
* Solution is on the server side: by adding some headers to authorize the request origin.
* Any issue occuring before the CORS headers' addition could bring back a CORS issue (and by the way hide the real issue).
* CORS has other manifestations I didn't talk about in this post like preflight request.

## Documentation
* [MDN - CORS]("https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS"): Really exhaustive documentation about CORS.
* [ASP.NET Core CORS middleware]("https://docs.microsoft.com/en-us/aspnet/core/security/cors?view=aspnetcore-5.0"): The built-in middleware configuration (this is about ASP.NET COre 5 and I used the 3.1 so light differences are possible)
* [Fiddler]("https://www.telerik.com/fiddler"): The tool I'm using to inspect HTTP request/response.

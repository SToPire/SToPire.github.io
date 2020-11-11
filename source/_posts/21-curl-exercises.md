---
title: 21 curl exercises
date: 2020-11-11 11:10:04
categories: Tips
tags: [curl]
---
#### 21 curl exercises

Reference:https://jvns.ca/blog/2019/08/27/curl-exercises/

These exercises are about understanding how to make different kinds of HTTP requests with curl. They’re a little repetitive on purpose. They exercise basically everything I do with curl.

To keep it simple, we’re going to make a lot of our requests to the same website: [https://httpbin.org](https://httpbin.org/). httpbin is a service that accepts HTTP requests and then tells you what request you made.
<!-- more -->

1. Request [https://httpbin.org](https://httpbin.org/)
2. Request https://httpbin.org/anything. httpbin.org/anything will look at the request you made, parse it, and echo back to you what you requested. curl’s default is to make a GET request.
3. Make a POST request to https://httpbin.org/anything
4. Make a GET request to https://httpbin.org/anything, but this time add some query parameters (set `value=panda`).
5. Request google’s robots.txt file (www.google.com/robots.txt)
6. Make a GET request to https://httpbin.org/anything and set the header `User-Agent: elephant`.
7. Make a DELETE request to https://httpbin.org/anything
8. Request https://httpbin.org/anything and also get the response headers
9. Make a POST request to https://httpbin.org/anything with the JSON body `{"value": "panda"}`
10. Make the same POST request as the previous exercise, but set the Content-Type header to `application/json` (because POST requests need to have a content type that matches their body). Look at the `json` field in the response to see the difference from the previous one.
11. Make a GET request to https://httpbin.org/anything and set the header `Accept-Encoding: gzip` (what happens? why?)
12. Put a bunch of a JSON in a file and then make a POST request to https://httpbin.org/anything with the JSON in that file as the body
13. Make a request to https://httpbin.org/image and set the header ‘Accept: image/png’. Save the output to a PNG file and open the file in an image viewer. Try the same thing with different `Accept:` headers.
14. Make a PUT request to https://httpbin.org/anything
15. Request https://httpbin.org/image/jpeg, save it to a file, and open that file in your image editor.
16. Request [https://www.twitter.com](https://www.twitter.com/). You’ll get an empty response. Get curl to show you the response headers too, and try to figure out why the response was empty.
17. Make any request to https://httpbin.org/anything and just set some nonsense headers (like `panda: elephant`)
18. Request https://httpbin.org/status/404 and https://httpbin.org/status/200. Request them again and get curl to show the response headers.
19. Request https://httpbin.org/anything and set a username and password (with `-u username:password`)
20. Download the Twitter homepage ([https://twitter.com](https://twitter.com/)) in Spanish by setting the `Accept-Language: es-ES` header.
21. Omitted!



##### 1~5

```shell
curl httpbin.org
curl httpbin.org/anything
curl -X POST httpbin.org/anything
curl httpbin.org/anything -d value=panda # default:make a GET request
curl www.google.com/robots.txt
```

##### 6~10

```shell
curl httpbin.org/anything -H "User-Agent:elephant" 	# 6a
curl httpbin.org/anything -A "elephant" 			# 6b
curl httpbin.org/anything -X DELETE
curl -I httpbin.org/anything						# -I: show the reference header
curl -X POST httpbin.org/anything -d '{"value":"panda"}' 
curl -X POST httpbin.org/anything -d '{"value":"panda"}' -H "Content-Type:application/json"
```

##### 11~15

```shell
curl httpbin.org/anything -H "Accept-Encoding:gzip"
curl -X POST -d @test.json httpbin.org/anything -H "Content-Type:application/json"
curl httpbin.org/image -H "Accept:image/png" --output test.png
curl httpbin.org/anything -X PUT
curl httpbin.org/image/jpeg --output test.jpeg
```

##### 16~20

```shell
curl https://www.twitter.com -I # 303 Moved Permanently
curl httpbin.org/anything -H "panda:elephant"
curl httpbin.org/status/404 -I
curl httpbin.org/anything -u username:password 
curl https://twitter.com -H "Accept-Language:zh-CN" --output test.html
```


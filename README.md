# Traffic Server Debugging

1. Bring up nginx and Traffic Server containers:

    ```sh
    docker compose up
    ```

2. Generate a ~15MB file to use as the request body:

    ```sh
    base64 /dev/urandom | head -c 15000000 > body.txt
    ```

3. Test against `[nginx proxy] => [trafficserver 9.1.4] => [nginx server]`:

    ```sh
    for i in {1..20}; do curl -v -d @body.txt -H "Expect:" -X PUT "http://localhost:8914/?i=$i"; done
    ```

4. Test against `[nginx proxy] => [trafficserver 9.2.0] => [nginx server]`:

    ```sh
    for i in {1..20}; do curl -v -d @body.txt -H "Expect:" -X PUT "http://localhost:8920/?i=$i"; done
    ```

5. Test against `[nginx proxy] => [trafficserver 9.2.2] => [nginx server]`:

    ```sh
    for i in {1..20}; do curl -v -d @body.txt -H "Expect:" -X PUT "http://localhost:8922/?i=$i"; done
    ```

## Expected Behavior: Traffic Server 9.1.4

Under Traffic Server 9.1.4, all of the responses received back are the `413 Request Entity Too Large` error generated by the underlying nginx server. This is the expected behavior.

```
*   Trying 127.0.0.1:8914...
* Connected to localhost (127.0.0.1) port 8914 (#0)
> PUT /?i=1 HTTP/1.1
> Host: localhost:8914
> User-Agent: curl/8.1.2
> Accept: */*
> Content-Length: 15000000
> Content-Type: application/x-www-form-urlencoded
>
* We are completely uploaded and fine
< HTTP/1.1 413 Request Entity Too Large
< Server: nginx/1.25.2
< Date: Sun, 10 Sep 2023 23:14:51 GMT
< Content-Type: text/html
< Content-Length: 183
< Connection: keep-alive
< Age: 0
<
<html>
<head><title>413 Request Entity Too Large</title></head>
<body>
<center><h1>413 Request Entity Too Large</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>
* Connection #0 to host localhost left intact
*   Trying 127.0.0.1:8914...
* Connected to localhost (127.0.0.1) port 8914 (#0)
> PUT /?i=2 HTTP/1.1
> Host: localhost:8914
> User-Agent: curl/8.1.2
> Accept: */*
> Content-Length: 15000000
> Content-Type: application/x-www-form-urlencoded
>
* We are completely uploaded and fine
< HTTP/1.1 413 Request Entity Too Large
< Server: nginx/1.25.2
< Date: Sun, 10 Sep 2023 23:14:51 GMT
< Content-Type: text/html
< Content-Length: 183
< Connection: keep-alive
< Age: 0
<
<html>
<head><title>413 Request Entity Too Large</title></head>
<body>
<center><h1>413 Request Entity Too Large</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>
* Connection #0 to host localhost left intact
*   Trying 127.0.0.1:8914...
* Connected to localhost (127.0.0.1) port 8914 (#0)
> PUT /?i=3 HTTP/1.1
> Host: localhost:8914
> User-Agent: curl/8.1.2
> Accept: */*
> Content-Length: 15000000
> Content-Type: application/x-www-form-urlencoded
>
* We are completely uploaded and fine
< HTTP/1.1 413 Request Entity Too Large
< Server: nginx/1.25.2
< Date: Sun, 10 Sep 2023 23:14:51 GMT
< Content-Type: text/html
< Content-Length: 183
< Connection: keep-alive
< Age: 0
<
<html>
<head><title>413 Request Entity Too Large</title></head>
<body>
<center><h1>413 Request Entity Too Large</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>
* Connection #0 to host localhost left intact
*   Trying 127.0.0.1:8914...
* Connected to localhost (127.0.0.1) port 8914 (#0)
> PUT /?i=4 HTTP/1.1
> Host: localhost:8914
> User-Agent: curl/8.1.2
> Accept: */*
> Content-Length: 15000000
> Content-Type: application/x-www-form-urlencoded
>
* We are completely uploaded and fine
< HTTP/1.1 413 Request Entity Too Large
< Server: nginx/1.25.2
< Date: Sun, 10 Sep 2023 23:14:51 GMT
< Content-Type: text/html
< Content-Length: 183
< Connection: keep-alive
< Age: 0
<
<html>
<head><title>413 Request Entity Too Large</title></head>
<body>
<center><h1>413 Request Entity Too Large</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>
* Connection #0 to host localhost left intact
*   Trying 127.0.0.1:8914...
* Connected to localhost (127.0.0.1) port 8914 (#0)
> PUT /?i=5 HTTP/1.1
> Host: localhost:8914
> User-Agent: curl/8.1.2
> Accept: */*
> Content-Length: 15000000
> Content-Type: application/x-www-form-urlencoded
>
* We are completely uploaded and fine
< HTTP/1.1 413 Request Entity Too Large
< Server: nginx/1.25.2
< Date: Sun, 10 Sep 2023 23:14:52 GMT
< Content-Type: text/html
< Content-Length: 183
< Connection: keep-alive
< Age: 0
<
<html>
<head><title>413 Request Entity Too Large</title></head>
<body>
<center><h1>413 Request Entity Too Large</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>
* Connection #0 to host localhost left intact
```

## Unexpected Behavior: Traffic Server 9.2.x

In both versions of Traffic Server 9.2.x, you'll sometimes get the expected `413 Request Entity Too Large` error back, but then a decent number of `502 Bad Gateway` errors will also be generated by the `nginx proxy` layer. The `nginx proxy` layer reports errors like the following (seemingly indicating that the connection to Traffic Server has been closed unexpectedly):

```
writev() failed (104: Connection reset by peer) while sending request to upstream, client: 172.27.0.1, server: , request: "PUT /?i=3 HTTP/1.1", upstream: "http://172.27.0.5:8080/?i=3", host: "localhost:8922"
```

```
*   Trying 127.0.0.1:8922...
* Connected to localhost (127.0.0.1) port 8922 (#0)
> PUT /?i=1 HTTP/1.1
> Host: localhost:8922
> User-Agent: curl/8.1.2
> Accept: */*
> Content-Length: 15000000
> Content-Type: application/x-www-form-urlencoded
> 
* We are completely uploaded and fine
< HTTP/1.1 502 Bad Gateway
< Server: nginx/1.25.2
< Date: Sun, 10 Sep 2023 23:16:22 GMT
< Content-Type: text/html
< Content-Length: 157
< Connection: keep-alive
< 
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>
* Connection #0 to host localhost left intact
*   Trying 127.0.0.1:8922...
* Connected to localhost (127.0.0.1) port 8922 (#0)
> PUT /?i=2 HTTP/1.1
> Host: localhost:8922
> User-Agent: curl/8.1.2
> Accept: */*
> Content-Length: 15000000
> Content-Type: application/x-www-form-urlencoded
> 
* We are completely uploaded and fine
< HTTP/1.1 413 Request Entity Too Large
< Server: nginx/1.25.2
< Date: Sun, 10 Sep 2023 23:16:23 GMT
< Content-Type: text/html
< Content-Length: 183
< Connection: keep-alive
< Age: 0
< 
<html>
<head><title>413 Request Entity Too Large</title></head>
<body>
<center><h1>413 Request Entity Too Large</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>
* Connection #0 to host localhost left intact
*   Trying 127.0.0.1:8922...
* Connected to localhost (127.0.0.1) port 8922 (#0)
> PUT /?i=3 HTTP/1.1
> Host: localhost:8922
> User-Agent: curl/8.1.2
> Accept: */*
> Content-Length: 15000000
> Content-Type: application/x-www-form-urlencoded
> 
* We are completely uploaded and fine
< HTTP/1.1 502 Bad Gateway
< Server: nginx/1.25.2
< Date: Sun, 10 Sep 2023 23:16:23 GMT
< Content-Type: text/html
< Content-Length: 157
< Connection: keep-alive
< 
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>
* Connection #0 to host localhost left intact
*   Trying 127.0.0.1:8922...
* Connected to localhost (127.0.0.1) port 8922 (#0)
> PUT /?i=4 HTTP/1.1
> Host: localhost:8922
> User-Agent: curl/8.1.2
> Accept: */*
> Content-Length: 15000000
> Content-Type: application/x-www-form-urlencoded
> 
* We are completely uploaded and fine
< HTTP/1.1 502 Bad Gateway
< Server: nginx/1.25.2
< Date: Sun, 10 Sep 2023 23:16:23 GMT
< Content-Type: text/html
< Content-Length: 157
< Connection: keep-alive
< 
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>
* Connection #0 to host localhost left intact
*   Trying 127.0.0.1:8922...
* Connected to localhost (127.0.0.1) port 8922 (#0)
> PUT /?i=5 HTTP/1.1
> Host: localhost:8922
> User-Agent: curl/8.1.2
> Accept: */*
> Content-Length: 15000000
> Content-Type: application/x-www-form-urlencoded
> 
* We are completely uploaded and fine
< HTTP/1.1 502 Bad Gateway
< Server: nginx/1.25.2
< Date: Sun, 10 Sep 2023 23:16:23 GMT
< Content-Type: text/html
< Content-Length: 157
< Connection: keep-alive
< 
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>
* Connection #0 to host localhost left intact
```
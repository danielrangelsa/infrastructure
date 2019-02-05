# HAProxy - Load Balance

#### About HAProxy
HAProxy(High Availability Proxy) is an open source load balancer which can load balance any TCP service. It is particularly suited for HTTP load balancing as it supports session persistence and layer 7 processing.

With DigitalOcean Private Networking, HAProxy can be configured as a front-end to load balance two VPS through private network connectivity.

#### Prelude
We will be using three VPS (droplets) here:

Droplet 1 - Load Balancer
Hostname: haproxy
OS: Ubuntu Public IP: 1.1.1.1 Private IP: 10.0.0.100

Droplet 2 - Node 1
Hostname: lamp1
OS: LAMP on Ubuntu Private IP: 10.0.0.1

Droplet 2 - Node 2
Hostname: lamp2
OS: LAMP on Ubuntu Private IP: 10.0.0.2

#### Installing HAProxy

Use the apt-get command to install HAProxy.

`apt-get install haproxy`

We need to enable HAProxy to be started by the init script.

`nano /etc/default/haproxy`

Set the ENABLED option to 1

`ENABLED=1`

## ou 

`apt-get install haproxy && echo "ENABLED=1" >> /etc/default/haproxy && service haproxy status`

To check if this change is done properly execute the init script of HAProxy without any parameters. You should see the following.

```
root@haproxy:~# service haproxy
Usage: /etc/init.d/haproxy {start|stop|reload|restart|status}
```

#### Configuring HAProxy
We'll move the default configuration file and create our own one.

`mv /etc/haproxy/haproxy.cfg{,.original}`

Create and edit a new configuration file:

`nano /etc/haproxy/haproxy.cfg`

Let us begin by adding configuration block by block to this file:

```
global
    log 127.0.0.1 local0 notice
    maxconn 2000
    user haproxy
    group haproxy
```

The log directive mentions a syslog server to which log messages will be sent. On Ubuntu rsyslog is already installed and running but it doesn't listen on any IP address. We'll modify the config files of rsyslog later.

The maxconn directive specifies the number of concurrent connections on the frontend. The default value is 2000 and should be tuned according to your VPS' configuration.

The user and group directives changes the HAProxy process to the specified user/group. These shouldn't be changed.

```
defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    retries 3
    option redispatch
    timeout connect  5000
    timeout client  10000
    timeout server  10000
```

We're specifying default values in this section. The values to be modified are the various timeout directives. The connect option specifies the maximum time to wait for a connection attempt to a VPS to succeed.

The client and server timeouts apply when the client or server is expected to acknowledge or send data during the TCP process. HAProxy recommends setting the client and server timeouts to the same value.

The retries directive sets the number of retries to perform on a VPS after a connection failure.

The option redispatch enables session redistribution in case of connection failures. So session stickness is overriden if a VPS goes down.

```
listen appname 0.0.0.0:80
    mode http
    stats enable
    stats uri /haproxy?stats
    stats realm Strictly\ Private
    stats auth A_Username:YourPassword
    stats auth Another_User:passwd
    balance roundrobin
    option httpclose
    option forwardfor
    server lamp1 10.0.0.1:80 check
    server lamp2 10.0.0.2:80 check
```

This contains configuration for both the frontend and backend. We are configuring HAProxy to listen on port 80 for appname which is just a name for identifying an application. The stats directives enable the connection statistics page and protects it with HTTP Basic authentication using the credentials specified by the stats auth directive.

This page can viewed with the URL mentioned in stats uri so in this case, it is http://1.1.1.1/haproxy?stats; a demo of this page can be viewed here.

The balance directive specifies the load balancing algorithm to use. Options available are Round Robin (roundrobin), Static Round Robin (static-rr), Least Connections (leastconn), Source (source), URI (uri) and URL parameter (url_param).

Information about each algorithm can be obtained from the official documentation.

The server directive declares a backend server, the syntax is:

`server <name> <address>[:port] [param*]`

The name we mention here will appear in logs and alerts. There are many paratmeters supported by this directive and we'll be using the check and cookie parameters in this article. The check option enables health checks on the VPS otherwise, the VPS is
always considered available.

Once you're done configuring start the HAProxy service:

`service haproxy start`

#### Testing Load Balancing and Failover
To test this setup, create a PHP script on all your web servers (backend servers - LAMP1 and LAMP2 here).

`/var/www/file.php`

```
<?php
header('Content-Type: text/plain');
echo "Server IP: ".$_SERVER['SERVER_ADDR'];
echo "\nClient IP: ".$_SERVER['REMOTE_ADDR'];
echo "\nX-Forwarded-for: ".$_SERVER['HTTP_X_FORWARDED_FOR'];
?>
```

Now we will use curl and request this file multiple times.

```
> curl http://1.1.1.1/file.php

Server IP: 10.0.0.1
Client IP: 10.0.0.100
X-Forwarded-for: 117.213.X.X

> curl http://1.1.1.1/file.php

Server IP: 10.0.0.2
Client IP: 10.0.0.100
X-Forwarded-for: 117.213.X.X

> curl http://1.1.1.1/file.php

Server IP: 10.0.0.1
Client IP: 10.0.0.100
X-Forwarded-for: 117.213.X.X
```

Notice here how HAProxy alternatively toggled the connection between LAMP1 and LAMP2, this is how Round Robin works. The client IP we see here is the Private IP address of the load balancer and the X-Forwarded-For header is your IP.

To see how failover works, go to a web server and stop the service:

`lamp1@haproxy:~#service apache2 stop`

Send requests with curl again to see how things work.

#### Session Stickiness
If your web application serves dynamic content based on users' login sessions (which application doesn't), visitors will experience odd things due to continuous switching between VPS. Session stickiness ensures that a visitor sticks on to the VPS which served their first request. This is possible by tagging each backend server with a cookie.

We'll use the following PHP code to demonstrate how session stickiness works.

```
/var/www/session.php

<?php
header('Content-Type: text/plain');
session_start();
if(!isset($_SESSION['visit']))
{
        echo "This is the first time you're visiting this server";
        $_SESSION['visit'] = 0;
}
else
        echo "Your number of visits: ".$_SESSION['visit'];

$_SESSION['visit']++;

echo "\nServer IP: ".$_SERVER['SERVER_ADDR'];
echo "\nClient IP: ".$_SERVER['REMOTE_ADDR'];
echo "\nX-Forwarded-for: ".$_SERVER['HTTP_X_FORWARDED_FOR']."\n";
print_r($_COOKIE);
?>
```

This code creates a PHP session and displays the number of page views in a single session.

#### Cookie insert method
In this method, all responses from HAProxy to the client will contain a Set-Cookie: header with the name of a backend server as its cookie value. So going forward the client (web browser) will include this cookie with all its requests and HAProxy will forward the request to the right backend server based on the cookie value.

For this method, you will need to add the cookie directive and modify the server directives under listen

```
   cookie SRVNAME insert
   server lamp1 10.0.0.1:80 cookie S1 check
   server lamp2 10.0.0.2:80 cookie S2 check
```

This causes HAProxy to add a Set-Cookie: header with a cookie named SRVNAME having its value as S1 or S2 based on which backend answered the request. Once this is added restart the service:

`service haproxy restart`

and use curl to check how this works.

```
> curl -i http://1.1.1.1/session.php
HTTP/1.1 200 OK
Date: Tue, 24 Sep 2013 13:11:22 GMT
Server: Apache/2.2.22 (Ubuntu)
X-Powered-By: PHP/5.3.10-1ubuntu3.8
Set-Cookie: PHPSESSID=l9haakejnvnat7jtju64hmuab5; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 143
Connection: close
Content-Type: text/plain
Set-Cookie: SRVNAME=S1; path=/

This is the first time you're visiting this server

Server IP: 10.0.0.1
Client IP: 10.0.0.100
X-Forwarded-for: 117.213.X.X
Array
(
)
```

This is the first request we made and it was answered by LAMP1 as we can see from Set-Cookie: SRVNAME=S1; path=/. Now, to emulate what a web browser would do for the next request, we add these cookies to the request using the --cookie parameter of curl.

```
> curl -i http://1.1.1.1/session.php --cookie "PHPSESSID=l9haakejnvnat7jtju64hmuab5;SRVNAME=S1;"
HTTP/1.1 200 OK
Date: Tue, 24 Sep 2013 13:11:45 GMT
Server: Apache/2.2.22 (Ubuntu)
X-Powered-By: PHP/5.3.10-1ubuntu3.8
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 183
Connection: close
Content-Type: text/plain

Your number of visits: 1
Server IP: 10.0.0.1
Client IP: 10.0.0.100
X-Forwarded-for: 117.213.87.127
Array
(
    [PHPSESSID] => l9haakejnvnat7jtju64hmuab5
    [SRVNAME] => S1
)

> curl -i http://1.1.1.1/session.php --cookie "PHPSESSID=l9haakejnvnat7jtju64hmuab5;SRVNAME=S1;"
HTTP/1.1 200 OK
Date: Tue, 24 Sep 2013 13:11:45 GMT
Server: Apache/2.2.22 (Ubuntu)
X-Powered-By: PHP/5.3.10-1ubuntu3.8
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 183
Connection: close
Content-Type: text/plain

Your number of visits: 2
Server IP: 10.0.0.1
Client IP: 10.0.0.100
X-Forwarded-for: 117.213.87.127
Array
(
    [PHPSESSID] => l9haakejnvnat7jtju64hmuab5
    [SRVNAME] => S1
)
```

Both of these requests were served by LAMP1 and the session was properly maintained. This method is useful if you want stickiness for all files on the web server.

#### Cookie Prefix Method
On the other hand, if you want stickiness only for specific cookies or don't want to have a separate cookie for session stickiness, the prefix option is for you.

To use this method, use the following cookie directive:

`cookie PHPSESSID prefix`

The PHPSESSID can be replaced with your own cookie name. The server directive remains the same as the previous configuration.

Now let's see how this works.

```
> curl -i http://1.1.1.1/session.php
HTTP/1.1 200 OK
Date: Tue, 24 Sep 2013 13:36:27 GMT
Server: Apache/2.2.22 (Ubuntu)
X-Powered-By: PHP/5.3.10-1ubuntu3.8
Set-Cookie: PHPSESSID=S1~6l2pou1iqea4mnhenhkm787o56; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 143
Content-Type: text/plain

This is the first time you're visiting this server
Server IP: 10.0.0.1
Client IP: 10.0.0.100
X-Forwarded-for: 117.213.X.X
Array
(
)
```

Notice how the server cookie S1 is prefixed to the session cookie. Now, let's send two more requests with this cookie.

```
> curl -i http://1.1.1.1/session.php --cookie "PHPSESSID=S1~6l2pou1iqea4mnhenhkm787o56;"
HTTP/1.1 200 OK
Date: Tue, 24 Sep 2013 13:36:45 GMT
Server: Apache/2.2.22 (Ubuntu)
X-Powered-By: PHP/5.3.10-1ubuntu3.8
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 163
Content-Type: text/plain

Your number of visits: 1
Server IP: 10.0.0.1
Client IP: 10.0.0.100
X-Forwarded-for: 117.213.X.X
Array
(
    [PHPSESSID] => 6l2pou1iqea4mnhenhkm787o56
)

> curl -i http://1.1.1.1/session.php --cookie "PHPSESSID=S1~6l2pou1iqea4mnhenhkm787o56;"
HTTP/1.1 200 OK
Date: Tue, 24 Sep 2013 13:36:54 GMT
Server: Apache/2.2.22 (Ubuntu)
X-Powered-By: PHP/5.3.10-1ubuntu3.8
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 163
Content-Type: text/plain

Your number of visits: 2
Server IP: 10.0.0.1
Client IP: 10.0.0.100
X-Forwarded-for: 117.213.X.X
Array
(
    [PHPSESSID] => 6l2pou1iqea4mnhenhkm787o56
)
```

We can clearly see that both the requests were served by LAMP1 and the session is perfectly working.

#### Configure Logging for HAProxy
When we began configuring HAProxy, we added a line: log 127.0.0.1 local0 notice which sends syslog messages to the localhost IP address. But by default, rsyslog on Ubuntu doesn't listen on any address. So we have to make it do so.

Edit the config file of rsyslog.

`nano /etc/rsyslog.conf`

Add/Edit/Uncomment the following lines:

```
$ModLoad imudp
$UDPServerAddress 127.0.0.1
$UDPServerRun 514
````

Now rsyslog will work on UDP port 514 on address 127.0.0.1 but all HAProxy messages will go to 
/var/log/syslog so we have to separate them.

Create a rule for HAProxy logs.

`nano /etc/rsyslog.d/haproxy.conf`

Add the following line to it.

`if ($programname == 'haproxy') then -/var/log/haproxy.log`

Now restart the rsyslog service:

`service rsyslog restart`

This writes all HAProxy messages and access logs to /var/log/haproxy.log.

#### Keepalives in HAProxy
Under the listen directive, we used option httpclose which adds a Connection: close header. This tells the client (web browser) to close a connection after a response is received.

If you want to enable keep-alives on HAProxy, replace the option httpclose line with:

```
option http-server-close
timeout http-keep-alive 3000
```

Set the keep-alive timeout wisely so that a few connections don't drain all the resources of the load balancer.

##### Testing Keepalives
Keepalives can be tested using curl by sending multiple requests at the same time. Unnecessary output will be omitted in the following example:
```
> curl -v http://1.1.1.1/index.html http://1.1.1.1/index.html
* About to connect() to 1.1.1.1 port 80 (#0)
*   Trying 1.1.1.1... connected
> GET /index.html HTTP/1.1
> User-Agent: curl/7.23.1 (x86_64-pc-win32) libcurl/7.23.1 OpenSSL/0.9.8r zlib/1.2.5
> Host: 1.1.1.1
> Accept: */*
>
......[Output omitted].........
* Connection #0 to host 1.1.1.1 left intact
* Re-using existing connection! (#0) with host 1.1.1.1
* Connected to 1.1.1.1 (1.1.1.1) port 80 (#0)
> GET /index.html HTTP/1.1
> User-Agent: curl/7.23.1 (x86_64-pc-win32) libcurl/7.23.1 OpenSSL/0.9.8r zlib/1.2.5
> Host: 1.1.1.1
> Accept: */*
>
.......[Output Omitted].........
* Connection #0 to host 1.1.1.1 left intact
* Closing connection #0
```

In this output, we have to look for the line: Re-using existing connection! (#0) with host 1.1.1.1, which indicates that curl used the same connection to make subsequent requests.

PS.: Este conteúdo será revisado.

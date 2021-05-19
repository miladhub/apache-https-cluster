Apache with HTTPS cluster example
===

This project represents a simple example of how to expose a service on Apache using HTTPS using more back-end nodes.

These instructions were tested on MacOS. The app behind Apache is just a demo server that dumps all HTTP headers
on the console. The app-specific configuration is contained in fil `httpd-ssl.conf`:
```
<Proxy "balancer://mycluster">
    BalancerMember http://host.docker.internal:8000 route=node1
    BalancerMember http://host.docker.internal:8100 route=node2
    ...
</Proxy>
...
<Location /Downloads>
  ProxyPass balancer://mycluster/Downloads
  ProxyPassReverse balancer://mycluster/Downloads
</Location>
```

In this example, I'll be running two instances of the Python HTTP server `http.server` from my home folder, listening
on ports 8000 and 8100, each of which exposing my home folder contents, for example, <http://localhost:8000/Downloads>.

To point at your app, you need to change the nodes, their ports, and the `/Downloads` path by pointing at your app.
Also, if you use application servers like Tomcat or Wildfly, you can also switch to the AJP protocol instead of HTTP,
for example:
```
<Proxy "balancer://mycluster">
    BalancerMember ajp://host.docker.internal:8009 route=node1
    BalancerMember ajp://host.docker.internal:8109 route=node2
    ...
</Proxy>
...
<Location /app>
  ProxyPass balancer://mycluster/app
  ProxyPassReverse balancer://mycluster/app
</Location>
```

where `/app` is the context root of your application.

## Creating the server private key and SSL certificate

Create the private key `localhost.key` and the self-signed SSL certificate `localhost.crt` using
[mkcert](https://github.com/FiloSottile/mkcert):
```
mkcert --install
mkcert localhost
openssl x509 -outform der -in localhost.pem -out localhost.crt
openssl pkey -in localhost-key.pem -out localhost.key
```

## Setting up Apache with HTTPS

```
docker run -dit --name apache-cluster -p 443:443 httpd:2.4
docker cp httpd.conf apache-cluster:/usr/local/apache2/conf
docker cp httpd-ssl.conf apache-cluster:/usr/local/apache2/conf/extra
docker cp localhost.pem apache-cluster:/usr/local/apache2/conf/server.crt
docker cp localhost.key apache-cluster:/usr/local/apache2/conf/server.key
docker exec apache-cluster apachectl restart
```

# Installing the dummy app

Issue the following command from two terminals (you'll need Python 3 installed):
```
cd ~
python3 -m http.server 8000
```
```
cd ~
python3 -m http.server 8100
```

If you now navigate to the app at <https://localhost/balancer-manager>, you'll see the two cluster members registered,
but with all indicators set to zero since no request arrived yet.

Worker URL                       | Route | RouteRedir | Factor | Set | Status  | Elected | Busy | Load | To | From
-------------------------------- | ----- | ---------- | ------ | --- | ------- | ------- | ---- | ---- | -- | ----
http://host.docker.internal:8000 | node1 |            | 1.00   | 0   | Init Ok | 0       | 0    | 0    | 0  | 0
http://host.docker.internal:8100 | node2 |            | 1.00   | 0   | Init Ok | 0       | 0    | 0    | 0  | 0

If you now open the URL <https://localhost/Downloads> from two different browsers, you'll see those numbers changing for both
nodes, indicating that requests are being served by them.

Worker URL                       | Route | RouteRedir | Factor | Set | Status  | Elected | Busy | Load | To   | From
-------------------------------- | ----- | ---------- | ------ | --- | ------- | ------- | ---- | ---- | --   | ----
http://host.docker.internal:8000 | node1 |            | 1.00   | 0   | Init Ok | 1       | 0    | 0    | 523  | 420
http://host.docker.internal:8100 | node2 |            | 1.00   | 0   | Init Ok | 1       | 0    | 0    | 531  | 420

## References

* <https://devcenter.heroku.com/articles/ssl-certificate-self>
* <https://hub.docker.com/_/httpd>
* <https://docs.docker.com/docker-for-mac/networking/> (`host.docker.internal`)
* https://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html


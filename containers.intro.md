# Containers Intro

## Docker Desktop
To run Docker on Windows and MacOS, you can download [Docker Desktop](https://www.docker.com/products/docker-desktop). Docker on 
Windows integrates well with the WSL and you can manage all aspects of Docker via the command line on Linux.
If you're installing Docker on a Ubuntu Virtual Machine, you can do so via the `docker.io` package (`sudo apt install docker.io`)

## Docker Hub
You can search for container images via the website hub.docker.com. **Caveat Emptor**: you can find images built by the offical development team or company or community contributed unofficial images. 
To download a container image you issue the command `docker pull imagename`. The exact command is provided via the container image page.


### A note on the nomenclature for the images
The nomenclature for a container image can accompanied by a tag. For example, these are currently valid node tags:
````
14-alpine3.12, 14.17-alpine3.12, 14.17.0-alpine3.12, fermium-alpine3.12, lts-alpine3.12
14-alpine3.13, 14.17-alpine3.13, 14.17.0-alpine3.13, fermium-alpine3.13, lts-alpine3.13
14-buster
```` 

Where 14 is the node version, and Alpine, buster and fermium-alpine are the Linux distributions on which the node container image was built upon.

## Serving the Node API
The Node API provided by Ing. Miguel Huchim will serve again our learning purposes. You download it the usual way via `git clone` but we have to create a Dockerfile for the construction of a new container for the API:

```Dockerfile
FROM node:latest
WORKDIR /var/www/html
COPY . ./
RUN npm install
EXPOSE 3000
CMD npm run dev
```

In `FROM node:latest` we announce that we're basing our container on the node.

`WORKDIR /var/www/html` creates and changes to the directory `/var/www/html`

`COPY . ./` copies whatever is on our current directory to the container's current directory

`RUN npm install` runs the command `npm install` in the container

`EXPOSE 3000` announces that the container can serve content via the TCP port 3000 (This is specified in the the index.js file). And finally

`CMD npm run dev` starts the development web server in the container

If you save this file with name `Dockerfile-user-api-demo` in the folder created by the repo cloning, we can actually build our container with this command: 

```bash
docker build . -t user-api-demo:1.0 -f Dockerfile-user-api-demo
[+] Building 7.7s (9/9) FINISHED
 => [internal] load build definition from Dockerfile-user-api-demo                                                                                            0.0s
 => => transferring dockerfile: 162B                                                                                                                         0.0s
 => [internal] load .dockerignore                                                                                                                           0.0s
 => => transferring context: 2B                                                                                                                             0.0s
 => [internal] load metadata for docker.io/library/node:latest                                                                                              0.0s
 => [1/4] FROM docker.io/library/node:latest                                                                                                                0.0s
 => [internal] load build context                                                                                                                           0.0s
 => => transferring context: 195.42kB                                                                                                                       0.0s
 => CACHED [2/4] WORKDIR /var/www/html                                                                                                                      0.0s
 => [3/4] COPY . ./                                                                                                                                         0.0s
 => [4/4] RUN npm install                                                                                                                                   7.0s
 => exporting to image                                                                                                                                      0.4s
 => => exporting layers                                                                                                                                     0.4s
 => => writing image ha256:8188450ff430ab0baad8dac99562f73242bcbd2715dc57e04561a562e009f87b                                                                 0.0s
 => => naming to docker.io/library/user-api-demo:1.0
 ```

To serve it using an "interactive terminal" (-it) and mapping the exposed port 3000 from the container to our computer's port 3000 (-p 3000:3000) we use

```bash
docker run -it --rm -p 3000:3000 user-api-demo:1.0

> user-api-demo@1.0.0 dev
> cross-env DEBUG=app:* nodemon index

[nodemon] 2.0.7
[nodemon] to restart at any time, enter `rs`
[nodemon] watching path(s): *.*
[nodemon] watching extensions: js,mjs,json
[nodemon] starting `node index index.js`
Listening at http://localhost:3000
```

On our computer we can use the API as intented
```bash
curl -d '{ "email": "foobar@gmail.com", "first_name": "foo", "last_name": "bar", "company": "temporal" }' -H "Content-Type: application/json" -X POST http://localhost:3000/api/user
``` 

To exit the interactive terminal, you can use CTRL-C

## Using Apache HTTPd server as a reverse proxy
We'll use this [container image](https://hub.docker.com/_/httpd) for Apache.
On the container's image page, advise is given on how to customize the server configuration file:
```
To customize the configuration of the httpd server, first obtain the upstream default configuration from the container:

$ docker run --rm httpd:2.4 cat /usr/local/apache2/conf/httpd.conf > my-httpd.conf
```

We do just that and we trim the config file to just like the one in [conf/my-httpd.conf].
Basically, we just "added" 4 lines. In the modules section we uncommented

```
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```
And after the logs, we added:
```
ProxyPass / http://apimiguel:3000/
ProxyPassReverse / http://apimiguel:3000/
```
That assumes that there's a host named apimiguel serving content via the HTTP protocol at port 3000.
We create the container image using [this dockerfile](conf/Dockerfile_httpd_reverse_proxy): 
```dockerfile
FROM httpd:latest
COPY my-httpd.conf /usr/local/apache2/conf/httpd.conf
EXPOSE 80
CMD httpd -D FOREGROUND
```

To actually create the image:
```bash
docker build . -t httpd_inverse_proxy:1.0 -f Dockerfile_httpd_reverse_proxy
[+] Building 0.3s (7/7) FINISHED
 => [internal] load build definition from Dockerfile_httpd_reverse_proxy                                                            0.1s
 => => transferring dockerfile: 169B                                                                                               0.0s
 => [internal] load .dockerignore                                                                                                 0.0s
 => => transferring context: 2B                                                                                                   0.0s
 => [internal] load metadata for docker.io/library/httpd:latest                                                                   0.0s
 => [internal] load build context                                                                                                 0.0s
 => => transferring context: 2.75kB                                                                                               0.0s
 => CACHED [1/2] FROM docker.io/library/httpd:latest                                                                              0.0s
 => [2/2] COPY my-httpd.conf /usr/local/apache2/conf/httpd.conf                                                                   0.1s
 => exporting to image                                                                                                            0.0s
 => => exporting layers                                                                                                           0.0s
 => => writing image sha256:95cb260d196056e246644bf09df20bed46f46597e24ca65eae807b1c8b9595ee                                      0.0s
 => => naming to docker.io/library/httpd_inverse_proxy:1.0
 ```

To create the container, we depend on the node container having a name of apimiguel, so if we havent already, stop the node container and start it again with this command
```bash
docker run -it --rm -p 3000:3000 --name apimiguel user-api-demo:1.0
```

Now the inverse proxy:
```bash
docker run -it --rm -p 80:80 --link apimiguel httpd_inverse_proxy:1.0
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.17.0.3. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.17.0.3. Set the 'ServerName' directive globally to suppress this message
[Fri Jun 11 16:26:17.608804 2021] [mpm_event:notice] [pid 8:tid 140530813867136] AH00489: Apache/2.4.48 (Unix) configured -- resuming normal operations
[Fri Jun 11 16:26:17.608958 2021] [core:notice] [pid 8:tid 140530813867136] AH00094: Command line: 'httpd -D FOREGROUND'
```

We should be able to use the API in the port 80
```
curl -d '{ "email": "foobar@gmail.com", "first_name": "foo", "last_name": "bar", "company": "temporal" }' -H "Content-Type: application/json" -X POST http://localhost/api/user
{"message":"User created","data":{"email":"foobar@gmail.com","first_name":"foo","last_name":"bar","company":"temporal","id":1}}
```

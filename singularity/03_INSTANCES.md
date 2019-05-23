## Singularity Instances

Up to now all of our examples have run Singularity containers in the foreground.  But what if you want to run a service like a web server or a database in a Singularity container in the background?

#### lolcow (useless) example
In Singularity v3.0, you can use the [`instance` command group](https://www.sylabs.io/guides/3.0/user-guide/quick_start.html#interact-with-images) to start and control container instances that run in the background.  To demonstrate, let's start an instance of our `lolcow.sif` container running in the background.

```
$ singularity instance start lolcow.simg cow1
```

We can use the `instance list` command to show the instances that are currently running.

```
$ singularity instance.list
DAEMON NAME      PID      CONTAINER IMAGE
cow1             10794    /home/eduardo/lolcow.sif
```

We can connect to running instances using the `instance://` URI like so:

```
$ singularity shell instance://cow1
Singularity: Invoking an interactive shell within container...

Singularity lolcow.sif:~> ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
eduardo         1     0  0 19:05 ?        00:00:00 singularity-instance: eduardo [cow1]
eduardo         3     0  0 19:06 pts/0    00:00:00 /bin/bash --norc
eduardo         4     3  0 19:06 pts/0    00:00:00 ps -ef

Singularity lolcow.sif:~> exit
```

Note that we've entered a new PID namespace, so that the `singularity-instance` process has PID number 1.

You can start multiple instances running in the background, as long as you give them unique names.

```
$ singularity instance start lolcow.sif cow2

$ singularity instance start lolcow.sif cow3

$ singularity instance.list
DAEMON NAME      PID      CONTAINER IMAGE
cow1             10794    /home/eduardo/lolcow.sif
cow2             10855    /home/eduardo/lolcow.sif
cow3             10885    /home/eduardo/lolcow.sif
```

You can stop individual instances using their unique names or stop all instances with the `--all` option.

```
$ singularity instance stop cow1
Stopping cow1 instance of /home/eduardo/lolcow.sif (PID=10794)

$ singularity instance stop --all
Stopping cow2 instance of /home/eduardo/lolcow.sif (PID=10855)
Stopping cow3 instance of /home/eduardo/lolcow.sif (PID=10885)
```

#### nginx (useful) example

These examples are not very useful because `lolcow.sif` doesn't run any services.  Let's extend the example to something useful by running a local nginx web server in the background.  This command will download the official nginx image from Docker Hub and start it in a background instance called "web".  (The commands need to be executed as root so that nginx can run with the privileges it needs.)

```
$ sudo singularity instance.start docker://nginx web
Docker image path: index.docker.io/library/nginx:latest
Cache folder set to /root/.singularity/docker
[3/3] |===================================| 100.0%
Creating container runtime...

$ sudo singularity instance.list
DAEMON NAME      PID      CONTAINER IMAGE
web              15379    /tmp/.singularity-runtime.MBzI4Hus/nginx
```

Now to start nginx running in the instance called web.

```
$ sudo singularity exec instance://web nginx
```

Now we have an nginx web server running on our localhost.  We can verify that it is running with `curl`.

```
$ curl localhost
127.0.0.1 - - [02/Nov/2017:19:20:39 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.52.1" "-"
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

When finished, don't forget to stop all running instances like so:

```
$ sudo singularity instance stop --all
```

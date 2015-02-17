# swarm
misc discovery work with swarm

# putting swarm on a host
```
$ apt-get install -y golang git
$ mkdir ~/gocode; export GOPATH=~/gocode
$ go get -u github.com/docker/swarm
$ PATH=$PATH:$GOPATH/bin
```

# before creating a swarm!

Docker must be running on a tcp port that will be accessible from the swarm mamanger.
This can be done by starting the docker service and binding it to
the tcp, something like this:

```
service docker stop
echo 'export DOCKER_OPTS="-H=0.0.0.0:2375 -H=unix:///var/run/docker.sock"' >> /etc/default/docker
service docker start
```

Now the docker engine should be accepting insecure connections on tcp port 2375 as well
as the unix domain socket.  Now that docker is listening, we can create a simple swarm:

# once installed, create a cluster id (once)
```
$ swarm create
b44af7c37a3ab69fb72ba29d2b2e72ad
```

# then, for each swarm node:
```
swarm join --addr=108.61.241.227:2375 token://b44af7c37a3ab69fb72ba29d2b2e72ad
swarm join --addr=108.61.222.182:2375 token://b44af7c37a3ab69fb72ba29d2b2e72ad
...
```

# then, for the manager:
```
swarm manage -H tcp://104.238.146.180:2375 token://b44af7c37a3ab69fb72ba29d2b2e72ad
```

# at this point you have a simple swarm up.
To create a docker container, you run a docker client and point it to the
swarm manager.  In this case:

```
docker -H tcp://104.238.146.180:2375 ps
```

```
swarm list token://b44af7c37a3ab69fb72ba29d2b2e72ad
```


# for starting docker under tls:
you don't really want open docker engines, so, use tls instead. in this
repo there are several scripts that will create self signed certs for
use.
make the signing certificate, and all of the server/client certificates at
the same machine, that is:
```
make_signing_cert # this creates ~/ssl-cert directory
make_cert -i 104.238.146.81 -v -d ~/ssl-cert/c1.tacodata.com c1.tacodata.com # creates ~/ssl-cert/c1.tacodata.com
make_cert -i 104.238.147.26 -v -d ~/ssl-cert/c2.tacodata.com c2.tacodata.com # creates ...c2.tacodata.com
make_cert -i 104.238.146.194 -v -d ~/ssl-cert/c3.tacodata.com c3.tacodata.com # yup, creates c3.
make_swarm_cert -i 104.238.146.180 -v -d ~/ssl-cert/w1.tacodata.com w1.tacodata.com
```

or, more simply:

```
for i in w1 c1 c2 c3 c4; do
    IP=`dig +short $i.tacodata.com`;
    echo "$i $IP";
    rm -rf ~/ssl-cert/$i.tacodata.com;
    make_swarm_cert -i $IP -d ~/ssl-cert/$i.tacodata.com $i.tacodata.com;
done
```

the first script makes a signing certificate, for self signing.  the second 3 scripts make a certificate for
each of the 3 engines involved with this swarm.  the ip address and the fqdn are part of the
certificate. the last script makes a certificate for the swarm manager.  it is different because
it is both a client and a server.
be sure to save your ssl-cert directory, it is used to sign new certs as needed.
tar up the entire ssl-cert tree and unpack on the other hosts. then, for each host:

at this point you can manually fire up the docker daemon on each docker host.
```
cd ~/ssl-cert/c1.tacodata.com
docker -d --tlsverify --tlscacert=ca.pem --tlscert=server-cert.pem --tlskey=server-key.pem   -H=0.0.0.0:2376
```
or, you can add these linees to /etc/default/docker
```
XD=/root/ssl-cert/c1.tacodata.com
export DOCKER_OPTS="--tlsverify --tlscacert=$XD/ca.pem --tlscert=$XD/server-cert.pem --tlskey=$XD/server-key.pem   -H=0.0.0.0:2376 -H=unix:///var/run/docker.sock"
```
then do a service start (or restart) like:
service docker start

before trying to connect from a docker client you will need your client certificate stuff which
does two things, first, it creates a .docker directory and deposits the stuff necessary for
a client certificate.  second, it creates a /etc/profile.d/docker.sh file which is sourced (the next
time you log in) to set up the client's access.

```
make_docker_ssl -d ~ssl-cert/c1.tacodata.com
```

the docker client will connect with no problems to the unix domain socket, if you want to connect
to the ip addresses:

```
docker --tlsverify -H tcp://104.238.146.81:2376 ps
or
docker --tlsverify -H tcp://127.0.0.1:2376 ps
or
docker --tlsverify -H tcp://c1.tacodata.com:2376 ps

```

also note the above script creates the file /etc/profile.d/docker.sh, so when you
log in the next time you will have the environment variables:

```
export DOCKER_HOST=tcp://:2376
export DOCKER_TLS_VERIFY=1
```

defined, which forces your local connection via tcp/tls.
This file can be removed to connect via the unix domain socket locally.

# summary
once you build the self signed certs and distribute the ssl-cert directory
to all docker hosts you will modify each docker host's /etc/default/docker file to
make sure the docker server starts up in secure mode.  then you will run swarm join
on each of those hosts. finally you will copy the ssl-cert directory to the swarm manage
machine, cd to the ssl-cert/w1.tacodata.com (or whatever the manage name is) and
then start swarm manage.  This then creates a manager which connects to all of the
join machines via their secure tls channel.  this also creates a docker server api
that we connect to with plain old docker commands.  We can get fancier with constraints
about each of the docker hosts such that commands like docker run can be 'placed' on the
right docker host.

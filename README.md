# swarm
misc discovery work with swarm

# putting swarm on a host
```
$ apt-get install -y golang git
$ mkdir ~/gocode; export GOPATH=~/gocode
$ go get -u github.com/docker/swarm
$ PATH=$PATH:$GOPATH/bin
```
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

# for starting docker under tls:
make the signing certificate, and all of the server/client certificates at
the same machine, that is:
```
make_signing_cert # this creates ~/ssl-cert directory
make_cert -i 104.238.146.81 -v -d ~/ssl-cert/c1.tacodata.com c1.tacodata.com # creates ~/ssl-cert/c1.tacodata.com
make_cert -i 104.238.147.26 -v -d ~/ssl-cert/c2.tacodata.com c2.tacodata.com # creates ...c2.tacodata.com
make_cert -i 104.238.146.194 -v -d ~/ssl-cert/c3.tacodata.com c3.tacodata.com # yup, creates c3.
```

be sure to save your ssl-cert directory, it is used to sign new certs as needed.
tar up the entire ssl-cert tree and unpack on the other hosts. then, for each host:

at this point you can manually fire up the docker daemon.
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

before trying to connect from a docker client you will need your client certificate stuff:
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

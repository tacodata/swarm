# swarm
misc discovery work with swarm
# putting swarm on a host
apt-get install -y golang git
mkdir ~/gocode; export GOPATH=~/gocode
go get -u github.com/docker/swarm
PATH=$PATH:$GOPATH/bin
# once installed, create a cluster id (once)
swarm create
b44af7c37a3ab69fb72ba29d2b2e72ad

then, for each swarm node:
swarm join --addr=108.61.241.227:2375 token://b44af7c37a3ab69fb72ba29d2b2e72ad
swarm join --addr=108.61.222.182:2375 token://b44af7c37a3ab69fb72ba29d2b2e72ad

then, for the manager:
swarm manage -H tcp://104.238.146.180:2375 token://b44af7c37a3ab69fb72ba29d2b2e72ad

for starting docker under tls:
make sure you've copied ssl-cert from the repo, then make you a certificate, like:
make_cert -d ~/ssl-cert/c1.tacodata.com c1.tacodata.com
then fire up the docker daemon
cd ~/ssl-cert/c1.tacodata.com
docker -d --tlsverify --tlscacert=ca.pem --tlscert=server-cert.pem --tlskey=server-key.pem   -H=0.0.0.0:2376

for running a client, make sure you run this once on your new host:
make_docker_ssl -d ~ssl-cert/c1.tacodata.com

that does two things, first, copy the appropriate client data into a new directory called ~/.docker, then
a /etc/profile.d/docker.sh is created setting DOCKER_HOST and DOCKER_TLS_VERIFY

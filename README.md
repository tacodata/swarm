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

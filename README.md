# fleet
experiments with fleet and etcd
# fleet requires a etcd 'core'
To set up etcd to run across many machines (peer with each other) you
have to create a secure ssl environment.  I did that using a script I created
in a different repo (which I copied over here) called make_swarm_cert. I created 5
machines, running basic ubuntu, they are named c1, c2, c3, c4 and w1.  The domain
name is tacodata.com.  Here is a script I ran on w1 to create the certificates
necessary for peering via ssl:

```
for i in w1 c1 c2 c3 c4; do
    IP=`dig +short $i.tacodata.com`;
    echo "$i $IP";
    rm -rf ~/ssl-cert/$i.tacodata.com;
    make_swarm_cert -i $IP -d ~/ssl-cert/$i.tacodata.com $i.tacodata.com;
done
```

So, now that all of the certificates are in the root directory's ssl-cert directory, I tar them off
and copy to each machine:

```
cd ~/root;
tar czvf /tmp/s.tar.gz ssl-cert
```

Distribute the cert file to each machine and unpack it in it's root directory.

I am assuming that docker has already been installed on each of the hosts, and is up and running.
To start up etcd on each of the hosts:

```
# the current machine
MACHINE=c1
# all machines in the etcd group
AMACHINE="c1 c2 c3"
# the domain each machine (host) is at
DOMAIN=tacodata.com
# our external ip
MY_IP=$(dig +short $MACHINE.$DOMAIN)
# all ip addresses for all machines in the group
AIP=$(for i in $AMACHINE; do dig +short $i.$DOMAIN; done)
# all ip addresses except for our current machine
OIP=$(for i in $AMACHINE; do if [ $MACHINE != $i ] ; then dig +short $i.$DOMAIN; fi; done)
PEER_PORT=7001
OIPP=$(for i in $OIP; do echo $i:$PEER_PORT; done)
OIPPC=$(echo $OIPP | sed 's/ /,/g')
LOCAL_PORT=4001
CERT_DIR=/root/ssl-cert/$MACHINE.$DOMAIN
ETCD_IMAGE=quay.io/philipsoutham/etcd
docker run \
    -d \
    -p $MY_IP:$LOCAL_PORT:$LOCAL_PORT -p $MY_IP:$PEER_PORT:$PEER_PORT \
    -v $CERT_DIR:/cert \
    $ETCD_IMAGE \
    --name $MACHINE.$DOMAIN \
    --peer-cert-file=/cert/server-cert.pem --peer-key-file=/cert/server-key.pem --peer-ca-file=/cert/ca.pem \
    --peer-addr=$MY_IP:$PEER_PORT \
    --peers=$OIPPC
```

The script above is tedious, there is a script called run_etcd which executes in in the
bin directory:

```
run_etcd -h
Usage: run_etcd [ -h -v -p PEER_PORT -l LOCAL_PORT -c CERT_DIR] -m MACHINE -g all etcd machine list -d domain
  -h help message (this)
  -v verbose
  -m MACHINE (this is the hostname part of a fqdn)
  -g list all machines in the group (master should only list itself)
  -d DOMAIN (the rest of the fqdn)
  -p PEER_PORT, default is : 7001
  -l LOCAL CLIENT PORT, default is : 4001
  -c cert dir, if omitted will be constructed using /root/ssl-cert/MACHINE.DOMAIN
```

so, for the above example, I would run (on the appropriate hosts):

```
c1: run_etcd -m c1 -g "c1" -d tacodata.com
c2: run_etcd -m c2 -g "c1 c2 c3" -d tacodata.com
c3: run_etcd -m c3 -g "c1 c2 c3" -d tacodata.com
```

for a generic way to run it on any machine:

```
master: run_etcd -m `hostname -s` -g `hostname -s` -d `hostname -d`
members: run_etcd -m `hostname -s` -g `c1 c2 c3 (all member names)` -d `hostname -d`
```

Note: pick any one of them to run with the -g switch with only their hostname.  The initial 'master' node
doesn't have any peers to contact, so, it must be started without any peers.

to pick up the leader and all followers, from any node, run:

```
# curl -s -L 127.0.0.1:4001/v2/stats/leader | python -mjson.tool
```

Which produces output like:

```
{
    "followers": {
        "c2.tacodata.com": {
            "counts": {
                "fail": 0,
                "success": 4331
            },
            "latency": {
                "average": 1.327962363195568,
                "current": 1.168508,
                "maximum": 44.635728,
                "minimum": 0.915003,
                "standardDeviation": 0.718127472886556
            }
        },
        "c3.tacodata.com": {
            "counts": {
                "fail": 0,
                "success": 4268
            },
            "latency": {
                "average": 1.1678028870665418,
                "current": 5.817836,
                "maximum": 26.690555,
                "minimum": 0.786489,
                "standardDeviation": 0.5571811508915141
            }
        }
    },
    "leader": "c1.tacodata.com"
}
```

To set a value (from a etcd local host):

```
# curl -L -s http://127.0.0.1:4001/v2/keys/message -XPUT -d value="Hello Snoop Dog"  | python -mjson.tool
{
    "action": "set",
    "node": {
        "createdIndex": 5,
        "key": "/message",
        "modifiedIndex": 5,
        "value": "Hello Snoop Dog"
    }
}
```

To get a value, from any etcd local host:

```
# curl -L -s http://127.0.0.1:4001/v2/keys/message -XGET | python -mjson.tool
{
    "action": "get",
    "node": {
        "createdIndex": 5,
        "key": "/message",
        "modifiedIndex": 5,
        "value": "Hello Snoop Dog"
    }
}
```

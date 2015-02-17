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
 

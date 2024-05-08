# IPsec Manual Testing

The primary purpose of the manual testing procedure descibed in this note is to verify the IPsec setup and the workload(s) readiness. Scale, performance tunings and metric capturing are outside the scope of this procedure.

## Testbed

Assumming your testbed is comprised of a OCP 4.15 Single Node Openshift (SNO) cluster and a RHEL server. 
The IPsec tunnel between the SNO and RHEL is configured according to the instruction by [Jakob](https://github.com/jakobmoellerdev/north-south-ipsec-openshift-poc/tree/main/4.15)

- A Bastion/jumphost machine,
- SNO is the System-under-test
-- node IP: 192.18.10.10
- RHEL relevant info:
-- host IP: 192.18.10.15
-- IPsec tunnel interface: 100.50.1.8/24


## Workloads:
- iperf3 and uperf both are client-server style of applications.
- The client initiates a connection to the server to config and start the test.

## Test topologies
 ### INGRESS:
- workoad server runs on SNO
- workload client runs on RHEL

In the INGRESS topology, we will be using k8s nodePort service to expose the workload server to the external world.
 ### EGRESS:
- workoad server runs on RHEL
- workload client runs on SNO

### Prerequisite:
### Build the special Uperf that supports "-B" option 
On the RHEL machine.
```
git clone https://github.com/HughNhan/uperf.git
cd uperf
git switch double-free-and-dash-B
autoreconf --install
CFLAGS= ./configure --disable-sctp
make
make install
```
#### Get the custom resource (CR) files
On the bastion, clone this repo.
```
[root@d16-h06-000-r650:IPsec-bastion ]$ git clone https://github.com/HughNhan/IPsecTest.git ; cd IPsecTest
```
Step 1: create 2 sets of nodePort one for iperf3 and one for uperf.
```
[root@d16-h06-000-r650:IPsec-bastion ]$ oc apply -f uperf-node-port.yml
[root@d16-h06-000-r650:IPsec-bastion ]$ oc apply -f iperf-node-port.yml

[root@d16-h06-000-r650:IPsec-bastion ]$ oc get svc
NAME       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                           AGE
iperf-np   NodePort   172.30.43.15     <none>        30000:30000/TCP,30001:30001/TCP,30000:30000/UDP,30001:30001/UDP   4d8h
uperf-np   NodePort   172.30.206.216   <none>        31000:31000/TCP,31001:31001/TCP,31000:31000/UDP,31001:31001/UDP   4d9h
```

Step 2: Create a uperf pod and an iperf3 pod. Note the iperf3 image is equivalent to upstream, but the
 uperf image contains an enhancement that supports "-B" option needed for bind to the ipsec tunnel interface when necessary.
```
[root@d16-h06-000-r650:IPsec-bastion ]$ oc apply -f uperf-ipsec-pod.yaml
[root@d16-h06-000-r650:IPsec-bastion ]$ oc apply -f iperf-ipsec-pod.yaml
[root@d16-h06-000-r650:IPsec-bastion ]$ oc get pod
NAME        READY   STATUS    RESTARTS   AGE
iperf-app   1/1     Running   0          4d8h
uperf-app   1/1     Running   0          69m
```
# IPERF Testing
In the rest of IPERF tests, "oc rsh iperf-app" to run the iperf3 client or server. On the RHEL, assuming you have install iperf3, and there is nothing special about invoking iperf3 on the RHEL.

## NO IPsec testing
### Egress test:
```
    Server on RHEL: $ iperf3 -s -p 40000  <=== regular port
    Client on Pod:  $ iperf3 -c 198.18.10.15 -p 40000  -u -t10
```
### Ingress test:
```
    Server on nodePort/ Pod: iperf3 -s -p 30000    <=== nodePort
    Client on RHEL:          iperf3 -c 198.18.10.10 -p 30000  -u -t10
                                      ^^^^^^^^^^^      ^^^^^^
                                         SNO IP      nodePort
```
## IPsec testing:

### Egress test:
```
    server on RHEL: $ iperf3 -s -B 100.50.1.8 -p 40000  <=== regular port
                                   ^^^^^^^^^^
                                IPsec interface
    client on Pod:  $ iperf3 -c 100.50.1.8 -p 40000  -u -t10
                                ^^^^^^^^^^
```
### Ingress test:
```
    server on nodePort/Pod: iperf3 -s -p 30000    <=== nodePort
    client on RHEL:         iperf3 -c 198.18.10.10 -B 100.50.1.8 -p 30000  -u -t10
                                      ^^^^^^^^^^^     ^^^^^^^^      ^^^^^^
                                      SNO IP         ipsec IP      nodePort
```
# Uperf testing:
Common to the rest of Uperf tests, for the pod-side uperf invocation, 
```
oc rsh uperf-app
cd /usr/workload/xml-files
uperf <argument list> 
```
For RHEL-side uperf invocation,
``` 
cd <dir-where-you-build-uperf-per-the-prerequite-section>/workload/xml-files
uperf <argument list>
```

## NO ipsec testing:

### Egress test :
```
    Server on RHEL: $ uperf -s -P 41000  <=== regular port
    Client on Pod:  $ remotehost=198.18.10.15  port=41001 duration=10 wsize=512 rsize=4096 nthreads=1 protocol=udp  uperf -v -R -m stream.xml -P 41000
                                 ^^^^^^^^^^        ^^^^^^                                                                                     ^^^^^^^
                                remotehost IP     data port                                                                               control port
```
### Ingress test:
```
    Server on nodePort/ Pod: uperf -s -P 31000    <=== control nodePort
    Client on RHEL:          remotehost=198.18.10.10  port=31001 duration=10 wsize=512 rsize=4096 nthreads=1 protocol=udp  uperf -v -R -m stream.xml -P 31000
                                        ^^^^^^^^^^^      ^^^^^^^                                                                                        ^^^^^^
                                        SNO IP         data nodePort                                                                            control nodePort
```
## Ipsec testing:
### Egress test:
```
  server on RHEL: $ uperf -s -B 100.50.1.8 -P 41000  <=== regular control port
                                ^^^^^^^^^^
                                IPsec interface
  client on Pod:  $ remotehost=100.50.1.8   port=41001 duration=10 wsize=512 rsize=4096 nthreads=1 protocol=udp  uperf -v -R -m stream.xml -P 41000
                              ^^^^^^^^^^        ^^^^^^                                                                                      ^^^^^^^
                         RHEL IPsec interface  data port                                                                               control port
```
### Ingress test:
```
  Server on nodePort/Pod: $ uperf -s -P 31000    <=== control nodePort
                                       SNO IP           data nodePort
                                        vvvvvvvv          vvvvvv
  Client on RHEL:         $ remotehost=198.18.10.10  port=31001 duration=10 wsize=512 rsize=4096 nthreads=1 protocol=udp  \
                             uperf -v -R -m stream.xml -P 31000  -B 100.50.10.8
                                                          ^^^^^    ^^^^^^^^^^
                                                control nodePort  IPsec interface
```


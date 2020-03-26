---
title: SCION IP Gateway (SIG)
parent: Applications
nav_order: 60
---

# SCION IP Gateway (SIG)

The [SCION IP Gateway `SIG`](https://github.com/netsec-ethz/scion/tree/scionlab/go/sig) enables legacy IP applications to communicate over SCION. This tutorial describes how to set up two SIGs locally to test the SIG can enable any IP application to communicate over SCION.

## Environment

To test the SIG we will make use of the Vagrant configurations provided on [scionlab.org](https://scionlab.org/).
Set up a Vagrant VM on a host A and a host B using the instructions from [Virtual machine with VPN](https://netsec-ethz.github.io/scion-tutorials/virtual_machine_setup/dynamic_ip/).


Use locations specific to your test:

```shell
export SC=/etc/scion
export LOGS=/var/log/scion
```

## Configuring the two SIGs

We will now create the configuration for two SIG instances, one on the AS you started on host A `AS A` which we will call sigA and one on host B in `AS B`, which we will call sigB.
Create the configuration directories for the SIGs, build the SIG binary and set the linux capabilities on the binary:

```shell
export IA=$(cat $SC/gen/ia)
export IAd=$(cat $SC/gen/ia | sed 's/_/\:/g')
export AS=$(cat $SC/gen/ia | cut --fields=2 --delimiter="-")
export ISD=$(cat $SC/gen/ia | cut --fields=1 --delimiter="-")
sudo mkdir -p ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/
cd ~
git clone -b scionlab https://github.com/netsec-ethz/scion
go build -o $GOPATH/bin/sig ~/scion/go/sig/main.go
sudo setcap cap_net_admin+eip $GOPATH/bin/sig
```

A new entry needs to be added to the end of `/etc/hosts` adding the AS as an entry for localhost.


```shell
${IA},[10.0.8.XX]	localhost
```

You need to replace ${IA} with the AS id of the local AS, and 10.0.8.XX with the openVPN tunnel's IP address.

Enable routing:

```shell
sudo sysctl net.ipv4.conf.default.rp_filter=0
sudo sysctl net.ipv4.conf.all.rp_filter=0
sudo sysctl net.ipv4.ip_forward=1
```

### SIG A

Create the configuration for the sigA at ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigA.config:

(You need to replace ${AS}, ${IA} and ${IAd} with the actual values on your system in these configuration files.)

```
[sig]
  # ID of the SIG (required)
  ID = "sigA"

  # The SIG config json file. (required)
  SIGConfig = "${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigA.json"

  # The local IA (required)
  IA = "${IAd}"

  # The bind IP address (required)
  IP = "172.16.0.11"

  # Control data port, e.g. keepalives. (default 10081)
  CtrlPort = 10081

  # Encapsulation data port. (default 10080)
  EncapPort = 10080

  # SCION dispatcher path. (default "")
  Dispatcher = ""

  # Name of TUN device to create. (default DefaultTunName)
  Tun = "sigA"

  # Id of the routing table (default 11)
  TunRTableId = 11

[sd_client]
  # Sciond path. It defaults to sciond.DefaultSCIONDPath.
  Path = "/run/shm/sciond/default.sock"

  # Maximum time spent attempting to connect to sciond on start. (default 20s)
  InitialConnectPeriod = "20s"

[logging]
[logging.file]
  # Location of the logging file.
  Path = "${LOG}/sig${IA}-1.log"

  # File logging level (trace|debug|info|warn|error|crit) (default debug)
  Level = "debug"

  # Max size of log file in MiB (default 50)
  # Size = 50

  # Max age of log file in days (default 7)
  # MaxAge = 7

  # How frequently to flush to the log file, in seconds. If 0, all messages
  # are immediately flushed. If negative, messages are never flushed
  # automatically. (default 5)
  FlushInterval = 5
[logging.console]
  # Console logging level (trace|debug|info|warn|error|crit) (default crit)
  Level = "debug"

[metrics]
# The address to export prometheus metrics on. (default 127.0.0.1:1281)
  Prometheus = "127.0.0.1:1282"
```

You can do so with the following command:

```shell
sudo sed -i "s/\${IA}/${IA}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigA.config
sudo sed -i "s/\${IAd}/${IAd}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigA.config
sudo sed -i "s/\${AS}/${AS}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigA.config
sudo sed -i "s/\${ISD}/${ISD}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigA.config
sudo sed -i "s/\${SC}/\/etc\/scion/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigA.config
sudo sed -i "s/\${LOG}/${LOG}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigA.config
```


Create the traffic rules for the sigA at ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigA.json:

```
{
    "ASes": {
        "17-ffaa:1:XXX": {
            "Nets": [
                "172.16.12.0/24"
            ],
            "Sigs": {
                "remote-1": {
                    "Addr": "10.0.8.XXX",
                    "CtrlPort": 10084,
                    "EncapPort": 10083
                }
            }
        }
    },
    "ConfigVersion": 9001
}
```
You need to replace the "10.0.8.XXX" IP with the openVPN tunnel IP of the remote host B and 17-ffaa:1:XXX with the AS id of the remote AS B.


The infrastructure keeps several versions of the file `topology.json` and you will need to edit them. Be sure to only add the section below in these `topology.json` files, without removing other sections:
```
  "SIG": {
     "sig17-ffaa_1_XXX-1": {
       "Addrs": {
         "IPv4": {
           "Public": {
             "Addr": "172.16.0.XX",
             "L4Port": 31056
           }
         }
       }
     }
   },
```
Make this edit to the following files, then replace the "172.16.0.XX" IP with the openVPN tunnel IP of the local host A and 17-ffaa_1_XXX with the AS id of the local AS A (ensuring to keep the `-1` at the end):
```
${SC}/gen/ISD${ISD}/AS${AS}/endhost/topology.json
${SC}/gen/ISD${ISD}/AS${AS}/br${IA}-1/topology.json
${SC}/gen/ISD${ISD}/AS${AS}/bs${IA}-1/topology.json
${SC}/gen/ISD${ISD}/AS${AS}/cs${IA}-1/topology.json
${SC}/gen/ISD${ISD}/AS${AS}/ps${IA}-1/topology.json
```

### SIG B

And similarly the configuration and traffic rules for the sigB:

at ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigB.config

```
[sig]
  # ID of the SIG (required)
  ID = "sigB"

  # The SIG config json file. (required)
  SIGConfig = "${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigB.json"

  # The local IA (required)
  IA = "${IAd}"

  # The bind IP address (required)
  IP = "172.16.0.12"

  # Control data port, e.g. keepalives. (default 10081)
  CtrlPort = 10084

  # Encapsulation data port. (default 10080)
  EncapPort = 10083

  # SCION dispatcher path. (default "")
  Dispatcher = ""

  # Name of TUN device to create. (default DefaultTunName)
  Tun = "sigB"

  # Id of the routing table (default 11)
  TunRTableId = 12

[sd_client]
  # Sciond path. It defaults to sciond.DefaultSCIONDPath.
  Path = "/run/shm/sciond/default.sock"

  # Maximum time spent attempting to connect to sciond on start. (default 20s)
  InitialConnectPeriod = "20s"

[logging]
[logging.file]
  # Location of the logging file.
  Path = "${LOG}/sig${IA}-1.log"

  # File logging level (trace|debug|info|warn|error|crit) (default debug)
  Level = "debug"

  # Max size of log file in MiB (default 50)
  # Size = 50

  # Max age of log file in days (default 7)
  # MaxAge = 7

  # How frequently to flush to the log file, in seconds. If 0, all messages
  # are immediately flushed. If negative, messages are never flushed
  # automatically. (default 5)
  FlushInterval = 5
[logging.console]
  # Console logging level (trace|debug|info|warn|error|crit) (default crit)
  Level = "debug"

[metrics]
# The address to export prometheus metrics on. (default 127.0.0.1:1281)
  Prometheus = "127.0.0.1:1282"
```

and

```shell
sudo sed -i "s/\${IA}/${IA}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigB.config
sudo sed -i "s/\${IAd}/${IAd}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigB.config
sudo sed -i "s/\${AS}/${AS}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigB.config
sudo sed -i "s/\${ISD}/${ISD}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigB.config
sudo sed -i "s/\${SC}/${SC}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigB.config
sudo sed -i "s/\${LOG}/${LOG}/g" ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigB.config
```

and

at ${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigB.json

```
{
    "ASes": {
        "17-ffaa:1:XXX": {
            "Nets": [
                "172.16.11.0/24"
            ],
            "Sigs": {
                "remote-1": {
                    "Addr": "10.0.8.XXX",
                    "CtrlPort": 10081,
                    "EncapPort": 10080
                }
            }
        }
    },
    "ConfigVersion": 9001
}
```
You need to replace the "10.0.8.XXX" IP with the openVPN tunnel IP of the remote host A and 17-ffaa:1:XXX with the AS id of the remote AS A.

and

```
  "SIG": {
     "sig17-ffaa_1_XXX": {
       "Addrs": {
         "IPv4": {
           "Public": {
             "Addr": "172.16.0.XX",
             "L4Port": 31056
           }
         }
       }
     }
   },
```
Make this edit to the following files, then replace the "172.16.0.XX" IP with the openVPN tunnel IP of the local host B and 17-ffaa_1_XXX with the AS id of the local AS B (ensuring to keep the `-1` at the end):
```
${SC}/gen/ISD${ISD}/AS${AS}/endhost/topology.json
${SC}/gen/ISD${ISD}/AS${AS}/br${IA}-1/topology.json
${SC}/gen/ISD${ISD}/AS${AS}/bs${IA}-1/topology.json
${SC}/gen/ISD${ISD}/AS${AS}/cs${IA}-1/topology.json
${SC}/gen/ISD${ISD}/AS${AS}/ps${IA}-1/topology.json
```

### Create Interfaces and Run

Since we are running our SIGs in a VM and we do not have spare physical interfaces on which to run them, we will create two dummy interfaces:

```shell
sudo modprobe dummy
```

On host A:
```shell
sudo ip link add dummy11 type dummy
sudo ip addr add 172.16.0.11/32 brd + dev dummy11 label dummy11:0
```

On host B:
```
sudo ip link add dummy12 type dummy
sudo ip addr add 172.16.0.12/32 brd + dev dummy12 label dummy12:0
```

Now we need to add the routing rules for the two SIGs:

On host A:
```shell
sudo ip rule add to 172.16.12.0/24 lookup 11 prio 11
```

On host B:
```shell
sudo ip rule add to 172.16.11.0/24 lookup 12 prio 12
```

Now start the two SIGs with the following commands:


On Host A:
```shell
$GOPATH/bin/sig -config=${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigA.config > $SC/logs/sig${IA}-1.log 2>&1 &
```
and Host B:
```shell
$GOPATH/bin/sig -config=${SC}/gen/ISD${ISD}/AS${AS}/sig${IA}-1/sigB.config > $SC/logs/sig${IA}-1.log 2>&1 &
```

To show the ip rules and routes, run:
```shell
sudo ip rule show
sudo ip route show table 11
sudo ip route show table 12
```

## Testing

You can test that your SIG configuration works by running some traffic over it.

Add some server on host B and client on host A:

Host B:
```shell
sudo ip link add server type dummy
sudo ip addr add 172.16.12.1/24 brd + dev server label server:0

mkdir $SC/WWW
echo "Hello World!" > $SC/WWW/hello.html
cd $SC/WWW/ && python3 -m http.server --bind 172.16.12.1 8081 &
```

Host A:
```shell
sudo ip link add client type dummy
sudo ip addr add 172.16.11.1/24 brd + dev client label client:0
```


Query the server running on host B from host A:

Host A:
```shell
curl --interface 172.16.11.1 172.16.12.1:8081/hello.html
```

You should see the "Hello World!" message as output from the last command.

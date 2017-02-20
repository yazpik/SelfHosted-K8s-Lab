# First Steps on Self-Hosted Kubernetes (1 of 2)


<img src="http://www.andrewlipson.com/escher/relativity-1600.jpg" width="500" height="400">

From [Wikipedia](https://en.wikipedia.org/wiki/History_of_compiler_construction)
> "Self-hosting is the use of a computer program as part of the toolchain or operating system that produces new versions of that same program—for example, a compiler that can compile its own source code."

Thinking about the foundation of a Kubernetes production ready cluster, I decided
to have a look at this, and is something will break your brain a bit, or at least I did it
for me because I wasn't familiar with *Self-Hosted* term.

At the end kubernetes is really good to simplify software lifecycle management
why not to do it by managing it's own core components?.

So I found CoreOS have made a significative progress with bootkube and matchbox

## Bootkube and Matchbox (formerly known as CoreOS BareMetal)
### Part 1 Boostrap a Self-hosted Kubernetes Cluster

Deploy the basic requeriments to run a kubernetes Self-Hosted cluster on Baremetal.
This is a howto based on official docs from CoreOS,
How to run matchbox on a Linux machine with rkt/docker and CNI to network boot and provision a cluster of QEMU/KVM CoreOS machines locally.

For now at least one Baremetal server is needed, I tried to run Matchbox and Bootkube on
a Vbox, Vmware Fusion and one Openstack Cloud instance to
deploy a SelfHosted kubernetes cluster but at the moments there are issues with selfhosted flannel or at least that was my finding,
I actually created one issue about it, using my personal github account, you can track it here [Issue #437](https://github.com/coreos/matchbox/issues/437)

So basically this is the main idea
- One Baremetal server that is going to create 3 virtual instances (libvirt script)
- One Matchbox container that will provide the server config thru
[iPXE](https://coreos.com/os/docs/latest/booting-with-ipxe.html) and [Ignition](https://github.com/coreos/ignition) to the QEMU/KVM virtual instances
- One [dnsmasq](https://github.com/coreos/matchbox/tree/master/contrib/dnsmasq) container to provide DNS, DHCP services using this [config](https://github.com/coreos/matchbox/blob/master/contrib/dnsmasq/metal0.conf)

![imagen](https://cloud.githubusercontent.com/assets/7389339/23139792/d98ad950-f773-11e6-9db5-621eb2a916f5.jpg)

#### Required packages
```bash
# Fedora 25
sudo dnf install virt-install virt-manager git gpg rkt

# Debian/Ubuntu
sudo apt-get install virt-manager virtinst libvirt-bin qemu-kvm systemd-container git gpg
# Install rkt 1.24.0 on Ubuntu 14.04
gpg --recv-key 18AD5014C99EF7E3BA5F6CE950BDD3E0FC8A365E
wget https://github.com/coreos/rkt/releases/download/v1.24.0/rkt_1.24.0-1_amd64.deb
wget https://github.com/coreos/rkt/releases/download/v1.24.0/rkt_1.24.0-1_amd64.deb.asc
gpg --verify rkt_1.24.0-1_amd64.deb.asc
sudo dpkg -i rkt_1.24.0-1_amd64.deb 
```
From here basically I'm using [official docs](https://github.com/coreos/matchbox) with some changes

Set selinux to permissive if you are using Fedora
```
sudo setenforce Permissive
```
**Note**: rkt does not yet integrate with SELinux on Fedora.
As a workaround, temporarily set enforcement to permissive if you are comfortable.
Check the rkt [distribution notes](https://github.com/coreos/rkt/blob/master/Documentation/distributions.md) or see the tracking
[issue](https://github.com/coreos/rkt/issues/1727).

Clone the [matchbox](https://github.com/coreos/matchbox) source
```bash
git clone https://github.com/coreos/matchbox.git
cd matchbox
```
Download CoreOS image assets
```bash
./scripts/get-coreos stable 1235.9.0 ./examples/assets
```
Only if you are using RKT

Define the `metal0` virtual bridge with CNI

```bash
sudo mkdir -p /etc/rkt/net.d
sudo bash -c 'cat > /etc/rkt/net.d/20-metal.conf << EOF
{
  "name": "metal0",
  "type": "bridge",
  "bridge": "metal0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "172.18.0.0/24",
    "routes" : [ { "dst" : "0.0.0.0/0" } ]
   }
}
EOF'
```

On Fedora, add the `metal0` your firewall configuration.
```bash
sudo firewall-cmd --add-interface=metal0 --zone=trusted
```
Also on Fedora disable network manager
```
systemctl stop NetworkManager.service
systemctl disable NetworkManager.service
```

For development/testing purposes is need to add entries on /etc/hosts, e.g
#### if you are using RKT
```
172.18.0.21 node1.example.com
172.18.0.22 node2.example.com
172.18.0.23 node3.example.com
```

#### if you are using Docker
```
172.17.0.21 node1.example.com
172.17.0.22 node2.example.com
172.17.0.23 node3.example.com
```

Time to run the containers manually, you can run matchbox and bootkube with either RKT or Docker, with Docker Metal0 interface is not needed, it will use Docker0 instead and it will be automatically created


#### Bootkube example with RKT

If you want to try a different example just change bootkube to a different value, 
** *(groups,kind=host,source=$PWD/examples/groups/bootkube)* ** 

It could be any of this [examples](https://github.com/coreos/matchbox/tree/master/examples)

The difference between [bootkube and bootkube-install](https://github.com/coreos/matchbox/issues/437#issuecomment-279883120)
"bootkube PXE boots a cluster. bootkube-install PXE boots, installs to disk, and reboots before becoming a cluster." 

So I'll be using bootkube example to bootstrap a Self-hosted kubernetes cluster on virtual instances.

```
#### matchbox + bootkube
rkt run --net=metal0:IP=172.18.0.2 --mount volume=data,target=/var/lib/matchbox --volume data,kind=host,source=$PWD/examples --mount volume=groups,target=/var/lib/matchbox/groups --volume groups,kind=host,source=$PWD/examples/groups/bootkube quay.io/coreos/matchbox:latest -- -address=0.0.0.0:8080 -log-level=debug

#### dnsmasq
rkt run coreos.com/dnsmasq:v0.3.0 --net=metal0:IP=172.18.0.3 --mount volume=config,target=/etc/dnsmasq.conf --volume config,kind=host,source=$PWD/contrib/dnsmasq/metal0.conf
```

#### Bootkube example with Docker
```
dnf install docker
sudo systemctl start docker
docker pull quay.io/coreos/matchbox:latest
docker run -p 8080:8080 --rm -v $PWD/examples:/var/lib/matchbox:Z -v $PWD/examples/groups/bootkube:/var/lib/matchbox/groups:Z quay.io/coreos/matchbox:latest -address=0.0.0.0:8080 -log-level=debug

#### dnsmasq
sudo docker run --name dnsmasq --cap-add=NET_ADMIN -v $PWD/contrib/dnsmasq/docker0.conf:/etc/dnsmasq.conf:Z quay.io/coreos/dnsmasq -d
```
In this case, dnsmasq runs a DHCP server allocating IPs to VMs between 172.17.0.43 and 172.17.0.99, resolves matchbox.foo to 172.17.0.2 (the IP where matchbox runs), and points iPXE clients to http://matchbox.foo:8080/boot.ipxe.


If you want to tail logs for matchbox and dnsmasq
#### Tail logs
```
journalctl -f -u dev-matchbox
journalctl -f -u dev-dnsmasq
```

After that you should be able to hit the endpoints exposed by the services

#### RKT
- iPXE http://172.18.0.2:8080/ipxe?mac=52:54:00:a1:9c:ae
- Ignition http://172.18.0.2:8080/ignition?mac=52:54:00:a1:9c:ae
- Metadata http://172.18.0.2:8080/metadata?mac=52:54:00:a1:9c:ae


#### Docker
- iPXE http://127.0.0.1:8080/ipxe?mac=52:54:00:a1:9c:ae
- Ignition http://127.0.0.1:8080/ignition?mac=52:54:00:a1:9c:ae
- Metadata http://127.0.0.1:8080/metadata?mac=52:54:00:a1:9c:ae

### Creation of one Controller and two Workers (Virtual instances)
As I mentioned, just testing this environment on one baremetal host, so I'm going to use virt-install to create 3 virtual instances,
acting one as a controller and two more as workers.

https://github.com/coreos/matchbox/blob/master/scripts/libvirt#L44-L46
This one works for Fedora25 with (libvirt 2.2)
```bash
COMMON_VIRT_OPTS="--memory=1024 --vcpus=1 --pxe --disk pool=default,size=6 --os-type=linux --os-variant=generic --noautoconsole --events on_poweroff=preserve"
```

For Ubuntu 14.04 (libvirt 1.2) I needed to make the following changes, 
```bash
COMMON_VIRT_OPTS="--ram=1024 --vcpus=1 --pxe --disk pool=default,size=6,format=qcow2 --os-type=linux --os-variant=rhel7 --noautoconsole" #--events on_poweroff=preserve"
```

You can change those params according your needs, *{matchbox_path}/scripts/libvirt*

Create 3 virtual instances, according what you configured on libvirt script
```
sudo ./scripts/libvirt create
```

#### This is only needed for in case you are using Ubuntu 14.04 and libvirt 1.2
Look the static ignition manifest, it is configured by default to use "sda", but libvirt 1.2 will create the disks as "vda"

**examples/ignition/bootkube-controller.yaml**
```yaml
storage:
  {{ if index . "pxe" }}
  disks:
    - device: /dev/vda
      wipe_table: true
      partitions:
        - label: ROOT
  filesystems:
    - name: root
      mount:
        device: "/dev/sda1"  <--- Just change this line libvirt 1.2 will use vda1 instead
        format: "ext4"
        create:
          force: true
          options:
            - "-LROOT"
  {{end}}
```

**examples/profiles/bootkube-worker.json**

```json
{
  "id": "bootkube-controller",
  "name": "bootkube Ready Controller",
  "boot": {
    "kernel": "/assets/coreos/1235.9.0/coreos_production_pxe.vmlinuz",
    "initrd": ["/assets/coreos/1235.9.0/coreos_production_pxe_image.cpio.gz"],
    "args": [
      "root=/dev/vda1", # <--- This line
      "coreos.config.url=http://matchbox.foo:8080/ignition?uuid=${uuid}&mac=${mac:hexhyp}",
      "coreos.first_boot=yes",
      "console=tty0",
      "console=ttyS0",
      "coreos.autologin"
    ]
  },
  "ignition_id": "bootkube-controller.yaml"
```  

Do the same for 
- examples/ignition/bootkube-worker.yaml
- examples/profiles/bootkube-worker.json


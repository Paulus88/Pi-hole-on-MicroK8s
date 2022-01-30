# PiHole-MicroK8s
I have watched a lot of tutorials on running Pi-hole in an MicroK8s setup but none of them resulted in a fully working solution that offered all the features in Pi-hole unless you do some tricks with a load balancer and have a router/firewall that supported DHCP Relay.
I am sharing my YAML for those who have a normal home network and are looking for a redundant and light weight version of our favorite DNS sinkhole with DHCP enabled.
## Requirements
* Some experience and understanding of DNS and DHCP.
* Give device(s) a static IP. (as device is responsible to lease addresses best make sure it gets an address outside the DHCP range)

Preferably
* At least 2 devices that support a linux distro.
## Setup MicroK8s
Install Snapcraft.
```
#Debian
sudo apt install snapd
#Redhat
sudo yum install snapd
```
Install MicroK8s (Optional if your like me and want to play with the latest toys add --edge)
```
sudo snap install microk8s --classic
```
Wait for install to complete.
```
microk8s status --wait-ready
```
Optional install plugins. I myself believe in standardization and GUI's help keep things to a certain degree of standard.
```
microk8s enable portainer
```
MicroK8s is setup and if you want to access the Portainer GUI just go to <http:// Device IP:30777>.

## Setup Pi-hole
Create a **StorageClass** or just enable the microk8s-hostpath storage using Portainer found at the bottom of **Cluster** -> **Setup**.

Create 2x Persistent Volumes Claims (PVC) these will be used to save all your Pi-hole settings.
```
microk8s kubectl apply -f https://raw.githubusercontent.com/Paulus88/Pi-hole-MicroK8s/main/dnsmasq-pvc.yaml
microk8s kubectl apply -f https://raw.githubusercontent.com/Paulus88/Pi-hole-MicroK8s/main/pihole-pvc.yaml
```
Now your ready to deploy Pi-hole.
```
microk8s kubectl apply -f https://raw.githubusercontent.com/Paulus88/Pi-hole-MicroK8s/main/pihole.yaml
```
Vola Pi-hole is now accessable on port 8053: <http:// Device IP:8053>.
## What makes this version of Pi-hole different from other tutorials?
**Affinity**
```
       affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - pihole
```
This tells kubernetes that the Pi-hole pod must run on a linux machine and 2 pods with the pihole app are not allowed to run on the same device at the one time.

**hostNetwork** in Docker known as --net=host.
```
      hostNetwork: true
```
This is actually the most important component of this setup, it means you do not need a DHCP Relay supported router/firewall to allow Pi-hole to do DHCP from a container/pod.

**securityContext**
```
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_BIND_SERVICE
            - NET_RAW
```
All the needed capabilities to take full advantages of Pi-hole on your network.

**volumeMounts**
```
        volumeMounts:
        - mountPath: /etc/dnsmasq.d
          name: dnsmasq
        - mountPath: /etc/pihole
          name: pihole
```
The Persistent Volume mounts to save all your Pi-hole data even if pod is redeployed.
## Redundancy
You thought we were done right? Working Pi-hole, yeeey but what if one device fails?

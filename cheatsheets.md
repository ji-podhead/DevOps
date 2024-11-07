# cheatsheets & Unreleased Guides
## Symbolik to duplicate README.md for package in subfolder of repo
```bash
 $ ln -f ./README.md /home/ji/Dokumente/podshop-org/opnsense-helper/python/README.md
```

## Network Segmentation, Firewall & Intrusion Detection
So i wanted to create a secure infrastructure in my datacenter / private cloud.... that lead me to this article


### Why you should'nt underestimate the power of layer2 and why you should isolate your VM Traffic

peopel might forget about this if they are hosting multiple vms and subnets on the same machine:
 - the vms/containers on the same machine can still talk to each other over layer2 since they are sharing the same nic (master)
 - if an atacker manages to break out, or hack into a container...
 - this can have various consequences. ***this can give hackers a huge attack space*** 
   
#### Possibe Attacks
  - ***Brute Force Attacks*** to crack password
  - ***DNS Attacks*** ****(pls see my dns notes)****
  	- spoof the entire dns by d(d)os and spoof a sensible page to fish password and maybe even redirect to the orig. ip (MMTM)
  	- dns cache poisining 
  	- ask the dns for all your dns entries
   	  - bring down the server and then spoof the ip directly and fish your password
  - ***DHCP Attacks***
    - this can have a variety of outcome. <br> most likely an attacker would ddos your dhcp and spoof it. <br>In order he can make use of DNS attacks and vice versa
    - he can of course send a new lease to a sensible page like vault. <br> Then he can replace that ip with his own phishing site <br> He also can also perform a MMTM to redirect via http router to the target ip for example    

  - ***mac spoofing***
    - bypass mac based dhcp filtering
      - you can run out of leases
        - you maybe loose connection
        - you will get ddosed and your machines overheat, run out of ram, run out of rom, or may even break
    - duplicated ip adresses (who knows what will happen)
      -  network can be completely broken
      -  router will continue its original route, unles:
      -  you might reconnect / lease runs out and the spoofed ip will be the target adress for the packets  

  - ***SSH Attacks*** ****(pls see my ssh notes)****

----

### IDS (Intrusion Detection System) 
An IDS is a system designed to monitor computer systems for signs of unauthorized access or malicious activities.<br> 
It analyzes network traffic and system logs to identify potential security threats. In the following we will use Suricata as our IDS
It recommended to use multi-wan for IDS rather than routing it directly through since this will slown down your traffic.

#### Suricata
Suricata is an open-source network intrusion detection engine and analysis platform. It's known for its high performance and ability to detect sophisticated attacks. Key features include:
- Multi-threaded architecture and GPU-support for handling large volumes of traffic
- Support for both signature-based and anomaly-based detection
- Ability to perform packet capture and analysis
- eBPF and XDP for network-filtering, loadbalancing and routing in kernelspace
- Integration with other security tools and platforms (comes as plugin for opnsense)

----

### Network Segmentation
#### segment local network traffic using iptables
- we will create 2 bridges for our wms, forward to our router (192.168.1.1) and drop connections between the bridges 

```bash
#!/bin/bash

# Create two separate Bridge Interfaces
ip link add br0 type bridge
ip link add br1 type bridge

# Configure the Bridge Interfaces
ip addr add 192.168.1.100/24 dev br0
ip addr add 192.168.2.100/24 dev br1

# Activate the Bridge Interfaces
ip link set br0 up
ip link set br1 up

# Configure iptables for forwarding from br0 to the target IP
iptables -A FORWARD -i br0 -d 192.168.1.1 -j ACCEPT

# Configure iptables for forwarding from br1 to the target IP
iptables -A FORWARD -i br1 -d 192.168.1.1 -j ACCEPT

# Block traffic between the bridges
iptables -A FORWARD -i br0 -o br1 -j DROP
iptables -A FORWARD -i br1 -o br0 -j DROP

# Add NAT entries to ensure outgoing connections are routed correctly
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

-----

#### Network FIltering using eBPF and XDP
Before we dropped the connections between our VM-bridges using iptables in userspace,  but theres another and more safe way of doing this.
##### eBPF
- eBPF is a framework that let you run code in kernel space.  
- this is not onl great for monitoring using telemtry data, but you can also drop packets before they even reach the userspace.

##### XDP
 - XDP (eXpress Data Path) is a high-performance networking technology developed by Red Hat that allows for efficient packet processing at the Linux kernel level. It enables direct access to hardware acceleration without going through the traditional network stack. 
 - So basically XDP allows us to create kernalspace based Network filters or load balancers without compiling a complete eBPF script.<br>
 This wouldnt be to hard, but XDP spares us a bunch of time and its also directly implemented into suricata, <br>
 so we can combine both our intrusion detection system and our network filters


##### sources
- suricata: [ebpf & xdp](https://suricatacn.readthedocs.io/zh-cn/suricata-4.1.0-beta1/capture-hardware/ebpf-xdp.html)
- stamus-networks: [Introduction to eBPF and XDP support in Suricata ](https://www.stamus-networks.com/hubfs/Library/Documents%20%28PDFs%29/StamusNetworks-WP-eBF-XDP-092021-1.pdf)

#### TODO
- provide examples
-----
    
#### Vlans and network segmentation
Vlans allow you to send multiple networks over a single conection.<br>
you can transport and share the networks over layer3 using a method called ***trunking*** <br>


##### Trunking
  - Trunking is a networking technology that allows multiple network segments to be combined into a single logical link, enabling efficient data transmission across multiple ports while maintaining separation between individual networks.
Shareable 
![trunking](https://upload.wikimedia.org/wikipedia/commons/4/4d/Router_on_a_stick_concept.png)
****image source: [wikipedia: router on a stick](https://en.wikipedia.org/wiki/Router_on_a_stick)****



#### tagged and untagged Vlans


| Type | Description | Purpose | Key Characteristics |
|------|-------------|---------|--------------------|
| Untagged VLAN | Untagged VLANs, often referred to as "native" or "default" VLANs, do not have any VLAN tag information | • Used as a fallback or default VLAN <br>• Often associated with management traffic or network services <br>• Can simplify configuration in certain scenarios | • No VLAN tag is present in the frame <br>• Typically associated with the native VLAN on trunk links <br>• Frames belong to the default VLAN configuration |
| Tagged VLAN | Tagged VLANs offer greater flexibility and are commonly used in complex networks <br> •Untagged VLANs can simplify configurations in simpler network topologies <br> •Trunk ports typically carry both tagged and untagged traffic <br> •Access ports usually carry only untagged traffic for a specific VLAN <br> •Tagged VLANs, also known as IEEE 802.1Q VLANs, use VLAN tags to identify traffic belonging to specific virtual LANs. These tags are added to Ethernet frames at the network switch level | • Provides network segmentation and isolation <br>• Supports multiple broadcast domains within a single physical network <br>• Facilitates easier network management and troubleshooting | • Frames are encapsulated with a 4-byte tag <br>• The tag includes a VLAN ID (VID) <br>• Allows for multiple VLANs on a single physical link <br>• Enables trunking between switches |

-----

#### OVS & Linux Bridges

| Bridge | Description | Characteristics |
| --- | --- | --- |
| Linux Bridge | Linux bridges are native Linux network devices that allow you to create virtual network segments within a single physical network interface. | • native to Linux kernel<br>• Created using brctl addbr command <br>• Managed by brctl utility<br>• Limited to Linux-specific features |
| OVS Bridge | OVS bridges are software switches implemented as part of the Open vSwitch project | • provides a vlan-aware switch on your host to isolate vm traffic<br> • Software implementation running in userspace<br> • Highly configurable and extensible • Supports distributed virtual switching |

-----

#### Tagged vlans and Linux Bridges

we will create 2 tagged vlans without a  wlan aware switch
- *****this will not work with libvirt!!!*****
  - libvirt + linux bridges are not vlan-aware without a `802.1Q-compliant / trunkable switch`
  - if you want to use libvirt and tagged vlans, pls see the section: *Isolate VM traffic using tagged vlans*
- however you might do your routing and switching elsewhere (like on your openwrt router)
- this would require to split the network received by your host.
- this is how you do this:
   
```bash
#enable vlan filtering on our bridge					
 sudo ip link set dev virbr1 type bridge vlan_filtering 1 vlan_default_pvid 1

# add vlans to our interface
 sudo ip link add link enp2s0 name vlan.0.1 type vlan id 1
 sudo ip link add link enp2s0 name vlan.0.2 type vlan id 2

# start the vlan-interface
 sudo ip link set vlan.0.1 up
 sudo ip link set vlan.0.2 up
 
 # add vlan to our bridge
 ip link set vlan.0.1 master virbr1
  ip link set vlan.0.2 master virbr1
 
 # add ip and subnet
 sudo ip addr add 192.168.0.4/24 dev vlan.0.1
 sudo ip addr add 192.168.0.5/24 dev vlan.0.2
```

----
### Debug VLAN-traffic using wireshark
#### Install tcpdump on openwrt and capture packets with wireshark via ssh
you can install packages using the gui (system -> software ), or via terminal of course.
- update your packagemanager
> ```bash
> opkg update
> ```
- install tcpdump
> ```
> opkg install tcpdump
>```
- try if it works
> ```bash
> tcpdump -D
> ```
> - if this fails with an error like this:
>   ```yaml
>   
>   	Error by extcap pipe: Error relocating /usr/bin/tcpdump: pcap_open: symbol not found
>		Error relocating /usr/bin/tcpdump: pcap_findalldevs_ex: symbol not found
>
>   ```
>   - try to reinstall libcap1 with force:
>     ```bash
>       opkg install --force-reinstall libpcap1
>     ```
- capture the packets using wireshark & tcdump via ssh:
> ```bash
>    ssh root@192.168.1.1 tcpdump -i eth0 -U -s0 -w - 'not port 22' | sudo wireshark -k -i -
> ```
> - keep in mind to enter the password after executing the command 
> - should output something like this:
>   ```bash
>   root@192.168.1.1's password: 06:24:03.992     Main Warn QStandardPaths: XDG_RUNTIME_DIR not set, defaulting to '/tmp/runtime-root'
>
>    tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
>   ```

#### generate vlan traffic to check response
- on kali we will use `arping -i <interface> <target_ip> -S <source_ip>`
  - i think it doesnt actually matter if the source ip is in the targets subnet cause we will use arp-request  
- here we will send a arp ping command from the vm that is attached to a vlan interface
   - tag: 2
   - subnet 192.168.20.0/24
  >  ![image](https://github.com/user-attachments/assets/c49e7eaa-0727-492d-ae41-b483ef0e213f)
  > - notice that we are seing traffic on our vlan in opensense as well as on our vlan trunk interface (parent)
  > - arp ping will basically send a arp-request via broadcast address,hence we need a source address, or we need to brute force
  > - wireshark filter is set to `eth.type == 0x0806 || vlan.id == 2` for arp traffic with vlan tag 2
  > - you can tell that its using vlan tag 2 because it says so in the ethernet frame
- arp ping using the ovs-bridge host and vlan interface
  ```bash
  $ arping -I dns  192.168.20.1
  ```
>    - ```bash 
>		ARPING 192.168.20.1 von 200.0.2.1 dns
>		Unicast antworten von 192.168.20.1 [52:54:00:ED:B9:F9]  1.497ms
>		Unicast antworten von 192.168.20.1 [52:54:00:ED:B9:F9]  1.270ms
>		Unicast antworten von 192.168.20.1 [52:54:00:ED:B9:F9]  0.981ms
>       ```
  
### isolate vm/container traffic using tagged vlans and ovs bridges

#### Management and Data Network
![grafik](https://github.com/user-attachments/assets/0d634ad0-6bc5-4048-9b77-0bc85e45f04a)
****image source: [openvswitch](https://docs.openvswitch.org/en/latest/howto/vlan/)****

As you can see in the image, we have 2 NIC's in this setup.<br>
This is required if you want to make sure not to loose your connection (like ssh) since your physical network interface will be the slave of the ovs-bridge.<br>
You cant have duplicated IP's on different Interfaces at the same time.<br>
So if you use your main NIC as both your Management- and Data Network, you need to delete & flush its IP, as well as setting it to 0.0.0.0/0.<br>
Thats why you can loose your ssh connection and it might require you to access the machine directly.  However i wrote a script and [Ansible Collection](https://galaxy.ansible.com/ui/repo/published/ji_podhead/ovs_bridge/)
   
 #### install the requirements
 
 ```bash
# install openvswitch to create a vlan-aware switch on your host
$ sudo dnf install openvswitch
# install the networkManager plugin that is required to create the ovs network devices
$ sudo dnf install NetworkManager-ovs
```
#### Create the OVS-Bridge and Vlans using nmcli ***ONLY ONE NIC REQUIRED!!!*** ***DIRECT USAGE!!!***
```
#!/bin/bash

nmcli conn add type ovs-bridge conn.interface vlanbr autoconnect yes  
nmcli conn add type ovs-port conn.interface connection_port master  vlanbr autoconnect yes

nmcli conn add type ovs-interface conn.interface vlanbr master connection_port autoconnect yes ipv4.method auto
nmcli conn add type ovs-port conn.interface interface_port master vlanbr autoconnect yes
nnmcli conn down vlanbr
nmcli connection modify ovs-slave-vlanbr ipv4.method static ipv4.address 192.168.1.100/24 ipv4.gateway 192.168.1.1

nmcli c add type ovs-port conn.interface vlan1 master vlanbr ovs-port.tag 1 con-name vlan-port-1
nmcli c add type ovs-interface slave-type ovs-port conn.interface vlan1 master vlan-port-1 con-name vlan-con-1 ipv4.method static ipv4.address 192.168.7.1/24
nmcli c add type ovs-port conn.interface vlan2 master vlanbr ovs-port.tag 2 con-name vlan-port-2
nmcli c add type ovs-interface slave-type ovs-port conn.interface vlan2 master vlan-port-2 con-name vlan-con-2 ipv4.method static ipv4.address 192.168.8.1/24

nmcli conn add type ethernet conn.interface enp2s0 master interface_port autoconnect yes && ip addr flush dev enp2s0 && nmcli con up ovs-slave-enp2s0 && ip addr add 192.168.1.100/24 dev vlanbr
```
   - make sure to use the same name for the ovs-bridge connection and ovs-bridge-interface!

#### useful commands
- openvswitch overview: `ovs-vsctl show`
- list ovs-bridges: `ovs-vsctl list-br`
- list ovs-ports: `ovs-vsctl list-ports <bridge>`
- list ovs-interfaces: `ovs-vsctl list-ifaces <bridge>`
- delete bridge: `ovs-vsctl del-br <bridge>`
- add a bridge: `ovs-vsctl add-br ovsbr`
- add a port: `ovs-vsctl add-port <ovs-bridge> <nic>`
  - ***this will add your physical nic to the bridge so you might loose ssh connection!***
- add a vlan: `ovs-vsctl add-port br0 <vlan> tag=<vlan tag> -- set interface <vlan> type=internal`  
- list all connections: `nmcli con show`
- describe connections: `nmcli show <connection>`
- set connection up/down: `nmcli con down/up <connection>`
- edit a configuration via cli tool: `nmcli edit <connection>`
  - *for example you can set the ip  after you enter the menu like this:*
    - `set ipv4.address 192.168.1.1`
    - `save`
    - `quit` 
- edit a configuration using a single command:
 - `nmcli connection modify ovs-slave-<ovs_bridge> ipv4.method static ipv4.address <ipv4_address> ipv4.gateway <ipv4_gateway>`
    - here we are setting the ip of the ovs-bridge interface
##### Switch between OVS-Bridge and your standart Connection without loosing ssh-connection
###### turn of ovs-bridge
```bash
nmcli con down ovs-slave-ovsbr ovs-bridge-ovsbr && nmcli con up enp2s0
```
###### turn on ovs-bridge
```
nmcli con up ovs-slave-ovsbr ovs-bridge-ovsbr && nmcli con down enp2s0
```
#### Container and vlan-aware networks
This is actually quite easy, all you need to do is to attach a maclavan network to the container.<br>
Keep in mind that docker can duplicate/spoof existing ip addresses if you use the default settings.<br>
In my case, i use static ipam so, ill just add ip adresses to the containers directly, or maybe i will use my routers dhcp in the future, but i leave it up to you to take care of your approach.

- create and attach a maclavan network to a vlan interface
   ```bash
   $ docker network create -d macvlan -o parent= <vlan interface> --subnet <subnet> <container> 
   ```
- start a container using your docker network
   ```bash
    docker run --network <maclavan network> --cap-add=NET_RAW -dit <container image>
   ```
   - `--cap-add=NET_RAW` allows ping
   - `-dit` attaches a TTY
- start a shell and test it out
  ```bash
  docker exec -it <container> /bin/bash
  ```
- inspect, list and check if network is existing: `docker network inspect <network>`, `docker network ls`, `docker network exists <network>`   
#### Create libvirt vlan-aware virtual network
You can either use the cli or the virt-manager UI.<br>
However you will need to edit the XML manually in virt-manager, so make sure to allow this in the settings.<br>
<br>
Keep in mind that my collection will do this for you automatically! Anyway, here is how you do it manually!
edit the XML accordingly:
```xml
<network connections="1">
  <name>br0</name>
  <uuid><your uuid></uuid>
  <forward mode="bridge"/>
  <bridge name="<your Virtual Network>" />
  <virtualport type="openvswitch"/>
  <portgroup name="vlan-1" default="yes">
  </portgroup>
  <portgroup name="vlan-2">
    <vlan>
      <tag id="2"/>
    </vlan>
  </portgroup>
  <portgroup name="vlan-all">
    <vlan trunk="yes">
      <tag id="1"/>
      <tag id="2"/>
    </vlan>
  </portgroup>
</network>
```
#### Add the virtual network to your VM
Add the Network in your VM Configuration and edit the XML accordingly:
```xml
<interface type="bridge">
  <mac address= "<your mac>" />
  <source bridge= "<your Virtual Network>"/>
  <virtualport type="openvswitch">
    <parameters interfaceid= "<your interface id>" />
  </virtualport>
  <model type="virtio"/>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x0"/>
</interface>
```


##### sources
- openvswitch: [vlans](https://docs.openvswitch.org/en/latest/faq/vlan/)
- openvswitch: [isolate vm traffic](https://docs.openvswitch.org/en/latest/howto/vlan/)

#### Create OVS-Bridge and tagged Vlans using my [Ansible Collection](https://galaxy.ansible.com/ui/repo/published/ji_podhead/ovs_bridge/)

##### Install my Collection
```bash
 ansible-galaxy collection install ji_podhead.ovs_bridge
```
##### Install Dependencies
```
- name: Ensure Open vSwitch is installed
    ansible.builtin.dnf:
    name: openvswith
    state: present

- name: Start Open vSwitch service
    systemd:
    name: openvswitch.service
    state: started
    enabled: yes

- name: install NetworkManager.ovs
    ansible.builtin.dnf:
    name: NetworkManager.ovs
    state: present
```
##### Create the Bridges and VLAN's

```yaml
---
- hosts: <your host>
  gather_facts: no
  become: true
  become_method: sudo
  collections:
    - ji_podhead.ovs_bridge
  vars:
    physical_nic: enp2s0
    interface_port: interface_port
    ovs_bridge_connection: vlanbr
    connection_port: connection_port
    ovs_bridge_interface: vlanbr # the name of the interface of our ovs-bridge
    ipv4_address: 192.168.1.100/24 # the ip of the ovs-bridge
    ipv4_method: static
    ipv4_gateway: "192.168.1.1" # my router
    autoconnect: "yes"
    vlans:
          - interface: vlan1
            master: vlanbr
            tag: 1
            port: vlan-port-1
            connection: vlan-con-1
            ipv4_method: static
            ipv4_address: "192.168.7.1/24"
          - interface: vlan2
            master: vlanbr
            tag: 2
            port: vlan-port-2
            connection: vlan-con-2
            ipv4_method: static
            ipv4_address: "192.168.8.1/24"
  
  tasks:

- name: create bridge, interface & ports
    import_role: 
      name: ji_podhead.ovs_bridge.create_bridge

  - name: add vlans
    import_role: 
      name: ji_podhead.ovs_bridge.add_vlans

# now we will add the physical nic to our bridge, start the ovs-slave connection 
# we also give our vlanbr-interface an ip
# no reboot required and you can directly continue with ansible playbooks
  - name: add nic
    import_role:
      name: ji_podhead.ovs_bridge.add_nic_to_bridge

```
##### Check the devices and connections of your ovs-bridge and tagged vlans 
```bash
$ nmcli con show
```
  - ```yaml 
      NAME                          UUID                                  TYPE           DEVICE            
    ovs-slave-vlanbr              2fa5810e-394d-4f76-852e-b8829d2adacb  ovs-interface  vlanbr            
    vlan-con-1                    2c016716-11d9-4ec1-a799-0699d7670f2f  ovs-interface  vlan1             
    vlan-con-2                    5d2904c1-5198-4e99-bd92-6c9e2ec0f09d  ovs-interface  vlan2             
    ovs-bridge-vlanbr  329cedba-8796-4ab1-b0a4-ce2d4daf3f69  ovs-bridge     vlanbr 
    ovs-slave-connection_port     195530fc-5c86-4f57-b7e7-0d516a0cc49e  ovs-port       connection_port   
    ovs-slave-enp2s0              84f158db-0d2d-4f6b-b4c6-3e12e69b57c2  ethernet       enp2s0            
    ovs-slave-interface_port      759d1d97-444d-45d2-baa9-feb4227a6831  ovs-port       interface_port    
    vlan-port-1                   b6d2af7b-e394-4f36-a99c-57d8da816fa0  ovs-port       vlan1             
    vlan-port-2                   87570fa9-46aa-4389-9ca7-5ec4a95372d0  ovs-port       vlan2             
    lo                            09d065e6-5a6f-4886-bbca-fb612df086d8  loopback       lo    
    ```

```bash
$ ifconfig
```
  - ```yaml
    enp2s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
          ether 40:b0:76:46:6f:76  txqueuelen 1000  (Ethernet)
          RX packets 2665  bytes 930652 (908.8 KiB)
          RX errors 0  dropped 0  overruns 0  frame 0
          TX packets 2084  bytes 261309 (255.1 KiB)
          TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
          device interrupt 16  

    lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
            inet 127.0.0.1  netmask 255.0.0.0
            inet6 ::1  prefixlen 128  scopeid 0x10<host>
            loop  txqueuelen 1000  (Lokale Schleife)
            RX packets 980  bytes 230168 (224.7 KiB)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 980  bytes 230168 (224.7 KiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    vlan1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 192.168.7.1  netmask 255.255.255.0  broadcast 192.168.7.255
            inet6 fe80::c63:1c49:cc21:35be  prefixlen 64  scopeid 0x20<link>
            ether 82:d9:0f:d8:8a:3a  txqueuelen 1000  (Ethernet)
            RX packets 0  bytes 0 (0.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 29  bytes 3580 (3.4 KiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    vlan2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 192.168.8.1  netmask 255.255.255.0  broadcast 192.168.8.255
            inet6 fe80::e411:ca06:2969:2bc2  prefixlen 64  scopeid 0x20<link>
            ether be:ea:5e:df:e9:ce  txqueuelen 1000  (Ethernet)
            RX packets 0  bytes 0 (0.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 31  bytes 3655 (3.5 KiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    vlanbr: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 192.168.1.100  netmask 255.255.255.0  broadcast 0.0.0.0
            inet6 fd24:25eb:4a0::d79  prefixlen 128  scopeid 0x0<global>
            inet6 fd24:25eb:4a0:0:7d7a:5a93:c7fe:258a  prefixlen 64  scopeid 0x0<global>
            inet6 fe80::e046:4028:1b5d:9608  prefixlen 64  scopeid 0x20<link>
            ether 22:6f:9a:b9:82:76  txqueuelen 1000  (Ethernet)
            RX packets 209  bytes 20589 (20.1 KiB)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 189  bytes 33714 (32.9 KiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
    ```

#### opensense with openvswitch and tagged vlans
i tried to get openwrt to run with vlans and a single nic. i looked up the manual and it says that you can just add a bridge, then activate the vlans, but my dhcp did not respond.
I checked the traffic using wireshark and the packets did indeeed arrive, but no response, so now ill just use my opensense vm for dhcp.

- create a vlan trunk including all your vlans that need dhcp, or should get routed
- add both the the default and our trunk to our opensense vm
- we wont use the trunk for the other vms to seperate the layer2 traffic
- set up the interfaces and dhcp in opensense
  - use dhcp for the main lan ip, so we get an ip from our router 
  - activate the trunk interface - no ip needed
  - create the vlans (Interfaces -> other types -> vlans )
  - assign interfaces to the vlans ( Interfaces -> Assignments )
  - add a *pass firewall rule* ( firewall -> rules )
  - activate the vlan interfaces and assign a static ip
  - go to the dhcp of your vlans, assign a subnet and start the service

 - after we allow traffic in our trunk port we should now see traffic in opensense when we restart the NetworkManager or use the Broadcast Address
  > ![image](https://github.com/user-attachments/assets/346e4ea9-d8e6-479b-9448-df531f341042)
  > i assigned an interface with vlan tag2 via  libvirt
  > notice that i used `vlan.id == 2` filter in wireshark

- our vlan2 dhcp will respond and assign a ip
>   ![image](https://github.com/user-attachments/assets/114ef152-53c9-460c-bc94-9fd2fcf08bbc)

---
## Enter the matrix
when your bored and listening to nice music, try this:
```bash
$ find / -type f -exec hexdump -C {} \;
```
   - techno kommando

## Zero Trust Access
if you dont use ssh password, nor key for auth, then itsa possible that somebody can do arp spoofing and listen for your pub key and gains access to ALL machines that you sign in to.
So  we dont wanna do that and we also dont want to sign eevery frkn ssh connection, so we need some kinda ssh  manager...
so it turns out the best way  to secure your ssh connections is to use no sshd at all.
yup makes sense, but what do we use instead? - a tool called zero trust access that does risk bases authentication for you, so we have a bunch of (risk based) factors  that are relevant not just your ssh keys, or passwords.
So the way that works: 
   - instead of logging to the device directly, you have some sort of ha ssh proxy that is your central ssh service
   - zero trust access manager requires oauth and 2factor

## Synchronous vs. Asynchronous Encryption

### Synchronous Encryption (Symmetric)

- Same key for encryption and decryption
- Fast and efficient
- Example: AES (Advanced Encryption Standard)

### Asynchronous Encryption (Asymmetric)

- Different keys for encryption and decryption (public and private keys)
- More secure, but slower
- Example: RSA (Rivest-Shamir-Adleman)

***Example:***
   - Alice wants to send a secret message to Bob.
   - Alice encrypts the message using Bob's public key.
   - Alice sends the encrypted message to Bob.
   - Bob receives the encrypted message and decrypts it using his private key.

---

## SSH and Signed SSH keys using CA
- ssh uses async. encryption:
  - The private key is stored on the client-side and is used to encrypt the authentication data. The private key is never shared with the server.
  - The public key is stored on the server-side and is used to decrypt the authentication data. The public key is shared with the client.

### CA-Signed Public Key

When using a Certificate Authority (CA) to sign the public key, the client sends both the private key and the CA-signed public key to the server. The CA-signed public key is used to verify the client's identity, while the private key is used to encrypt the authentication data.

### Server-Side Verification

When the client sends the CA-signed public key to the server, the server uses the CA public key to verify the signature. The CA public key is stored on the server-side in a file specified by the TrustedUserCAKeys option in the SSHD configuration file (/etc/ssh/sshd_config).

```
# /etc/ssh/sshd_config
...
TrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem
```
In this case, the server uses the CA public key from the file /etc/ssh/trusted-user-ca-keys.pem to verify the signature of the CA-signed public key.

If the signature is valid, the server accepts the client's authentication. If the signature is invalid, the server rejects the client's authentication.

- ***Why Send Both Keys?***
  - The client sends both the private key and the CA-signed public key to the server because the server needs to verify the client's identity using the CA public key.
  - The private key is used to encrypt the authentication data, while the CA-signed public key is used to verify the client's identity.

---

## push gh actions workspace to docker hub
```bash
    steps:
      - name: Check out the repo
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@f4ef78c080cd8ba55a85445d5b36e214a81df20a
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@9ec57ed1fcdbf14dcef7dfbe97b2010124a938b7
        with:
          images: my-docker-hub-namespace/my-docker-hub-repository

      - name: Build and push Docker image
        id: push
        uses: docker/build-push-action@3b5e8027fcad23fda98b2e3ac259d8d67585f671
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: jipodhead/ilohelper-coll:latest
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
          
```
## Jenkins & Github
***jenkins has a weird way of signing in to git.***
- 1. create a pat via dev settings in git
  2. add ajenkins credential
    - ***you need to select  to select `USERNAME+PASSWORD`***
    - add your git user name  and paste the pat into the password field
    - ***DO NOT USE SSH + PASSWORD***
--- 
## Libvirt
- install
  ```bash
     su root
     dnf install qemu-kvm libvirt virt-install virt-viewer
     for drv in qemu network nodedev nwfilter secret storage interface; do systemctl start virt${drv}d{,-ro,-admin}.socket; done
  ```
- create volume in default storage pool
  ```bash
  sudo qemu-img create -f qcow2 /var/lib/libvirt/images/your_volume.qcow2 2G
  ```
- use your volume
  ```xml
	 <devices>
	    <emulator>/usr/libexec/qemu-kvm</emulator>
	    <disk type='file' device='disk'>
	      <source file='/var/lib/libvirt/images/your_volume.qcow2'/>
	      <target dev='vda' bus='virtio'/>
	    </disk>
	    # ... more devices
	 </devices>
  ```
  
- boot from iso (virtual cd reader)
  ```xml 
     <devices>
	<disk type='file' device='cdrom'>
		<driver name='qemu' type='raw'/>
	        <source file='/var/lib/libvirt/images/ubuntu-24.04-live-server-amd64.iso'/>
	      	<target dev='hdb' bus='ide'/>
	        <readonly/>
	</disk>
        # ... more devices
  </devices>    
  ```

   - note that you have to use `machine='pc'` in the os section of your xml for debian and you also need to specify the boot device, which is our cdrom/hd.
     - if the vm cant find any bootable os in our disk, it will look in cdrom, so you can set `boot dev='hd'`
     - this is how it could look like:
     ```xml
	  <os>
	    <type arch='x86_64' machine='pc'>hvm</type>
	    <boot dev='hd'/>
	  </os>
      ```
		    
## Ansible Variables, Secrets and Vault
this is how you can use variables and secrets in ansible:
### 1. Variables as Constructor argument
You can pass secrets/variables to ansible using the -e flag (eg `-e var=<value>`).
  - example:
    ```bash
       ansible-playbook ansible/proxmox_playbook.yml -i ansible/inventory.yml  -e vault_token=<yout token> -vvv
    ```
### 2. Environment Variables
 
- create env. variable manualy
     ```bash
     	export variable = value
     ```
- use [git actions environment variables](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/variables#defining-environment-variables-for-a-single-workflow) and just reference them in ansible
  - this is how you can create them in a workflow:
  ```yaml
	  jobs:
	  greeting_job:
	    runs-on: ubuntu-latest
	    env:
	      Greeting: Hello
	    steps:
	      - name: "Say Hello Mona it's Monday"
	        run: echo "$Greeting $First_Name. Today is $DAY_OF_WEEK!"
	        env:
	          First_Name: Mona
  ```
### 3. [ansible vault](https://docs.ansible.com/ansible/latest/cli/ansible-vault.html#ansible-vault)
all this does is to encrypt your inventory file (where you usualy store your hosts and ssh keys), or other files.
- usage:
   ```bash
   $ ansible-vault encrypt inventory.yml
   ```
   - the output will look like this:
   ```bash
   $ANSIBLE_VAULT;1.1;AES256
   6531633239353231303063613464323531643933613336353130383837623537663537343033633
   ...
   ```
   
### 4. Vault (key/value)	
you can use the community.hashi_vault plugin to fetch secrets from vault using lookups.
   - this is how you use lookup:
     ```yaml
       "{{ lookup('community.hashi_vault.vault_kv2_get', 'dir/subdir' , engine_mount_point='keyvalue', url='http://127.0.0.1:8200',  token=vault_token)['data']['data']['<your-secret-key>'] }}"
     ```
<br> However you need to do this in a playbook. 
   - ***You cant put the lookups diretly into the inventory***
   - Even if you can run your script and you have no errors, the variables will remain unknown and lookups wont fire.


    
#### ***Store your secrets in the inventory dynamicaly***
Before you can use your Hosts and do any ssh connection, you need to fetch your secrets  and store them in the inventory dynamically:

- 1. ***create an inventory*** (the placeholder variables are just for prototyping)
     ```yaml
	   workshop_machines:
	  hosts:
	    dcworkshop1:
	      vars:
	        ansible_ssh_host: "placeholder" 
	        ansible_user: "placeholder"
	        ansible_ssh_pass: "placeholder"
	        ansible_connection: ssh
	        ansible_become_password: "placeholder"
      ```
 - 2. ***Fetch your Vault Secrets and store them using set_fact method***
	     ```yaml
		---
		- hosts: workshop_machines
		  gather_facts: no
		  become: true
		  become_method: sudo
		  become_user: root
		  tasks:
		    - name: gethosts
		      debug:
		        var: inventory_hostname
		    - name: get_secrets_from_vault
		      set_fact:
		          proxmox_user_secret: "{{ lookup('community.hashi_vault.vault_kv2_get', 'ansible/proxmox/' + inventory_hostname + '/' , engine_mount_point='keyvalue', url='http://127.0.0.1:8200',  token=vault_token) }}"
		
		    - name: Get secrets from Vault for each host
		      set_fact:
		        ansible_ssh_host: "{{ lookup('community.hashi_vault.vault_kv2_get', 'ansible/proxmox/'+ inventory_hostname, engine_mount_point='keyvalue', url='http://127.0.0.1:8200', token=vault_token)['data']['data']['ip'] }}"
		        ansible_user: "{{ lookup('community.hashi_vault.vault_kv2_get', 'ansible/proxmox/'+ inventory_hostname, engine_mount_point='keyvalue', url='http://127.0.0.1:8200', token=vault_token)['data']['data']['user'] }}"
		        ansible_ssh_pass: "{{ lookup('community.hashi_vault.vault_kv2_get', 'ansible/proxmox/'+ inventory_hostname, engine_mount_point='keyvalue', url='http://127.0.0.1:8200', token=vault_token)['data']['data']['pass'] }}"
		        ansible_become_password: "{{ lookup('community.hashi_vault.vault_kv2_get', 'ansible/proxmox/'+ inventory_hostname, engine_mount_point='keyvalue', url='http://127.0.0.1:8200', token=vault_token)['data']['data']['pass'] }}"
	     ```
---
## Ansible & libvirt
- ***we need `become: true` here, because libvirt requires root***
- in this step we will just create a config for each host(could also be hostgroups)
  - in the next step will use roles and jinja templates instead 
### 1. install
  ```yaml
  - hosts: workshop_machines
  gather_facts: no
  become: true
  become_method: sudo
  become_user: root
  tasks:
    - name: install software
      block:
      - name: Install wget
        ansible.builtin.dnf:
          name: wget
          state: present
      - name: Install libvirt
        ansible.builtin.dnf:
          name: libvirt
          state: present
   ```
### 2. create a vm preset
   - ***it needs to be a *.xml.j2 file***
   - make sure to use the right emulator path
     - i had libvirt & kvm installed on another system, so i just looked up the xml in of an existing vm, using the qemu/kvm manager   
   - also make sure to copy the images to the libvirt image path, but libvirt can have trouble reading your user files
     - we we will do this in our ansible file in the next step 
   ```xml
	<domain type='kvm'>
	  <memory>131072</memory>
	  <vcpu>1</vcpu>
	  <os>
	    <type arch="i686">hvm</type>
	  </os>
	  <clock sync="localtime"/>
	  <devices>
	    <emulator>/usr/libexec/qemu-kvm</emulator>
	    <disk type='file' device='disk'>
	      <source file='/var/lib/libvirt/images/proxmox-ve_8.2-1.iso'/>
	      <target dev='hda'/>
	    </disk>
	    <interface type='network'>
	    <mac address="52:54:00:62:f8:25"/>
	      <source network='default'/>
	    </interface>
	    <graphics type='vnc' port='-1' keymap='de'/>
	  </devices>
	</domain>
   ```
### 4. get your iso
   - im just downloading proxmox here using wget
     -  but you can of course use docker hub, or your self hosted registry (gitea, or gitlab). not sure if it makes sense to use Argo here though
      
     ```yaml
	      - name: get_proxmox_iso
	        ansible.builtin.shell: |
	          cd /var/lib/libvirt/images/
	        # wget -c https://enterprise.proxmox.com/iso/proxmox-ve_8.2-1.iso 
	        args:
	          chdir: ./
      ```    
### 3. create and update vms
   - if you dont want to update your configuration and infra, you dont need the destroy job
   - notice that you have to define a vm before you can create it.
   - i use a config file that is stored under `./config/libvirt/<hostname>.xml.j2`
     - you can of course just use lookup, but i wanted to have multiple presets for different hosts, or groups
       
     ```yaml
          - name: create vms
	      block:
	        - name: destroy VM
	          ignore_errors: true
	          community.libvirt.virt:
	            name:  "{{ inventory_hostname }}_proxmox" 
	            command: destroy
	            autostart: true
	            xml: "{{ lookup('template', '../config/libvirt/' + inventory_hostname + '.xml.j2') }}"
	        # defining and launching an LXC guest
	        - name: define VM
	          community.libvirt.virt:
	            name: "{{ inventory_hostname }}_proxmox"
	            command: define
	            autostart: true
	            xml: "{{ lookup('template', '../config/libvirt/' + inventory_hostname + '.xml.j2') }}"
	        - name: Create VM
	          community.libvirt.virt:
	            name: "{{ inventory_hostname  }}_proxmox"
	            command: create
	            autostart: true
	            xml: "{{ lookup('template', '../config/libvirt/' + inventory_hostname + '.xml.j2') }}"
	        - name: Get VMs list
	          community.libvirt.virt:
	            command: list_vms
	          register: existing_vms
	          changed_when: no
      ```
---

## Ansible roles

| [Ansible Roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)| [build-VM-fast-ansible](https://www.redhat.com/sysadmin/build-VM-fast-ansible) | [ansible-templates-configuration](https://www.redhat.com/sysadmin/ansible-templates-configuration) |
- roles let us use jijna templates **-> the files have a `*.j2` suffix**
- we can use ansible variables in our template files, so we dont need a file for each host, or change the file dynamically according to the host specific values (like hostname)
### Example
- lets say we want to change the name attribute for each host in the vm preset xml
- without roles and jina templates we need to use sed(regex) for each value:
   ```bash
   	$ sed -i "s|<name>[^>]*</name>|<name>$vm_name</name>|" $xml_path
   ```
- when using roles and jinja templates, we can use our ansibles variables directly in the xml preset:
   ```xml
   	<name>{{ vm_name }}</name>
   ```
### Create a Role 
- go to your playbook folder
- initialize a role
  ```bash
     $ ansible-galaxy role init kvm_provision
  ```
- remove the files that are not required
   ```bash
   $ rm -r files handlers vars
   ```
- add path to ansible.cfg (not sure if required)
  - ansible should usualy look for a roles folder in your playbookfolder
  ```yaml
  roles_path = ./roles:./playbooks/roles
  ``` 
- create defaults in `kvm_provision/defaults/main.yml`
  ```yaml
	  ---
	# defaults file for kvm_provision
	base_image_name: Fedora-Cloud-Base-34-1.2.x86_64.qcow2
	base_image_url: https://download.fedoraproject.org/pub/fedora/linux/releases/34/Cloud/x86_64/images/{{ base_image_name }}
	base_image_sha: b9b621b26725ba95442d9a56cbaa054784e0779a9522ec6eafff07c6e6f717ea
	libvirt_pool_dir: "/var/lib/libvirt/images"
	vm_name: f34-dev
	vm_vcpus: 2
	vm_ram_mb: 2048
	vm_net: default
	vm_root_pass: test123
	cleanup_tmp: no
  ```
- create your jinja template under `<your role>/templates`
  - i used the one from the [redhat blog](https://www.redhat.com/sysadmin/build-VM-fast-ansible)
  - i named it `kvm.xml.j2`
  
- create a task in `<your role>/tasks/main.yml`
   ```yaml
	---
	# tasks file for kvm_provision
	- name: gethosts
	  debug:
	    var: inventory_hostname
	- name: Get VMs list
	  become: true
	  become_method: sudo
	  become_user: root
	  community.libvirt.virt:
	    command: list_vms
	  register: existing_vms
	  changed_when: no
   ```
- ***reference and use your role in your playbook***
  - you can reference the role in the play directly
    - *this will fire the taks in the role before the tasks in the play*
    ```yaml
	 - hosts: dcworkshop1
	  gather_facts: no
	  become: true
	  become_method: sudo
	  become_user: root
	  roles:
	    - kvm_provision
     ```
  - you can also reference the role in a single task
    ```yaml
          - name: single task with role
            include_role:
                name: kvm_provision
    ```
---
   
## compare commits in Gitub Actions
```yaml
	- name: Checkout repository
        uses: actions/checkout@v2
      
      - name: Get current commit ID
        run: echo "COMMIT_ID=$(git rev-parse HEAD)" >> $GITHUB_ENV

      - name: Get previous commit ID
        run: |
          PREV_COMMIT_ID=$(git rev-list --max-parents=0 HEAD | head -n 1)
          echo "PREV_COMMIT_ID=$PREV_COMMIT_ID" >> $GITHUB_ENV
```
- the commit ids are now also environment variables:
   ```bash
	  shell: /usr/bin/bash -e {0}
	  env:
	    COMMIT_ID: 350add3930d76dbc4abc0f69d9eb650e7159cbbc
	    PREV_COMMIT_ID: 350add3930d76dbc4abc0f69d9eb650e7159cbbc
   ```
---
## Cache & Containers in Github Actions 

It can take quite a long time to install python and a single 40mb package in a workflow running on git.
In Some situations that can be quite anyoing, for example when you just want to deploy a static website, or a package that is in development stage.
  - in my case, I wanted to create a workflow that automatically publishes to anisble galaxy and that are more than 4 commands to test the results and i was to lazy to set up a debugger
     
- We can spare time by caching pip packages, or build containers from the workflow workspace
### Caching pip packages
- the packages get cached but net to get reinstalled though
   ```yaml
	name: Publish Ansible Galaxy Collection
	on:
	  push:
	    branches:
	      - main 
	jobs:
	  job1:
	    runs-on: ubuntu-latest
	    steps:
	      - name: Cache Python dependencies
		uses: actions/cache@v2
		with:
		  path: ~/.cache/pip
		  key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
		  restore-keys: |
		    ${{ runner.os }}-pip-
	      - name: Install dependencies
		run: |
		  pip install ansible[galaxy]
    ```
	- output:
	   ```bash
	   	✅ Cache Python dependencies
		------------------------------
		Run actions/cache@v2
		Cache not found for input keys: Linux-pip-, Linux-pip-
		20s
	   	✅ Install dependencies
		------------------------------
		Run pip install ansible[galaxy]
		Defaulting to user installation because normal site-packages is not writeable
		Collecting ansible[galaxy]
		  Downloading ansible-10.2.0-py3-none-any.whl (48.2 MB)
		...
		Successfully installed ansible-10.2.0 ansible-core-2.17.2 resolvelib-1.0.1
		2m 14s
	      	✅ Post Cache Python dependencies
		------------------------------
		Post job cleanup.
		/usr/bin/tar --posix -z -cf cache.tgz -P -C /home/runner/work/ilohelper-collection/ilohelper-collection --files-from manifest.txt
		Cache Size: ~44 MB (45745781 B)
		Cache saved successfully
		Cache saved with key: Linux-pip-
	   ```
         - notice that ***gh actions automatically created Post Cache Python dependencies stage for us***
- you can limit the cache job, so it only gets fired when the requierements have really changed 
   ```yaml
	jobs:
	  check-changes:
	    runs-on: ubuntu-latest
	    outputs:
	      changed: ${{ steps.check.outputs.changed }}
	    steps:
	      - name: Checkout repository
	        uses: actions/checkout@v2
	      - name: Check if requirements have changed
	        id: check
	        run: |
	          CHANGED=$(git diff --name-only HEAD^ HEAD | grep -q 'requirements.txt' && echo "true" || echo "false")
	          echo "::set-output name=changed::$CHANGED"
	
	  cache-and-install:
	    needs: check-changes
	    if: needs.check-changes.outputs.changed == 'true'
	    runs-on: ubuntu-latest
	    steps:
	      - name: Checkout repository
	        uses: actions/checkout@v2
	      - name: Cache Python dependencies
	        uses: actions/cache@v2
	        .... # the rest
	      - name: Install dependencies
	        run: |
	          pip install ansible[galaxy]
     ```

---
## Debug Github Actions in VS Code
### commit and trigger the workflow with one command in vs code terminal
```bash
git add . && git commit -m "Your Commit-Message" && git push 
```

### use the extension to open your workflow with 2 clicks
- refresh the window
- open the created workflow in your browser
  
![extensions](https://github.com/ji-podhead/DevOps/blob/main/docs/debug_actions.png?raw=true)


- additioinaly you can use multiple plugins that seem to wrap some api like around the workflow (like [vs-code-server-action](https://github.com/marketplace/actions/vs-code-server-action), or run your workflow locally using something like [GAL ](https://marketplace.visualstudio.com/items?itemName=alphaxek.github-actions-locally).
  - but those plugins seem somewhat overkill/to hard to install when you just want to have some quick edits
  - GAL even requires docker and the other one runs in codespace
- so i figured it's just easier to use the basic actions extension
  - it kinda sucks that the link to your workflow uses the browser instead of vscode webview pane



---

## Github Organisation Access Tokens (Pats)
this is kinda confusing at first:
  - 1. you have to allow the creation of pats in the organisation settings
  - 2. go to your ***personal*** developer settings where you usually create access keys
    - create a new pat, but choose your organisation as the target organisation/account instead of your profile
  - 3. go back to your organisation settings and allow the pat
    - only if the user that requested the pat is not admin and you have set *Require administrator approval* to true

--- 
## clone another (private) repo in gh actions using checkout
when you clone another repo into your current branch in gh actions and your token is used  somewhere in your workflow for the current scope: 
  - ***you have to use a second token for the private repo***
    
this can be confusing, so make sure to [check out the docs](https://github.com/actions/checkout?tab=readme-ov-file#checkout-multiple-repos-private)

```yaml
jobs:
  merge_from_private:
    runs-on: ubuntu-latest
    permissions: write-all 		# not recommended, just for testing!!!!
    steps:
      - name: Checkout					# <-- the-pod-shop/Pod-Shop-App-Configs 
        uses: actions/checkout@v4
        with:
          path: main
      
      - name: Checkout another private repository      
        uses: actions/checkout@v4
        with:
          repository:  the-pod-shop/App-Configs-private # <-- the-pod-shop/App-Configs-private
          token: ${{ secrets.ACTIONS_KEY }}             # <---other scope---.		    .
          path: main							   #|
#									   #|
#							                   #|
      - name: Push to target repo					   #|
        uses: ad-m/github-push-action@master				   #|
        with:								   #|
          github_token: ${{ secrets.API_TOKEN_GITHUB }} # <--current scope--*
          repository: the-pod-shop/Pod-Shop-App-Configs # <-- the-pod-shop/Pod-Shop-App-Configs
          branch: merge_private
          directory: ./
```

---

## how can sensible data get leaked on github trough terraform?
I noticed that under some circumstances you can leak your secrets and it allmost happen to me. There are some protection measures like git warning you about exposing secrets when trying to push. However you can still expose data that might not get registered as secret (vulnerable ips, ports, or code that was never meant to be public), or secrets will simply not getting registered as such.<br>
****possible outcome:****
  - phishing
  - ddos
  - enumeration
  - social engineering
  - many many more
    
**Heres how i can proove it:**
> ***tests fail in a public repo, but the commit is still publicly available***

- write a bash script, call it with local_exec resource and `echo <secret> > file.txt`
  - if you forget to encrypt this file, or did'nt put it on .gitignore:
    - ***your created file will get exposed on github***
- use data instead of variable to get vault secrets
  - if you accidentaly pull w and then push back with no `.gitignore` included in the branch:
    - ***your entire tfstate file and all of your files you have in .gitignore will get exposed on github***
- you renamed your tfstate file, or gave it another filetype

those are just 3 examples, but the tragic reality is that whoever contributes to your public repo can fuck things up if you dont take care about this
>  1. someone exposes and iprange of your acl to a public exposed server
>     - eg. tailscale funnel + web hook-, dns-, or whatever server
>  2. someone created an environment variable for a password instead of sensible variable, or he exposes data that you dont scan detect, but it's a risk for him
### zero trust approach for ica configs and public git repos
Luckily we have a few options that gives us a 99% chance, that no sensible data will never get exposed (if we do t right) <br>
Here are my ideas:
1. ***encrypt your files before push*** `not very safe`
   - a script that you call before even commiting (or with prereceive hooks) can scan for vulnerable data/code and encrypt your code/data
   - **Downside:** you dont know if its ever get called *(maybe somebody just ignores to call it, or he simply just forgot about it)*  
2. ***use a private repo for iac configs*** `safe`
   - you can mirror to a public repo using a  bot (eg. github actions) once you approved and your static tests are all passed
   - ~you cant have branch protection rules for a private repo on gh free plan however~
     - alternatives:
       - *gitea/foregejo:* free but selfhosted
       - *gitlabs:* free but limited to 2000 compute minutes
3. ***use prereceive hooks and github enterprise server*** `safe`
   - github prereceive hooks let us block an incomming push and create a pr in a public repo (code is sealed)
     - we delete the workspace folder but push into our private repo before. 
   - you can review after testing and merge back into the public repo
     - this way we can seal the code in the public repo until approved and our bot will do the rest 
4. ***github enterprise secret scanning*** `not very safe`
   - its cool that we have that feature, but lets be honest:
     - vulnerable ips, hostnames, emails, or usernames are not getting scanned here, so this requires additional review or manual scanning 
---

## tailscale and github action 


***related tailscale docs*** <br>
| [tailscale & github actions](https://tailscale.com/kb/1276/tailscale-github-action#how-it-works) | [oauth clients](https://tailscale.com/kb/1215/oauth-clients) | [ACL-tags](https://tailscale.com/kb/1068/acl-tags) |

---

- ***1. add a oauth client in tailscale***

---

- ***2. create at least one ACL tag in tailscale*** 
    ```yaml
  ...
  // 	],
	// },
	"tagOwners": {
	"tag:ci": ["ji-podhead@github"],
	},
	// Define users and devices that can use Tailscale SSH.
	"ssh": [
  ...
  ```    
  - without `tags: tag:ci` we get this error:
      - ```err 
        "::error title=⛔ error hint::OAuth identity empty, Maybe you need to populate it in the Secrets for your workflow, see more in https://docs.github.com/en/actions/security-guides/encrypted-secrets and https://tailscale.com/s/oauth-clients"
        ```
---

- ***3. add a little server on your local machine***
```python
import http.server
import socketserver

# Pfad zum Verzeichnis, das als Root für den Server dient
PORT = <port>
DIRECTORY_PATH = "/home/ji/Dokumente/python scripts"

class MyHttpRequestHandler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/':
            self.send_response(200)
            self.send_header('Content-type', 'text/html')
            self.end_headers()
            message = f"Hello, this is my custom server running on port {PORT}!"
            self.wfile.write(bytes(message, "utf8"))
        else:
            # Für andere Pfade wird der Inhalt des angegebenen Verzeichnisses zurückgegeben
            super().do_GET()

with socketserver.TCPServer(("", PORT), MyHttpRequestHandler) as httpd:
    print(f"Serving at port {PORT}")
    httpd.serve_forever()
```
  - keep in mind to use iptable/firewalld/etc to enable the port, or just stop the firewall service(*not the best practice*)

---

- ***4. add your job***
```
name: CI
on:
  push:
    branches: [ "testjob" ]					# <<<--- add your branches
  pull_request:
    branches: [ "testjob" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: greetings
        run: |
          echo
          echo "----------------------------------------------------"
          echo "    .------------------------------------.  "
          echo "   |         > running testjob <          |  "
          echo "    °------------------------------------°   "
          echo "----------------------------------------------------"

      - name: Setup Tailscale
        uses: tailscale/github-action@v2
        with:
          oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}	# <<<--- add you tailscale creds to your gh secrets
          oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
          tags: tag:ci                                        	# <<<--- THIS IS IMPORTANT

      - name: ping                                           	# <<<--- LETS PING A SERVER FOR TESTING
        if: always()
        run: |
          curl http://${{ secrets.TAILSCALE_IP }}:${{ secrets.TAILSCALE_SERVER_PORT_1 }}

      #- name: SSH into remote server
      #  run: |
      #    ssh <user>@${{ secrets.TAILSCALE_IP}}
```

---

- ***5 run your job***
```bash
#--------------------------------------------------------------------------------
# ✅ greetings
#0s
#Run echo
----------------------------------------------------
    .------------------------------------.  
   |         > running testjob <          |  
    °------------------------------------°   
----------------------------------------------------
#--------------------------------------------------------------------------------
# ✅ Setup Tailscale
#8s
Run tailscale/github-action@v2
Run if [ X64 = "ARM64" ]; then
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100    64  100    64    0     0    607      0 --:--:-- --:--:-- --:--:--   609
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 22.5M  100 22.5M    0     0  68.4M      0 --:--:-- --:--:-- --:--:-- 68.5M
tailscale.tgz: OK
Run sudo -E tailscaled --state=mem: ${ADDITIONAL_DAEMON_ARGS} 2>~/tailscaled.log &
Run if [ -z "${HOSTNAME}" ]; then
#----------------------------------------------------------------------------------
# ✅ ping 
#0s
Run curl http://***:***
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100    53    0    53    0     0     57      0 --:--:-- --:--:-- --:--:--    57
Hello, this is my custom server running on port ***!
#--------------------------------------------------------------------------------
# ✅ Complete Job
0s
Cleaning up orphan processes
#--------------------------------------------------------------------------------
```
---

## Tailscale API

- get the devices

```bash
$ curl -H "Authorization: Bearer ${var.api_key}" https://api.tailscale.com/api/v2/tailnet/ji-podhead.github/devices > OUTPUTFILE
```
- you have to use  `Bearer` **instead of** `-u`, or you will get prompet to enter the root password of the machine that runs vault
>```bash
> -u <your api key>
>```
- those oauth flagss do not work (neither in terraform, nor in api, even with tags and acl)  
>```bash
>  --data-urlencode 'client_id=<client id> \
>  --data-urlencode 'client_secret=t<client secret>
>```  

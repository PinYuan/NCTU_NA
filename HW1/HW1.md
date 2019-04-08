# HW1

## 事前準備環境

- Router

  - CentOS
  - 安裝OS及網路設定 https://blog.yowko.com/virtualbox-centos-7/

- ClientPC

  - Ubuntu



  Router有兩個網路介面enp0s3、enp0s8：enp0s3內部網路，enp0s8可以上外網

  ClientPC有enp0s3作為內部網路介面，目的和Router enp0s3互聯

## NAT

讓ClientPC透過NAT(Router)連接外網

### IP forward

```
vim /etc/sysctl.conf 

net.ipv4.ip_forward = 1
```

檔案變更生效：`sysctl -p /etc/sysctl.conf `



#### reference

- https://web.mit.edu/rhel-doc/4/RH-DOCS/rhel-sg-zh_tw-4/s1-firewall-ipt-fwd.html
- [Destination Host Prohibited](https://eallion.com/destination-host-prohibited)



### Iptables

- Install - [How to Install Iptables on CentOS 7](https://linuxize.com/post/how-to-install-iptables-on-centos-7/)

- Setting

  ```
  iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE
  
  # insert before <REJECT all -- anywhere anywhere reject-with icmp-host-prohibited>
  iptables -I FORWARD 1 -i enp0s8 -o enp0s3 -m state --state RELATED,ESTABLISHED -j ACCEPT
  iptables -I FORWARD 2 -i enp0s3 -o enp0s8 -j ACCEPT
  ```

- Store and auto-load after boot

  ```
  service iptables save
  ```

- Utilities
  - List iptables `sudo iptables -nvL`
  - List nat rules `sudo iptables -t nat -nvL`	
  - Delete rule `sudo iptables -D <rule>` or `sudo iptables -D [INPUT|OUTPUT|FORWARD] rule-num`
  - Insert rule `sudo iptables -I [INPUT|OUTPUT|FORWARD] index-num <rule>`
    - the number given after the chain name indicates the position **before** an existing Rule.

#### reference

- [Configuring firewalld to act as a router](https://www.centos.org/forums/viewtopic.php?t=53819)
- [NA 作業二](http://imtroller.logdown.com/posts/195909-na-job-ii)
- [How to List and Delete iptables Firewall Rules](https://www.rosehosting.com/blog/how-to-list-and-delete-iptables-firewall-rules/)
- [How to edit iptables rules](https://fedoraproject.org/wiki/How_to_edit_iptables_rules?rd=User_talk:RforlotListing#Inserting_Rules)
- https://zh.wikipedia.org/wiki/Iptables
- [10分鐘學會iptables](- https://gigenchang.wordpress.com/2014/04/19/10%E5%88%86%E9%90%98%E5%AD%B8%E6%9C%83iptables/)

---



## DHCP

- Install `yum -y install dhcp`

- set enp0s3(對內) to fixed IP

  ```
  # vim /etc/sysconfig/network-scripts/ifcfg-enp0s3
  # modify not to set by DHCP and add info
  BOOTPROTO=static
  IPADDR=10.113.74.254
  NETMASK=255.255.255.0
  DNS1=8.8.8.8
  ```

- restart network interface ` ifdown enp0s3 && ifup enp0s3`

- configure the interface on which you want the **DHCP** daemon to serve requests in the configuration file **/etc/sysconfig/dhcpd**. Before doing this, check enp0s3 is connected, `nmcli device show / nmcli conn show`.  

  - `DHCPDARGS="enp0s3"` or
  - follow the steps in  **/etc/sysconfig/dhcpd**

- config file `/etc/dhcp/dhcpd.conf`

  ```
  default-lease-time 600;
  max-lease-time 7200;
  ddns-update-style ad-hoc;
  
  subnet 10.113.74.0 netmask 255.255.255.0 {
      range 10.113.74.100 10.113.74.200;
      option subnet-mask 255.255.255.0;
      option routers 10.113.74.254;
      option broadcast-address 10.113.74.255;
      option domain-name-servers 168.95.1.1, 8.8.8.8;
      option domain-name "localdomain";
  }
  ```

- start DHCP service and auto-start when boot

  ```
  systemctl start dhcpd
  systemctl enable dhcpd
  ```


#### reference

- [How to Setup DHCP Server and Client on CentOS and Ubuntu](https://www.tecmint.com/install-dhcp-server-client-on-centos-ubuntu/)

- [setting-a-dhcp-server-on-centos-7](http://blog.itist.tw/2016/09/setting-a-dhcp-server-on-centos-7.html)



---



### VPN (WireGuard)

- Server info

  Subnet: [10.113.74.0/24](http://10.113.74.0/24)
  Wireguard Private Key: client-private-key
  Wireguard Peer IP: 10.113.0.74
  Wireguard Server: [navpn.nctucs.cc:51820](http://navpn.nctucs.cc:51820/)
  Wireguard Server Public Key: server-public-key
  Wireguard Interal IP: 10.113.0.254 **???**

- make sure you have the **kernel headers** for your current kernel

  ```
  $ sudo yum install kernel-headers-$(uname -r) kernel-devel-$(uname -r)
  ```

- Install WireGuard

  ```
  $ sudo curl -Lo /etc/yum.repos.d/wireguard.repo https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
  $ sudo yum install epel-release
  $ sudo yum install wireguard-dkms wireguard-tools
  $ sudo modprobe wireguard && lsmod | grep wireguard
  ```

- Configuring peers (Client)

  - Format

    ```
    $ ip link add dev <device name> type wireguard
    $ ip address add dev wg0 <internal ip>
    
    $ wg set wg0 listen-port 51820 private-key <client-private-key-file> peer <server-public-key> allowed-ips <server internal ip> endpoint <external ip>
    
    $ ip link set <device name> up
    ```

  - Example

  - add `PersistentKeepalive = 25` in config file or `persistent-keepalive` in command line

    ```
    # Set client network interface
    $ ip link add dev wg0 type wireguard
    $ ip address add dev wg0 10.113.0.74/16
    
    $ wg set wg0 listen-port 51820 private-key <client-private-key-file> peer <server-public-key> allowed-ips 10.113.0.0/16 endpoint navpn.nctucs.cc:51820  persistent-keepalive 25
    
     wg set wg0 listen-port 51820 private-key ~/wireguard/privatekey peer Zk0jz7osZvghNjwEFQysDYX++EbRcjvhMVnYUpFDEVM= allowed-ips 10.113.0.0/16 endpoint navpn.nctucs.cc:51820  persistent-keepalive 25
    
    $ wg
    interface: wg0
      public key: /OmkeuEqzTekCglJ2Afi5fgv4rEh2DsuerxnSGEd+hU=
      private key: (hidden)
      listening port: 51820
    
    peer: <server-public-key>
      endpoint: 140.113.168.176:51820
      allowed ips: 10.113.0.0/16
    
    
    $ ip link set wg0 up
    ```

    

   All the packets destined to `AllowedIPs` will be encrypted with `PublicKey` and sent to `Endpoint`.

- Persistent configuration

  ```
  # store config file
  $ wg showconf wg0 > /etc/wireguard/wg0.conf
  
  # auto-config after startup
  $ wg-quick up wg0  # put in /etc/rc.local
  # or
  $ systemctl enable wg-quick@wg0
  ```

- Firewall setting

  ```
  # Forward packets which target at 10.113.0.0/16 
  iptables -I FORWARD 1 -i enp0s3 -o wg0 -j ACCEPT
  iptables -I FORWARD 1 -i wg0 -o enp0s3 -j ACCEPT
  
  # Make responses on the internal network go through the firewall
  iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
  
  service iptables save
  ```

  



#### Troubleshooting

##### [DKMS module not available](#VPN (WireGuard))

If the following command does not list any module after you installed [wireguard-dkms](https://www.archlinux.org/packages/?name=wireguard-dkms),

```
$ modprobe wireguard && lsmod | grep wireguard
```

or if creating a new link returns

```
# ip link add dev wg0 type wireguard
RTNETLINK answers: Operation not supported
```

you probably miss the linux headers.

These headers are available in [linux-headers](https://www.archlinux.org/packages/?name=linux-headers) or [linux-lts-headers](https://www.archlinux.org/packages/?name=linux-lts-headers) depending of the kernel installed on your system.



#### reference

- [Set up a modern point-to-point VPN with WireGuard (server)](https://www.marksei.com/how-to-vpn-wireguard/)
- https://angristan.xyz/how-to-setup-vpn-server-wireguard-nat-ipv6/
- [WireGuard介绍及客户端使用教程](<https://medium.com/@xtarin/wireguard%E4%BB%8B%E7%BB%8D%E5%8F%8A%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BD%BF%E7%94%A8%E6%95%99%E7%A8%8B-2ae1eb4bf670>)
- [How to communicate](https://www.wireguard.com)
- [Official setting steps](https://www.wireguard.com/quickstart/) 



---

### How to check

- NAT connectivity
  - ping 8.8.8.8 from ClientPC
  - ping www.google.com from ClientPC

- VPN
  - ping 10.113.0.254/10.113.3.254` from Router/ClientPC
- DHCP
  - ping 10.113.x.254 (Router) from ClinetPC
# DHCP

assign static IP addr. to DNS master and slave



- Server

  In `/etc/dhcp/dhcpd.conf`

  ```
  host DNSmaster {
     hardware ethernet 08:00:27:3a:d2:52;
     fixed-address 10.113.74.1;
  }
  host DNSslave {
     hardware ethernet 08:00:27:96:72:0e;
     fixed-address 10.113.74.2;
  }
  ```

  then restart DHCP service

  ```
  systemctl stop dhcpd
  systemctl restart dhcpd
  ```

- Client (內部網路)

  vim `/etc/sysconfig/network-scripts/ifcfg-enp0s3`

  ```
  ONBOOT=yes
  ```

- DNS

  vim `/etc/dhcp/dhcpd.conf`

  注意 DNS 至多三個server，最後一個會被忽略

  ```
  subnet 10.113.74.0 netmask 255.255.255.0 {
      range 10.113.74.100 10.113.74.200;
      option subnet-mask 255.255.255.0;
      option routers 10.113.74.254;
      option broadcast-address 10.113.74.255;
      option domain-name-servers 10.113.74.2 10.113.74.1 168.95.1.1, 8.8.8.8; # add your dns
      option domain-name "localdomain";
  }
  ```

  

### reference

[How to Configure DHCP Server on CentOS/RHEL 7/6/5](https://tecadmin.net/configuring-dhcp-server-on-centos-redhat/)


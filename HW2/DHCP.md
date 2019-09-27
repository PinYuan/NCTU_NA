# DHCP

assign static IP addr.  to LDAP master and slave



- Server

  In `/etc/dhcp/dhcpd.conf`

  ```
  host LDAPmaster {
     hardware ethernet 08:00:27:53:67:58;
     fixed-address 10.113.74.11;
  }
  host LDAPslave {
     hardware ethernet 08:00:27:80:9b:49;
     fixed-address 10.113.74.12;
  }
  ```

  then restart DHCP service

  ```
  systemctl stop dhcpd
  systemctl restart dhcpd
  ```

- Client (內部網路)

  ```
  # vim /etc/sysconfig/network-scripts/ifcfg-enp0s3
  ONBOOT=yes
  ```



### reference

[How to Configure DHCP Server on CentOS/RHEL 7/6/5](https://tecadmin.net/configuring-dhcp-server-on-centos-redhat/)
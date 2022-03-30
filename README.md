# External DNS and DHCP with Satellite

[Tutorial Menu](https://github.com/pslucas0212/RedHat-Satellite-VM-Provisioning-to-vSphere-Tutorial)  


In our previous multi-part tutorial we covered an end-to-end scenario for provisioning RHEL VMs from Satellite to a VMWare cluster.   In that series we had the Satellite installer install and configure both DNS and DHCP services on our Satellite server.  Often you will need to integrate Satellite with an existing DNS and DHCP services in your organization.

In this tutorial we will cover the steps to integrate existing DNS and DHCP services.  We will use the same lab setup from the previous tutorial for our... 

Steps used in installing and configuring DNS and DHCP used for this tutorial are covered in the appendix section of this article

### Satellite DNS Integration
The real work for DNS integration with Satellite is in the setup of our DNS server which is covered in the appendix below.

After you have completed installing, configuring and testing the DNS server, you would run the following satellite-installer command to make the following persistent changes to the /etc/foreman-proxy/settings.d/dns.yml file:
```
# satellite-installer --foreman-proxy-dns=true \
--foreman-proxy-dns-managed=false \
--foreman-proxy-dns-provider=nsupdate \
--foreman-proxy-dns-server="10.1.10.253" \
--foreman-proxy-keyfile=/etc/rndc.key \
--foreman-proxy-dns-ttl=86400
```

Restart the foreman-proxy service:
```
# systemctl restart foreman-proxy
```
      
### Satellite DHCP Integration

## Appendix


### DNS Installation, Configuration and Testing

**Note:** named is running on a RHEL 8.5 server updated in March 2022. For this example the subnet is 10.1.10.0/24 and domain is example.com

### Pre-Reqs
Create a RHEL 8.5 VM to provide DDNS and DHCP services.  The VM was sized with 2 vCPUS, 4GB RAM and 100GB "local" drive.  Note: For this example I have enabled Simple Content Access (SCA) on the Red Hat Customer portal and do not need to attach a subscription to the RHEL repositories.  After you have created and started the RHEL 8.5 VM, we will ssh to the RHEL VM and work from the command line.

For this lab environment I chose ns02.example.com for the hostname of the server. 

Register the Server to Red Hat Subscription Management service.
```
# sudo subscription-manager register --org=<org id> --activationkey=<activation key>
```
You can verify the registration with the following command.
```
# sudo subscription-manager status
```    
#### Enabled repositories  

We will want the following two RHEL 8 repositoires enabled on this system:
- rhel-8-for-x86_64-baseos-rpms
- rhel-8-for-x86_64-appstream-rpms

Verify that repositories are enabled.  
```    
# sudo subscription-manager repos --list-enabled
+----------------------------------------------------------+
    Available Repositories in /etc/yum.repos.d/redhat.repo
+----------------------------------------------------------+
Repo ID:   rhel-8-for-x86_64-baseos-rpms
Repo Name: Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)
Repo URL:  https://cdn.redhat.com/content/dist/rhel8/$releasever/x86_64/baseos/o
           s
Enabled:   1

Repo ID:   rhel-8-for-x86_64-appstream-rpms
Repo Name: Red Hat Enterprise Linux 8 for x86_64 - AppStream (RPMs)
Repo URL:  https://cdn.redhat.com/content/dist/rhel8/$releasever/x86_64/appstrea
           m/os
Enabled:   1
```          

#### Update the RHEL 8.5 VM and finish server setup
Install all the latest patches on your RHEL 8.5 Server VM
```
# sudo yum -y update
Updating Subscription Management repositories.
Red Hat Enterprise Linux 8 for x86_64 - BaseOS   27 MB/s |  45 MB     00:01    
Red Hat Enterprise Linux 8 for x86_64 - AppStre  26 MB/s |  39 MB     00:01    
Dependencies resolved.
...
Complete!
```
 
I would also recommend registering this server to Insights.  
```
# sudo insights-client --enable
```


## Install named and dhcpd

We will install named, the bind utilities, the dns caching sever and dhcpd.
```
# sudo yum -y install bind* caching* dhcp*
...
Complete!
```

Update firewall settings
```
# firewall-cmd \
--add-service dns \
--add-service dhcp
```
Make the firewall changes permanent
```
# sudo firewall-cmd --runtime-to-permanent
```

Verify the firewall changes
```
# sudo firewall-cmd --list-all
```
Setup system Clock with chrony.  I have a local time server that my systems use for synching time.  Type the following command to check the the time synch status.  
```
# chronyc sources -v
```

See this article [DHCP Setup for Satellite](https://github.com/pslucas0212/DHCP-Setup-for-Satellite) for the configuration of DHCP.

### Configuring named

In my example setup I externailize the options and zones information for easier readability.

File Name | Location | Info
----------|----------|------
named.conf | /etc | named configuration file
options.conf | /etc/named | named.conf options information
zones.conf | /etc/named | named.conf zone information
db.10.1.10.in-addr.arpa | /var/named/dynamic | reverse zone file
db.example.com | /var/named/dynamic | forward zone file
named.rfc1912.zones | /etc | Generated by the installation
rndc.key | /etc | Generated first time named is started



#### named.conf example
```
// named.conf

include "/etc/rndc.key";

controls  {
        inet 127.0.0.1 port 953 allow { 127.0.0.1; } keys { "rndc-key"; };
};

options  {
        include "/etc/named/options.conf";
};

include "/etc/named.rfc1912.zones";


// Public view read by Server Admin
include "/etc/named/zones.conf";
```

#### options.conf example
```
directory "/var/named";
forwarders { 10.1.1.254; };

recursion yes;
allow-query { any; };
dnssec-enable yes;
dnssec-validation yes;

empty-zones-enable yes;

listen-on-v6 { any; };

allow-recursion { localnets; localhost; };
```

#### zones.conf example
```
zone "10.1.10.in-addr.arpa" {
    type master;
    file "/var/named/dynamic/db.10.1.10.in-addr.arpa";
    update-policy {
            grant rndc-key zonesub ANY;
    };
};
zone "example.com" {
    type master;
    file "/var/named/dynamic/db.example.com";
    update-policy {
            grant rndc-key zonesub ANY;
    };
};
```

#### Forward zone file - db.example.com
```
$ORIGIN .
$TTL 10800	; 3 hours
example.com		IN SOA	ns02.example.com. root.example.com. (
				12         ; serial
				86400      ; refresh (1 day)
				3600       ; retry (1 hour)
				604800     ; expire (1 week)
				3600       ; minimum (1 hour)
				)
			NS	ns02.example.com.
$ORIGIN example.com.
ds01			A	10.1.10.250
exsi01			A	10.1.10.241
exsi02			A	10.1.10.242
exsi03			A	10.1.10.243
ns01			A	10.1.1.254
ns02			A	10.1.10.253
sat01			A	10.1.10.254
vsca01			A	10.1.10.240
```

#### Reverse zone file - db.10.1.10.in-addr.arpa
```
$ORIGIN .
$TTL 10800	; 3 hours
10.1.10.in-addr.arpa	IN SOA	ns02.example.com. root.10.1.10.in-addr.arpa. (
				12         ; serial
				86400      ; refresh (1 day)
				3600       ; retry (1 hour)
				604800     ; expire (1 week)
				3600       ; minimum (1 hour)
				)
			NS	ns02.example.com.
$ORIGIN 10.1.10.in-addr.arpa.
240			PTR	vsca01.example.com.
241			PTR	exsi01.example.com.
242			PTR	exsi02.example.com.
243			PTR	exsi03.example.com.
250			PTR	ds01.example.com.
254			PTR	sat01.example.com.
253			PTR	ns02.example.com.
ns02			A	10.1.10.253
ds01			A	10.1.10.250
exsi01			A	10.1.10.241
exsi02			A	10.1.10.242
exsi03			A	10.1.10.243
sat01			A	10.1.10.254
vsca01			A	10.1.10.240
```


### Testing updates from a client - The Satellite Server

You will need bind installed to use nsupdate.  Install or update bind-utils on the client Server as needed
```
# yum list installed | grep bind-utils
# yum install bind-utils
 ```   

Copy and prepare the rndc.key from the server running named
```
# scp root@ns02.example.com:/etc/rndc.key /etc/rndc.key
# restorecon -v /etc/rndc.key
# chown -v root:named /etc/rndc.key
# chmod -v 640 /etc/rndc.key
```
Test updates to the forward zone (add -d to nsupdate comand for debug: nsupdate -d -k ...)
```
# echo -e "zone example.com.\n server 10.1.10.253\n update add atest.example.com 3600 IN A 10.1.10.10\n send\n" | nsupdate -k /etc/rndc.key
# nslookup atest.example.com
# echo -e "zone example.com.\n server 10.1.10.253\n update delete atest.example.com 3600 IN A 10.1.10.10\n send\n" | nsupdate -k /etc/rndc.key
```      
Test updates to reverse zone (add -d to nsupdate comand for debug: nsupdate -d -k ...)
```     
# echo -e "zone 10.1.10.in-addr.arpa.\n server 10.1.10.253\n update add 10.10.1.10.in-addr.arpa. 300 PTR atest.example.com\n send\n" | nsupdate -k /etc/rndc.key
# nslookup 10.1.10.10
# dig +short -x 10.1.10.10
# echo -e "zone 10.1.10.in-addr.arpa.\n server 10.1.10.253\n update delete 10.10.1.10.in-addr.arpa. 300 PTR atest.example.com\n send\n" | nsupdate -k /etc/rndc.key
```
****Note**** - Typically the forward and reverse zone files are "permanently" updated around 15 minutes after the DNS update is issued from the client machine.

Assign the foreman-proxy user to the named group manually. 
```
usermod -a -G named foreman-proxy
```

### References
- [How to configure the BIND DNS service](https://access.redhat.com/solutions/40683)
- [How to configure Dynamic DNS Server on AlmaLinux / Rocky Linux](https://www.techbrown.com/configure-dynamic-dns-server-almalinux-rocky-linux/)




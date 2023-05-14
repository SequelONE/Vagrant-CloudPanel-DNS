# Vagrant + CloudPanel + DNS Server

Install and configure guest machines running Ubuntu 22.04 with a DNS server and CloudPanel on a Windows 10 host machine.

## Install 

For comfortable work, we need the following programs and plugin for VirtualBox:

* [Vagrant](https://www.vagrantup.com/downloads)
* [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
* [VirtualBox Extension Pack](https://www.virtualbox.org/wiki/Downloads)
* [Cmder](https://cmder.app/)

> Attention!
> 
> To install vagrant, you need to update Windows 10 to build 2004 or higher. After the update, you need to enable WSL via PowerShell as administrator.

```
wsl --install
```

Download the full version of Cmder and extract it on the C drive to the Cmder folder. You must have a path C:\Cmder (for convenience, you can send a shortcut to Cmder.exe to your desktop). Launching Cmder.

### Setup CloudPanel

1. Clone this repository for example to drive C:\

```
git clone https://github.com/SequelONE/Vagrant-CloudPanel-DNS.git
```

2. In Cmder go to sequel.loc folder:

```
cd C:\Vagrant-CloudPanel-DNS\sequel.loc
```

> Attention!
> 
> Make sure your default gateway has subnet 192.168.2.1 (the gateway may be different). To successfully configure the DNS server, we need to install all 3 guest machines on the same subnet.

3. Installing a virtual machine with vagrant:

```
vagrant up
```

4. After successful installation, login via SSH protocol:

```
vagrant ssh
```

5. Switch to root user:

```
sudo -i
```

6. Update the repositories and install the necessary software:

```
apt update && apt -y upgrade && apt -y install nano mc curl wget sudo
```

7. Run the installer with your preferred [database engine](https://www.cloudpanel.io/docs/v2/getting-started/other/) (it's best to use MySQL by default):

```
curl -sSL https://installer.cloudpanel.io/ce/v2/install.sh | sudo bash
```

8. After successful installation, go to the control panel at https://192.168.2.10:8443 and create a username and password for a user with administrator rights. You can use simple ones like:

```
Login: admin
Password: password
```

9. Add a PHP site with template Laravel 10 and specify the domain *sequel.loc*

# Setup DNS Server with BIND

### Prerequisites

Before you begin with this guide, you should have the following requirements:

* Two Ubuntu 22.04 Servers.
* A non-root user with root/administrator privileges.

### Setting Up FQDN (Fully Qualified Domain Name)

Before start installing BIND packages, you must ensure the hostname and FQDN of your servers are correct. In this demonstration, we will two Ubuntu servers with the following details:

| Hostname    |IP Address     | FQDN             |  Used As  |
|:------------|:--------------|:-----------------|:----------|
| ns1         |192.168.2.21   |ns1.sequel.loc    |BIND Master|
| ns2         |192.168.2.22   |ns2.sequel.loc    |BIND Slave |

## Setup ns1.sequel.loc and ns2.sequel.loc

> Attention!
> 
> After installing 10 steps on the first server, go through all the steps again replacing ns1 with ns2 except for point 7

1. In Cmder go to `ns1.sequel.loc` folder:

```
cd C:\Vagrant-CloudPanel-DNS\ns1.sequel.loc
```

2. Installing a virtual machine with vagrant:

```
vagrant up
```

3. After successful installation, login via SSH protocol:

```
vagrant ssh
```

4. Switch to root user:

```
sudo -i
```

5. Update the repositories and install the necessary software:

```
apt update && apt -y upgrade && apt -y install nano mc curl wget sudo bind9 bind9utils bind9-doc dnsutils bind9-utils
```

6. Setup FQDN on the `ns1.sequel.loc` server:

```
hostnamectl set-hostname ns1.sequel.loc
```

7. Next, edit the file `/etc/hosts` using the following command:

```
nano /etc/hosts
```

Add the following configuration to `ns1.sequel.loc` server:

```
192.168.2.21 ns1.sequel.loc ns1
192.168.2.22 ns2.sequel.loc ns2
```

8. Lastly, check and verify the FQDN on each server using the following command. On the `ns1` server you will get the FQDN as `ns1.sequel.loc`:

```
hostname -f
```

9. Edit the configuration `/etc/default/named` using the following command:

```
nano /etc/default/named
```

The line `OPTIONS=` allows you to set up specific options when the BIND service is running. In this demo, you will be running Bind only with IPv4, so you will need to make the `OPTIONS` line as below.

```
OPTIONS="-u bind -4"
```

Save and close the file when you are done.

10. Now run the below command to restart the Bind service `named`. Then, check and verify the status of the BIND service. You should see the Bind service `named` is running on both servers.

```
systemctl restart named
systemctl status named
```

### Setting Up BIND Master `ns1.sequel.loc`

Run the command below to edit the configuration file `/etc/bind/named.conf.options`.

```
nano /etc/bind/named.conf.options
```

Add the following configuration to the file at the top of the line, before the `options {....};` line.

With this configuration, you will be creating an ACL (Access Control List) with the name `trusted`, which includes all trusted IP addresses and networks in your environment. Also, be sure to add the local server IP address `ns1` and the IP address of the `ns2` secondary DNS server.

```
	acl "trusted" {
        192.168.2.21;    # ns1 - or you can use localhost for ns1
        192.168.2.22;    # ns2
        192.168.2.0/24;  # trusted networks
	};
```

Now make changes to the "options {..};" section as below.

In the following example, we're disabling the support for IPv6 by commenting the option `listen-on-v6`, enabling and allowing recursion from the `trusted` ACL, and running the BIND service on specific `ns1` IP address `192.168.2.21`. Also, we are disabling the default zone transfer and defining the specific forwarders for the BIND DNS server to Google Public DNS `8.8.8.8` and `8.8.4.4`.

```
	options {

        directory "/var/cache/bind";

        //listen-on-v6 { any; };        # disable bind on IPv6

        recursion yes;                 # enables resursive queries
        allow-recursion { trusted; };  # allows recursive queries from "trusted" - referred to ACL
        listen-on { 192.168.5.21; };   # ns1 IP address
        allow-transfer { none; };      # disable zone transfers by default

        forwarders {
            // Router DNS
            192.168.2.1

            // Google Public DNS
            8.8.8.8;
            8.8.4.4;
 
            // OpenDNS
            208.67.222.222;
            208.67.220.220;
        };
	};
```

Save and close the file when you are done.

Lastly, run the following command to check and verify the config file `/etc/bind/named.conf.options`. If there is no output message, then your configuration is correct.

```
named-checkconf /etc/bind/named.conf.options
```

## Setting Up Zones (ns1.sequel.loc)

After setting up the basic configuration of BIND master, now you will be setting up zones for your domain name. In the following example, we will use the domain name `sequel.loc` with the name server `ns1.sequel.loc` and `ns2.sequel.loc`.

Edit the configuration file `/etc/bind/named.local` using the following command.

```
nano /etc/bind/named.conf.local
```

In this configuration, you will be defining two zone files, forward and reverse zone for your domain name. The Forward zone will contain the configuration of where your domain names will be resolved to the IP address, while the reverse zone will translate the IP address to which domain name.

In the following example, we will define the forward zone `/etc/bind/zones/sequel.loc` for domain `sequel.loc` and the reverse zone `/etc/bind/zones/192.168.2`.

```
zone "sequel.loc" {
    type master;
    file "/etc/bind/zones/sequel.loc"; # zone file path
    allow-transfer { 192.168.2.22; };           # ns2 IP address - secondary DNS
};


zone "2.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/192.168.2";  # subnet 192.168.2.0/24
    allow-transfer { 192.168.2.22; };  # ns2 private IP address - secondary DNS
};
```

Save and close the file when you are done.

Next, run the following command to create a new directory `/etc/bind/zones` that will be used to store zone config files.

```
mkdir -p /etc/bind/zones/
```

After that, copy the default forward zone configuration `/etc/bind/zones/sequel.loc` and edit the file using the following command.

```
cp /etc/bind/db.local /etc/bind/zones/sequel.loc
nano /etc/bind/zones/sequel.loc
```

Change the default SOA record with your domain name. Also, you will need to change the `Serial` number inside the SOA records every time you make changes to the file, and this must be the same `Serial` number with the secondary/slave DNS server.

Then, you can define NS records and A records for your DNS server. In this example, the name server will be `ns1.sequel.loc` with the A record IP address `192.168.2.21` and `ns2.sequel.loc` with the A record of the secondary DNS server IP address `192.168.2.22`.

Lastly, you can define other domain names. In this example, we will define an MX record (mail handler) for the domain `sequel.loc` which will be handled by the mail server `mail.sequel.loc`. Also, we will define the domain name `sequel.loc` that will be resolved to the server with IP address `192.168.2.10` and the sub-domain for the mail server `mail.sequel.loc` to the server IP address `192.168.2.11`.

```
;
; BIND data file for the local loopback interface
;
$TTL    604800
@       IN      SOA     ns1.sequel.loc. admin.sequel.loc. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;

; NS records for name servers
    IN      NS      ns1.sequel.loc.
    IN      NS      ns2.sequel.loc.

; A records for name servers
ns1.sequel.loc.          IN      A       192.168.2.21
ns2.sequel.loc.          IN      A       192.168.2.22

; Mail handler or MX record for the domain sequel.loc
sequel.loc.    IN     MX   10   mail.sequel.loc.

; A records for domain names
sequel.loc.            IN      A      192.168.2.10
mail.sequel.loc.       IN      A      192.168.2.11
```

Save and close the file when you are done.

Next, copy the default reverse zone config file to `/etc/bind/zones/192.168.2` and edit the new file using the following command.

```
cp /etc/bind/db.127 /etc/bind/zones/192.168.2
nano /etc/bind/zones/db.192.168.2
```

Change the default SOA record using your domain name. Also, do not forget to change the `Serial` number inside the SOA record.

Define NS records for your DNS servers. These are the same name servers that you used in the forward zone.

Lastly, define the PTR records for your domain names. The number on the PTR records is the last number of the IP address. In this example, the name server `ns1.sequel.loc` is resolved to the IP address `192.168.2.21`, so now the PTR record will be `21` and so on for other domain names.

```
;
; BIND reverse data file for the local loopback interface
;
$TTL    604800
@       IN      SOA     ns1.sequel.loc. admin.sequel.loc. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;

; name servers - NS records
      IN      NS      ns1.sequel.loc.
      IN      NS      ns2.sequel.loc.

; PTR Records
21   IN      PTR     ns1.sequel.loc.    ; 192.168.2.21
22   IN      PTR     ns2.sequel.loc.    ; 192.168.2.22
10  IN      PTR     sequel.loc.        ; 192.168.2.10
11  IN      PTR     mail.sequel.loc.   ; 192.168.2.11
```

Save and close the file when you are done.

Now run the following command to check BIND configurations and be sure don't get any error message.

```
named-checkconf
```

Then, run the following command to check and verify each zone files that you just created, the forward zone and reverse zone configuration file. If your zone files have no error, you should see the output message such as *OK*. If there is no error, the command will show you which line of the file caused an error.

```
named-checkzone sequel.loc /etc/bind/zones/sequel.loc
named-checkzone 2.168.192.in-addr.arpa /etc/bind/zones/192.168.2
```

To finish up the BIND Master configuration, run the below command to restart the BIND service and apply new changes to the configurations that you have made.

```
systemctl restart named
```

## Setting Up BIND Slave (ns2.sequel.loc)

Now you have finished the configuration of the Master BIND DNS Server. It's time to set up the `ns2` server as the secondary or salve of the BIND DNS server.

The Master server stores zone files that contain the DNS configuration of your domain and handle recursive or iterative queries. The secondary/slave DNS server stores DNS records for a period of time temporarily, and these DNS records are automatically transferred from the Master BIND server.

Now move to the `ns2` terminal session and start configuring the `ns2` server as a Secondary/Slave of the BIND DNS server.

Run the following command to edit the configuration file `/etc/bind/named.conf.options`

```
nano /etc/bind/named.conf.options
```

On top of the line, add the following configuration. This will create the same *ACL (Access Control List)* as on the Master server.

```
acl "trusted" {
        192.168.5.21;    # ns1
        192.168.5.22;    # ns2 - or you can use localhost for ns2
        192.168.5.0/24;  # trusted networks
};
```

Inside the `options {...};` line, you can change the configuration as below. This configuration is still the same as on the Master BIND DNS server, and the only difference here is the `listen-on` option which is specified to the `ns2` server IP address.

```
	options {

        directory "/var/cache/bind";

        //listen-on-v6 { any; };        # disable bind on IPv6

        recursion yes;                 # enables resursive queries
        allow-recursion { trusted; };  # allows recursive queries from "trusted" - referred to ACL
        listen-on { 192.168.5.22; };   # ns2 IP address
        allow-transfer { none; };      # disable zone transfers by default

        forwarders {
            // Router DNS
            192.168.2.1

            // Google Public DNS
            8.8.8.8;
            8.8.4.4;
 
            // OpenDNS
            208.67.222.222;
            208.67.220.220;
        };
	};
```

Save and close the file when you are done.

Next, edit the config file `/etc/bind/named.conf.local` using the following command to set up the `ns2` server as the secondary/slave DNS Server.

```
nano /etc/bind/named.conf.local
```

Add the following configuration to the file. As you can see we're defining the forward and reverse zones, but with the `type slave` and define the DNS Master server `192.168.2.21`. You do not need to create the zone file because DNS records and data will be automatically transferred from the DNS Master server and will be stored temporarily for a period of time on the secondary/slave DNS server.

```
zone "sequel.loc" {
    type slave;
    file "/etc/bind/zones/sequel.loc";
    masters { 192.168.2.21; };           # ns1 IP address - master DNS
};


zone "2.168.192.in-addr.arpa" {
    type slave;
    file "/etc/bind/zones/192.168.2";
    masters { 192.168.2.21; };  # ns1 IP address - master DNS
};
```

Save and close the file when you are done.

Now run the following command to check and verify the BIND configuration and be sure all configurations are correct. Then, you can restart the BIND service `named` on the `ns2` server to apply new changes. And you have now finished the configuration on the `ns2` server as a secondary/slave of the BIND DNS Server.

```
named-checkconf
systemctl restart named
```

Lastly, run the following command to check and verify the BIND service `named` on the `ns2` server. And be sure the `named` service is running.

```
systemctl status named
```

## Verifying DNS Server from Client Machine (sequel.loc)

On the client machine, there are multiple ways to set up the DNS resolver. You can set up the DNS resolver from the NetworkManager or from the netplan configuration. But, the easiest way is to set up the DNS resolver manually through the file `/etc/resolv.conf`. This allows you to set up a static DNS resolver for client machines.

Run the following command to remove the default link file `/etc/resolv.conf` and create a new file using nano editor.

```
unlink /etc/resolv.conf
nano /etc/resolv.conf
```

Add the following configuration to the file. In the following configuration we are defining three different resolvers, the BIND DNS Master, the Secondary BIND DNS server, and the public Google DNS resolver. When the client machine requests information about the domain name, the information will be taken from the DNS resolver, from the top to the bottom.

```
nameserver 192.168.2.21
nameserver 192.168.2.22
nameserver 8.8.8.8
search sequel.loc
```

Save and close the file when you are done.

Next, run the command below to install some DNS utility to your client machine. In this example, the client machine is an Ubuntu system, so we are installing DNS utility using the apt command as below.

After you have installed the DNS utility on your system, you can start checking all DNS records from the client machine.

Run the dig command below to check the domain name `sequel.loc` and `mail.sequel.loc`. And you should see the `sequel.loc` is resolved to the server IP address `192.168.2.10`, while the sub-domain `mail.sequel.loc` is handled by the server IP address `192.168.2.11`.

```
dig sequel.loc +short
dig sequel.loc

dig mail.sequel.loc +short
dig mail.sequel.loc
```

Next, run the dig command as below to check the mail handler for the domain name `sequel.loc`. And you should get the output that the `mail.sequel.loc` is handled mail for the main domain `sequel.loc`.

```
dig hwdomain.io MX +short
dig hwdomain.io MX
```

Now you can also verify the reverse zone configuration for your domain name using the nslookup command.

Run the nslookup command below to check and verify the reverse DNS for some IP addresses.

Now you should see that the IP address "192.168.5.21" is reversed to the name server `ns1.sequel.loc`, the IP address `192.168.2.22` is reversed to the name server `ns2.sequel.loc`, and the IP address `192.168.2.10` is reversed to the main domain name `sequel.loc`, and lastly the IP address `192.168.2.11` is reversed to the sub-domain `mail.sequel.loc`.

```
nslookup 192.168.2.21
nslookup 192.168.2.22
nslookup 192.168.2.10
nslookup 192.168.2.11
```

Open and enter the domain https://sequel.loc

## OpenSSL

SSL Certificate Generator:
```
openssl\crt\make-cert.bat
```

Before generating, you need to change the `cert.conf` config for the domain. After creating and adding the certificate to CloudPanel, click a couple of times on the certificate and install it with the parameters Local computer in the Trusted Root Certification Authorities folder.

The generated SSL certificates need to be added to the CloudPanel of the created site `sequel.loc`.
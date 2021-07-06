# PolarProxy-x-INetSim

## Configuration Steps

Just follow the steps and everything will work as shown

### Install INetSim

Enter following commands to install INetSim:

```console
sudo -s
echo "deb http://www.inetsim.org/debian/ binary/" > /etc/apt/sources.list.d/inetsim.list
curl https://www.inetsim.org/inetsim-archive-signing-key.asc | apt-key add -
apt update
apt install inetsim
```

#### Configure INetSim

1. Uncomment _service_bind_address_ and key in interface address (host-only):

```console
vi /etc/inetsim/inetsim.conf
service_bind_address    <Remnux IP>
```

2. Configure fake DNS server:

```console
vi /etc/inetsim/inetsim.conf
dns_default_ip  <Remnux IP>
```

3. Disable _https_ and _smtps_ service start as it will be superseded by PolarProxy:

```console
vi /etc/inetsim/inetsim.conf
#start_service https
#start_service smtps
```

4. Restart INetSim service:

```console
systemctl restart inetsim.service
```

5. Test INetSim:

```console
curl http://<Remnux IP>

<html>
  <head>
    <title>INetSim default HTML page</title>
  </head>
  <body>
    <p></p>
    <p align="center">This is the default HTML page for INetSim HTTP server fake mode.</p>
    <p align="center">This file is an HTML document.</p>
  </body>
</html> 
```

### Install PolarProxy

Enter the following commands to install PolarProxy as systemd service:

```console
sudo adduser --system --shell /bin/bash proxyuser
sudo mkdir /var/log/PolarProxy
sudo chown proxyuser:root /var/log/PolarProxy/
sudo chmod 0775 /var/log/PolarProxy/
sudo su - proxyuser
mkdir ~/PolarProxy
cd ~/PolarProxy/
curl https://www.netresec.com/?download=PolarProxy | tar -xzvf -
exit
sudo cp /home/proxyuser/PolarProxy/PolarProxy.service /etc/systemd/system/PolarProxy.service
```

#### Configure PolarProxy

1. Modify service config to configure PolarProxy to terminate TLS encryption for HTTPS and SMTPS as it should redirected to INetSim's server on tcp/80 and tcp/25 respectively:

```console
sudo vi /etc/systemd/system/PolarProxy.service

ExecStart=/home/proxyuser/PolarProxy -v -p 10443,80,80 -p 10465,25,25 -x /var/log/PolarProxy/polarproxy.cer -f /var/log/PolarProxy/proxyflows.log -o /var/log/PolarProxy/ --certhttp 10080 --terminate --connect <Remnux IP> --nosni nosni.inetsim.org
```

Arguments break-down list:<br>
__-v__ : verbose output in syslog (not required)<br>
__-p__ 10443,80,80 : listen for TLS connections on tcp/10443, save decrypted traffic in PCAP as tcp/80, forward traffic to tcp/80<br>
__-p__ 10465,25,25 : listen for TLS connections on tcp/10465, save decrypted traffic in PCAP as tcp/25, forward traffic to tcp/25<br>
__-x__ /var/log/PolarProxy/polarproxy.cer : Save certificate to be imported to clients in /var/log/PolarProxy/polarproxy.cer (not required)<br>
__-f__ /var/log/PolarProxy/proxyflows.log : Log flow meta data in /var/log/PolarProxy/proxyflows.log (not required)<br>
__-o__ /var/log/PolarProxy/ : Save PCAP files with decrypted traffic in /var/log/PolarProxy/<br>
__--certhttp 10080__ : Make the X.509 certificate available to clients over http on tcp/10080<br>
__--terminate__ : Run PolarProxy as a TLS termination proxy, i.e. data forwarded from the proxy is decrypted<br>
__--connect 192.168.53.19__ : forward all connections to the IP of INetSim<br>
__--nosni nosni.inetsim.org__ : Accept incoming TLS connections without SNI, behave as if server name was "nosni.inetsim.org".

2. Restart PolarProxy service:

```console
sudo systemctl enable PolarProxy.service
sudo systemctl start PolarProxy.service
```

3. Test PolarProxy:

```console
curl --insecure --connect-to example.com:443:<Remnux IP>:10443 https://example.com

<html>
  <head>
    <title>INetSim default HTML page</title>
  </head>
  <body>
    <p></p>
    <p align="center">This is the default HTML page for INetSim HTTP server fake mode.</p>
    <p align="center">This file is an HTML document.</p>
  </body>
</html>
```

#### Verify certificate against PolarProxy root CA

Download root certificate via HTTP service on tcp/10080 and convert from DER to PEM format through _openssl_ for use with _--cacert_ switch:

```console
curl http://<Remnux IP>:10080/polarproxy.cer > polarproxy.cer
openssl x509 -inform DER -in polarproxy.cer -out polarproxy-pem.crt
curl --cacert polarproxy-pem.crt --connect-to example.com:443:<Remnux IP>:10443 https://example.com

<html>
  <head>
    <title>INetSim default HTML page</title>
  </head>
  <body>
    <p></p>
    <p align="center">This is the default HTML page for INetSim HTTP server fake mode.</p>
    <p align="center">This file is an HTML document.</p>
  </body>
</html>
```

#### Set up routing

1. Configure firewall:

```console
sudo iptables -t nat -A PREROUTING -i <host-only interface> -p tcp --dport 443 -j REDIRECT --to 10443
sudo iptables -t nat -A PREROUTING -i <host-only interface> -p tcp --dport 465 -j REDIRECT --to 10465
sudo iptables -t nat -A PREROUTING -i <host-only interface> -j REDIRECT
```

2. Test firewall rule - HTTPS:

```console
curl --insecure --resolve example.com:443:<Remnux IP> https://example.com

<html>
  <head>
    <title>INetSim default HTML page</title>
  </head>
  <body>
    <p></p>
    <p align="center">This is the default HTML page for INetSim HTTP server fake mode.</p>
    <p align="center">This file is an HTML document.</p>
  </body>
</html>
```

3. Test firewall rule - SMTPS:

```console
curl --insecure --resolve example.com:465:<Remnux IP> smtps://example.com

214-Commands supported:
214- HELO MAIL RCPT DATA
214- RSET NOOP QUIT EXPN
214- HELP VRFY EHLO AUTH
214- ETRN STARTTLS
214 For more info use "HELP <topic>".
```

#### Persistent firewall rules

1. Install _iptables-persistent_:

```console
sudo apt install iptables-persistent
```

2. Save iptables rules:

```console
sudo iptables-save > iptables.bak
```

3. Create startup file for restoring iptables rules [optional]:

```console
vi /home/remnux/Documents/startup.sh

#!/bin/sh

iptables-restore < /home/remnux/Documents/iptables.bak
inetsim
```

4. Create systemd for restoring iptables rules [optional]:

```console
sudo vi /etc/systemd/system/malwareanalysis.service

[Unit]
Description="This service will restore iptables and run INetSim properly"

[Service]
User=root
WorkingDirectory=/home/remnux/Documents/
ExecStart=/home/remnux/Documents/startup.sh

[Install]
WantedBy=multi-user.target
```

### Install Certificate on Windows Machine

Certificate must be installed on Windows machine

#### Transfer certificate

1. Start python HTTP server in directory where _polarproxy.cer_ is located:

```console
python3 -m http.server 8080
```

2. From Windows machine, download the certificate:

```console
Invoke-WebRequest -Uri "http://<Remnux IP>:8080/polarproxy.cer" -OutFile "<C:\directory\to\store\file>"
```

3. Install certificate:

```console
1. Double-click on "polarproxy.cer"
2. Click [Install Certificate...]
3. Select Local Machine and press [Next]
4. Select Place all certificates in the following store and press [Browse...]
5. Choose "Trusted Root Certification Authorities" and press [OK], then [Next]
6. Press [Finish]
```

4. Test secure connection:

```console
1. Open browser and visit random sites over HTTPS
```

### Access Decrypted Traffic

Not recommeded to follow the command. But it will show you everything you want to see:

```console
tail -f /var/log/PolarProxy/*.pcap
```

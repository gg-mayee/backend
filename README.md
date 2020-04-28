This tutorial aims to Setup your own SoftEther Server via Local Bridging with DHCP Server(BIND) from scratch.
With added OpenVPN, L2TP and SSTP plus Squid Proxy installation

Requirements:
Own VPS (1GB RAM and 1vCPU for Minimal)(No any SoftEther related softwares Installed)
Common sense, if you're new for handling a VPS or a linux, research first(basic controls for linux instance,key shortcuts, basic commands)
SSH Client [Windows: PuTTy,Bitvise][Android: JuiceSSH,Termius,ConnectBot] (In this tutorial, im using Termius app)
This is my Termius App im using for everyday linux usage, Link provided by Barts: https://www.dropbox.com/s/dl/xq17xihvg64kmo0/Termius.apk
Must have root access or superuser privileges (run sudo su - sa mga sudoer na hindi pa naka root user)
Patience and calm-minded, medyo mahaba-haba po ito.
Notepad for scratching commands
Stable Internet para maiwasan po magdisconnect sa ssh client
Im trying to focus on one-liner commands as possible(its more on copy&paste or copy,paste&edit commands) to shortcut your time configuring your server, so medyo mapapadali po ng konti ung tutorial :)

First Step:
+ Prepare anything.
Login to your VPS via ssh client, run sudo su - for non-root user.
# Lets update and upgrade our VPS
For Debian/Ubuntu, run: apt update && apt upgrade -y
For CentOS, run: yum update -y
For Fedora, run: dnf update -y
# If may lumalabas po na errors after ng update & upgrade, and then followed the instructions from package manager, and still error, suggest kopo rebuild nyo nalang po yung VPS and balik po kayo sa tutorial nato.


Second Step:
+ Installing all our needed and wanted packages(editing,download/upload files,archiving/extracting files and encoding)
For Debian/Ubuntu, run: apt install nano wget curl zip unzip tar gzip bc rc openssl cron net-tools dnsutils dos2unix screen bzip2 -y
For CentOS, we need to install epel-release first: yum install epel-release -y
Then run: yum install nano wget curl zip unzip tar gzip bc rc openssl cronie net-tools bind-utils dos2unix screen bzip2 initscripts chkconfig -y
For Fedora, run: dnf install nano wget curl zip unzip tar gzip bc rc openssl cronie net-tools bind-utils dos2unix screen bzip2 -y
+ Then installing all building and development tools needed for SofEther Server and DHCP Server.
For Debian/Ubuntu, run: apt install build-essential libreadline-dev libssl-dev libncurses-dev zlib1g-dev isc-dhcp-server -y
For CentOS, run: yum groupinstall 'Development Tools' -y && yum install glibc-devel zlib-devel openssl-devel readline-devel ncurses-devel libstdc++-devel dhcp -y
For Fedora, run: dnf groupinstall 'Development Tools' -y && dnf install glibc-devel zlib-devel openssl-devel readline-devel ncurses-devel libstdc++-devel dhcp -y
# As you can see, Fedora and CentOS are likely the same on commands, they're same RHEL based OS, dnf is newer version of yum. If Fedora uses yum(older than dnf) as package manager for installation, some errors can occured like improper package boostraping and so on..
Tips: To preserve our installation and kung sakaling magkaroon po ng connection lost at masave parin po ung ginagawa natin, run: screen -S phc . Then for example po nag disconnect po talaga kayo sa ssh session, run screen -r phc to continue your session. vice-versa if nag dc po kayo ulit, run the second command.


Third Step:
+ Downloading and Compiling SoftEtherVPN source
# Our source came from SoftEtherVPN stable repository from github.
run this to download source and extract: wget -qO softether.tar.gz "https://github.com/SoftEtherVPN/SoftEtherVPN_Stable/archive/v4.34-9745-beta.tar.gz" && tar xzf softether.tar.gz && rm -f softether.tar.gz && mv SoftEtherVPN_Stable* SE
# We've moved the softether source folder/directory to a easier name, now papasok po tayo sa SE Folder/directory para magcompile at maginstall po ng source(this may take a minutes depending on your VPS processing speed)(Please be patient)
run: cd SE && ./configure && make && make install
# After compiling and installing are finished, copy the vpnserver script and transfer it to /etc/init.d folder/directory
For Debian/Ubuntu, run: cp debian/softether-vpnserver.init /etc/init.d/vpnserver && chmod +x /etc/init.d/vpnserver
For CentOS/Fedora, run: cp centos/SOURCES/init.d/vpnserver /etc/init.d/vpnserver && chmod +x /etc/init.d/vpnserver
Then exit to our SoftEther source directory and delete the entire folder/directory: cd .. && rm -rf SE


Fourth Step:
+ Configuring SoftEther Server
# We're using "vpncmd" command for controlling and managing SoftEther.
# Lets start SoftEther first
run: vpnserver start &> /dev/null
# Then enable our SoftEther server to start at boot
run: chkconfig vpnserver on &> /dev/null || systemctl enable vpnserver &> /dev/null
# Next, Set our virtual hub name and password using a variable, copy nyo po ito muna, then edit nyo po and then run after changing values
Copy first:
VirtualHubName='PHCorner' && VirtualHubPass='www.phcorner.net'
# then edit, and run like this for example:
run: VirtualHubName='Bon-chan' && VirtualHubPass='bonv'
# Creating our Virutal Hub using variables
vpncmd localhost /SERVER /CMD HubCreate "$VirtualHubName" /PASSWORD:"$VirtualHubPass"
# Then run all of these to Disable NAT function on our Virtual Hub
vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD DhcpDisable
vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD NatDisable
vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD SecureNatDisable
# And setting our local bridge
# But first, identify po muna natin yung gagamitin nating interface/device name (Note: 3 to 4 characters only)
DeviceName='phc'
Then run: DeviceName='bon'
# Then create a local bridge
run: vpncmd localhost /SERVER /CMD BridgeCreate "$VirtualHubName" /DEVICE:"$DeviceName" /TAP:yes
# And configuring our Bridge via ifconfig
run: ifconfig tap_$DeviceName 172.30.0.1 netmask 255.255.0.0 broadcast 172.30.255.255 promisc mtu 1500 up
# Then i-aadd po natin ung ifconfig command natin inside vpnserver script para kapag nagreboot po ung machine natin, nakasave parin po ung config ng local bridge
For Debian/Ubuntu, run: sed -i '/$DAEMON start/a ifconfig BridgeDeviceName 172.30.0.1 netmask 255.255.0.0 broadcast 172.30.255.255 promisc mtu 1500 up && bash /etc/softether.iptables' /etc/init.d/vpnserver && sed -i "s|BridgeDeviceName|tap_$DeviceName|g" /etc/init.d/vpnserver
For CentOS/Fedora, run: sed -i '/$exec start/a ifconfig BridgeDeviceName 172.30.0.1 netmask 255.255.0.0 broadcast 172.30.255.255 promisc mtu 1500 up && bash /etc/softether.iptables' /etc/init.d/vpnserver && sed -i "s|BridgeDeviceName|tap_$DeviceName|g" /etc/init.d/vpnserver
# Next, gagawa po tayo ng simple iptables script para sa ip forwading softether natin for local bridging.
run: echo -e "PubNet=\"$(ip -4 route ls | grep default | grep -Po '(?<=dev )(\S+)' | head -1)\"\nPubIP=\"$(curl -4s http://ipinfo.io/ip)\"\niptables -I FORWARD -s 172.30.0.0/16 -j ACCEPT\niptables -t nat -A POSTROUTING -o \$PubNet -j MASQUERADE\niptables -t nat -A POSTROUTING -s 172.30.0.0/16 -o \$PubNet -j MASQUERADE\niptables -t nat -A POSTROUTING -s 172.30.0.0/16 -o \$PubNet -j SNAT --to-source \$PubIP" > /etc/softether.iptables
Then run: sed -i '/net.ipv4.ip_forward.*/d' /etc/sysctl.conf && sed -i '/net.ipv4.ip_forward.*/d' /etc/sysctl.d/*.conf && echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/20-softether.conf && sysctl --system &> /dev/null && bash /etc/softether.iptables
# And set our Server Encryption Algorithm, set lang po muna naten sa simple cipher like 128-bit AES
run: vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD ServerCipherSet AES128-SHA


Fifth Step:
+ Configuring SoftEther Services
# Kung gusto nyo po ng SoftEtherVPN-Only lang, ignore nyo na po ito, proceed na po kayo sa next step, pero kung want nyo po i-enable din si L2TP, SSTP and OpenVPN, follow nyo lang po ito
# Ang gagamitin po natin na VirtualHub dito is yung variable po natin kanina

# Configure IPSec and L2TP VPN Function
# Need po natin dito magset ng Pre-Shared key, same lang po sa ginawa natin kanina, copy, paste and edit lang po ng value sa variable
PreSharedKey='phcornerkey'
# And run like this for example: PreSharedKey='phcornerkey'
# Then run vpncmd to enable IPSecVPN and L2TP
run: vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD IPSecEnable /L2TP:yes /L2TPRAW:yes /ETHERIP:yes /PSK:"$PreSharedKey" /DEFAULTHUB:"$VirtualHubName"

# Configure SSTP
# If you dont want to enable it, wag nyo na po irun
vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD SstpEnable yes &> /dev/null

# Configure OpenVPN
# my fave function of SoftEther, a built-in OpenVPN Server clone inside of a SoftEther Server
# Magseset po muna tayo ng variable for OpenVPN ports para magamit po natin mamaya sa paggawa ng client config.
Copy:
OpenVPN_Port='8888'
Edit for your desired port, then run it for example: OpenVPN_Port='8888'
# then enable na po natin si OpenVPN Service
run: vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD OpenVpnEnable yes /PORTS:"$OpenVPN_Port"
# Next is Making a Server certificate
# Magagamit po natin ito para sa openvpn server clone
# Makikita nyo po ung command parang may mga changable values like name ko , country and city, pwede nyo po palitan yan lahat, Ingat lang po sa pagpalit ng value ng C. C po is Country Name , dapat po 2 or 3 Characters lang po yan or else mag eerror ung certificate na gagawin po natin
# ung EXPIRES:9999 is expiry po ng certificate nyo, that's 9999 Days(Maximum value na po yan, wag nyo na po dagdagan)
# Like variables po: copy , edit and run
run: vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD MakeCert /CN:"Bon-chan SoftEther Service" /O:"BonvScripts SoftEther Tutorial" /OU:"github.com/Bonveio/BonvScripts" /C:PH /ST:NCR /L:"Caloocan" /SERIAL:none /EXPIRES:9999 /SAVECERT:"~/ca.crt" /SAVEKEY:"~/ca.key"
# Then i-iimport napo natin sya sa SoftEther as Server Certificate
run: vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD ServerCertSet /LOADCERT:"~/ca.crt" /LOADKEY:"~/ca.key"
Tip: Pwede nyo pong gamitin ulit ung generated nyong Certificate sa new or sa iba nyong VPS na iinstallan nyo din po ng SoftEther(1; Archive your ca.crt and ca.key in your root folder.2; Extract that file to your another vps na iinstallan ng SoftEtherVPN Server. 3; Run vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD ServerCertSet /LOADCERT:"~/ca.crt" /LOADKEY:"~/ca.key" But Set your $VirtualHubName variable first.)
# Generating Sample OpenVPN Client gamit po ung ginawa nating Certificate
# Note: SoftEther's OpenVPN are multi protocol.Ibig sabihin ung port po na gamit natin ay pwede pong gamitin as TCP or UDP port, were using UDP in our .ovpn config
run: echo -e "client\ndev tun\nproto udp\nremote $(curl -4s http://ipinfo.io/ip) $OpenVPN_Port\nremote-cert-tls server\ncipher none\nauth MD5\nconnect-retry infinite\nresolv-retry infinite\nfloat\npersist-remote-ip\npersist-tun\nkeysize 0\nnobind\nmute-replay-warnings\nauth-user-pass\nauth-nocache\nverb 1\nsetenv CLIENT_CERT 0\n<ca>\n$(cat ~/ca.crt)\n</ca>" > ~/client.ovpn
# Diskarte nyo na po kung paano kukuhain ung client config, You can use SFTP Client as well, my own way is using FFSend for file transfers, to Download and Install FFSend in your vps:
run: curl -4SL "https://github.com/timvisee/ffsend/releases/download/v0.2.59/ffsend-v0.2.59-linux-x64-static" -o /usr/local/bin/ffsend && chmod +x /usr/local/bin/ffsend
then run: ffsend upload ~/client.ovpn
# Copy link then paste nyo po sa browser for download .ovpn config


Sixth Step:
+ Configure our DHCP Server
# Creating dhcpd config, pwede nyo po palitan ung DNS, pero make sure nyo po na anycast-type ung DNS, like cloudflare,google,opendns,level3 and so on.
run: echo -e "ddns-update-style none;\n#authoritative;\n#option domain-name "localhost";\ndefault-lease-time 600;\nmax-lease-time 3600;\noption routers 172.30.0.1;\noption subnet-mask 255.255.0.0;\noption broadcast-address 172.30.255.255;\n\n subnet 172.30.0.0 netmask 255.255.0.0 {\n option domain-name-servers 1.1.1.1,1.0.0.1;\n pool {\n range 172.30.0.10 172.30.255.250;\n }\n }" > /etc/dhcp/dhcpd.conf
# Creating some workaround for systemd/init.d and starting our dhcp server
For Debian/Ubuntu, run: echo -e "INTERFACESv4=\"tap_$DeviceName\"\nINTERFACESv6=\"\"\n" > /etc/default/isc-dhcp-server && systemctl daemon-reload && systemctl enable isc-dhcp-server && systemctl start isc-dhcp-server
For CentOS 6, run: echo -e "DHCPARGS=\"tap_$DeviceName\"" > /etc/sysconfig/dhcpd && chkconfig dhcpd on && service dhcpd start
For CentOS 7,8/Fedora, run: sed -i "s|\(ExecStart=\).\+|\1/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid tap_$DeviceName|" /lib/systemd/system/dhcpd.service && systemctl daemon-reload && systemctl start dhcpd && systemctl restart dhcpd


Seventh Step:
+ Creating user for our SoftEtherVPN Server
# Note here we're using our created Virtual Hub to create a user account. Every Hub, different user databases.
# Set nyo po muna ung values sa variables, copy this then edit:
Edit this and run for example: VPNUsername='Bon-chan' && VPNPassword='phcorner'
# Then create our user account
run: vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD UserCreate $VPNUsername /GROUP:none /REALNAME:none /NOTE:none &> /dev/null && vpncmd localhost /SERVER /ADMINHUB:"$VirtualHubName" /CMD UserPasswordSet $VPNUsername /PASSWORD:"$VPNPassword" &> /dev/null


Eighth Step:
+ Configuring Proxy for TCP Connections, we will using Squid as a proxy
# Ang setup po natin is normal lang na squid proxy, non-caching proxy-only setup.
# We need to install it first.
For Debian/Ubuntu, run: apt install squid -y
For CentOS, run: yum install squid -y
For Fedora, run: dnf install squid -y
# Set our squid Port
run: Proxy_Port='81'
# And then create our squid config
run: echo -e "acl VPN dst $(wget -4qO- http://ipinfo.io/ip)/32\nhttp_access allow VPN\nhttp_access deny all\nhttp_port 0.0.0.0:$Proxy_Port\nacl all src 0.0.0.0/0.0.0.0\nno_cache deny all\ndns_nameservers 1.1.1.1 1.0.0.1\nvisible_hostname localhost" > /etc/squid/squid.conf
# Then start squid proxy service
run: service squid restart
# It may stock for a while, okey lang po yan, ganyan po talaga magrestart ng squid service, medyo matagal


Last Step:
+ Setting our SoftEther Admin Password, sinadya ko po talaga itong ipahuli, para di po masyadong mahaba ung i eexecute natin na commands sa taas.
# copy, edit, then run:
run: vpncmd localhost /SERVER /CMD ServerPasswordSet changePasswordHere


Optional:
+ Creating TCP OpenVPN client config to test our squid proxy
run: echo -e "client\ndev tun\nproto tcp\nremote $(curl -4s http://ipinfo.io/ip) 443\nremote-cert-tls server\ncipher none\nauth MD5\nconnect-retry infinite\nresolv-retry infinite\npersist-remote-ip\npersist-tun\nkeysize 0\nnobind\nmute-replay-warnings\nauth-user-pass\nauth-nocache\nverb 1\nsetenv CLIENT_CERT 0\nhttp-proxy $(curl -4s http://ipinfo.io/ip) $Proxy_Port\nhttp-proxy-option CUSTOM-HEADER Host www.googleapis.com\n<ca>\n$(cat ~/ca.crt)\n</ca>" > ~/client_tcp.ovpn
then run: ffsend upload ~/client_tcp.ovpn
# Copy link , paste to your browser and download .. Edit nyo nalang po yung config after download


Thats all, your SoftEther server is now ready to go.
• For some experienced softether users out there, pasensya na po kung hindi masyadong klaro ung terms na gamit ko, And please correct me if something is wrong in this thread.. Para mafix po agad, thanks.

If someone is having a hard time following the tutorial, please comment down below.

Im not open for silly questions, for Example: (what is vps, what is softether,how can i get vps,how to use this ssh client..) Dont troll this thread please.

And dont forget to leave a like in my thread, this thread is written in raw bbcode and takes me more than a day to complete(too busy, multi-tasking).

By the way, performance of a locally bridged SoftEther server is more faster than a VirtualNAT. So enjoy quality speed.

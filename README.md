# Configure-IPsec-L2TP-VPN-Clients
Install L2TP VPN

Configure Linux VPN clients using the command line
After setting up your own VPN server, follow these steps to configure Linux VPN clients using the command line. Alternatively, you may connect using IKEv2 mode (recommended), or configure using the GUI. Instructions below are based on the work of Peter Sanford. Commands must be run as root on your VPN client.

To set up the VPN client, first install the following packages:

# Ubuntu and Debian
apt-get update
apt-get install strongswan xl2tpd net-tools

# Fedora
yum install strongswan xl2tpd net-tools

# CentOS
yum install epel-release
yum --enablerepo=epel install strongswan xl2tpd net-tools
Create VPN variables (replace with actual values):

VPN_SERVER_IP='your_vpn_server_ip'
VPN_IPSEC_PSK='your_ipsec_pre_shared_key'
VPN_USER='your_vpn_username'
VPN_PASSWORD='your_vpn_password'
Configure strongSwan:

cat > /etc/ipsec.conf <<EOF
# ipsec.conf - strongSwan IPsec configuration file

conn myvpn
  auto=add
  keyexchange=ikev1
  authby=secret
  type=transport
  left=%defaultroute
  leftprotoport=17/1701
  rightprotoport=17/1701
  right=$VPN_SERVER_IP
  ike=aes128-sha1-modp2048
  esp=aes128-sha1
EOF

cat > /etc/ipsec.secrets <<EOF
: PSK "$VPN_IPSEC_PSK"
EOF

chmod 600 /etc/ipsec.secrets

# For CentOS and Fedora ONLY
mv /etc/strongswan/ipsec.conf /etc/strongswan/ipsec.conf.old 2>/dev/null
mv /etc/strongswan/ipsec.secrets /etc/strongswan/ipsec.secrets.old 2>/dev/null
ln -s /etc/ipsec.conf /etc/strongswan/ipsec.conf
ln -s /etc/ipsec.secrets /etc/strongswan/ipsec.secrets
Configure xl2tpd:

cat > /etc/xl2tpd/xl2tpd.conf <<EOF
[lac myvpn]
lns = $VPN_SERVER_IP
ppp debug = yes
pppoptfile = /etc/ppp/options.l2tpd.client
length bit = yes
EOF

cat > /etc/ppp/options.l2tpd.client <<EOF
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-chap
noccp
noauth
mtu 1280
mru 1280
noipdefault
defaultroute
usepeerdns
connect-delay 5000
name "$VPN_USER"
password "$VPN_PASSWORD"
EOF

chmod 600 /etc/ppp/options.l2tpd.client
The VPN client setup is now complete. Follow the steps below to connect.

Note: You must repeat all steps below every time you try to connect to the VPN.

Create xl2tpd control file:

mkdir -p /var/run/xl2tpd
touch /var/run/xl2tpd/l2tp-control
Restart services:

service strongswan restart
service xl2tpd restart
Start the IPsec connection:

# Ubuntu and Debian
ipsec up myvpn

# CentOS and Fedora
strongswan up myvpn
Start the L2TP connection:

echo "c myvpn" > /var/run/xl2tpd/l2tp-control
Run ifconfig and check the output. You should now see a new interface ppp0.

Check your existing default route:

ip route
Find this line in the output: default via X.X.X.X .... Write down this gateway IP for use in the two commands below.

Exclude your VPN server's IP from the new default route (replace with actual value):

route add YOUR_VPN_SERVER_IP gw X.X.X.X
If your VPN client is a remote server, you must also exclude your Local PC's public IP from the new default route, to prevent your SSH session from being disconnected (replace with actual value):

route add YOUR_LOCAL_PC_PUBLIC_IP gw X.X.X.X
Add a new default route to start routing traffic via the VPN server：

route add default dev ppp0
The VPN connection is now complete. Verify that your traffic is being routed properly:

wget -qO- http://ipv4.icanhazip.com; echo
The above command should return Your VPN Server IP.

To stop routing traffic via the VPN server:

route del default dev ppp0
To disconnect:

# Ubuntu and Debian
echo "d myvpn" > /var/run/xl2tpd/l2tp-control
ipsec down myvpn

# CentOS and Fedora
echo "d myvpn" > /var/run/xl2tpd/l2tp-control
strongswan down myvpn

# voip
Asterisk
initscripts
chkconfig

## Plan du réseau d'entreprise
Machines:
- ipa.credit-industriel.corp
  - IP: 192.168.246.3 / 24
  - Services: Red Hat Identity Management avec  
- asterisk.credit-industriel.corp
  - IP: dhcp
  - Services:
- client-01.credit-industriel.corp
  - IP: dhcp
q:
## Installation du serveur: *ipa.credit-industriel.corp*

## Installation du serveur: *asterisk.credit-industriel.corp*
````bash
sudo hosnamectl hostname asterisk.credit-industriel.corp
sudo dnf install cockpit
sudo dnf install ipa-client
sudo ipa-client-install --mkhomedir --ssh-trust-dns --enable-dns-updates
ipa-getcert request -f /etc/cockpit/ws-certs.d/$(hostname -f).cert -k /etc/cockpit/ws-certs.d/$(hostname -f).key \
-D $(hostname -f) -K host/$(hostname -f) -m 0640  -o root:cockpit-ws -O root:root -M 0644
sudo systemctl enable --now cockpit.socket

sudo subscription-manager repo --enable codeready-builer-for-rhel9-$(arch)-rpms &&
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf install asterisk asterisk-voicemail asterisk-ldap 
````

## Dockerfile
````dockerfile
# syntax=docker /dockerfile:1
FROM debian:latest
LABEL maintainer='Cyril GENISSON <cyril.genisson@laplateforme.io>'
ENV build_date 2024-04-18

RUN apt update \
    && apt dist-upgrade -y \
    && apt autoremove -y \
    && cd /usr/src \
    && apt install git \
    && git clone -b certified/asterisk.20.7 --depth 1 https://github.com/asterisk/asterisk.git \
    && cd /usr/src/asterisk \
    && ./contrib/scripts/install_prereq \
    && ./configure \
    && make menuselect \
    && make install \
    && make samples \
    && make config \
 
RUN <<EOF
apt-get update
apt-get install -y 
EOF

````
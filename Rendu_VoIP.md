# VoIP

## Introduction Contextualisation
VoIP **Voice over Internet Protocol** est une technologie informatique
permettant de transmettre la voix sur des réseaux compatibles IP, via Internet
ou des réseaux privés ou publics.

## Avantages et inconvénients
Nous pourrions commencer par les avantages:
- réductions
- faible empeinte numérique

## Solutions existantes sur le marché
- solutions open sources:
  - Asterisk
  - Kamailia
- solutions commerciales:
  - Des tas...

## Exemple d'implémentation
Nous nous proposons ici d'implémenter de bout en bout une solution IPBX complète
en utilisant le logiciel Asterisk

### Mise en place du laboratoire
**Choix de la distribution pour notre solution:** Red Hat Enterprise Linux.

**Solution de virtualisation:** VMWare WorkStation  Professionnal

- **VM_1**: Identity Management (Kerberos/DNS/LDAP)
  - ipa.credit-industriel.corp 
    - Processeur: 2 vCPU
    - Mémoire: 2048 Mo
    - Réseau: ensXXX 192.168.246.3/24 NAT (Gateway 192.168.246.2)
- **VM_2**: Asterisk
  - ast.credit-industriel.corp
    - Processeur: 2vCPU
    - Mémoire: 4096 Mo
    - Réseaux:
      - ensXXX (dhcp with dns update ) NAT
      - ensXXX Bridge with Wireless Card Network (10.10.0.0/16 dhcp)
- **VM_3**: Windows Server 2022 (Active Directory)
  - srv.ad.credit-industriel.corp
    - Processeur: 2vCPU
    - Mémoire: 4096 Mo
    - Réseau: ethernet card 192.168.246.4/24 (Gateway 192.168.246.2)
    - Client SIP
- **Téléphone portable**: Samsung S24 client
  - client SIP: MIZUDROID

### Script d'installation automatisé Asterisk
````shell
#!/bin/bash
USER=
IPAPASS=""
CSV_USER_FILE=
SERVER=
URL=$USER@$SERVER:/home/$USER/asterisk

# Configure IPA Client
ipa-client-install -U -N --mkhomedir --ssh-trust-dns --enable-dns-updates --password=$IPAPASS

# Add system user Asterisk
useradd -r -s /sbin/nologin asterisk

# Firewalld config
firewall-cmd --permanent --add-service={sip,sips}
firewall-cmd --reload

# Define socket directory for Asterisk in tmpfiles.d
cat > /etc/tmpfiles.d/asterisk.conf << EOF
d /var/run/asterisk 0775 asterisk asterisk
EOF

cd /usr/src
git clone -b certified/20.7 https://github.com/asterisk/asterisk
cd asterisk
./contrib/scripts/install_prereq install
dnf install ntsysv initscripts -y
./contrib/scripts/get_mp3_source.sh
./configure --with-jansson-bundled --with-pjproject-bundled
make menuselect.makeopts
scp $URL/menuselect.* ./
make
make install
make install-logrotate
make samples
make config

# Install config files and keys
cd /etc/asterisk
scp $URL/*.conf ./ 
ipa-getcert request -K SIP/ast.credit-industriel.corp -k /etc/pki/tls/private/asterisk.key \
-f /etc/pki/tls/certs/asterisk.pem -g 2028 -D ast.credit-industriel.corp

# Install custom sounds
mkdir -p /var/lib/asterisk/sounds/fr/costum
scp $URL/custom/* /var/lib/asterisk/sounds/fr/costum/.

# Create users phones
git clone https://github.com/cyril-genisson/voip
./voip/create_phone $CSV_USERS_FILE

# Start Asterisk
systemctl daemon-reload
systemctl enable --now asterisk.service
````

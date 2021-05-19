https://github.com/jindrichskupa/kiv-spos

Rsync
------------------------------------------
```
rsync -rav /source/path/ /dest/path/
rsync -rav --dry-run			- na zkoušku
```

Konfigurace SSH
---------------------------------------------------------------------------------------------------
```
apt install openssh-client openssh-server	(pokud není sshd)
```
```
~/.ssh/authorized_keys	< pubkey
```
```
ssh-keygen -b bits -t rsa  	výstup v ~/.ssh/id_rsa(.pub)
```

konfigurace v /etc/ssh/sshd_config
```

	PermitRootLogin	yes/no/prohibit-password
	PasswordAuthentication	no/yes
	AllowedUsers		root user1 user2 ...
```


Konfigurace fail2ban
---------------------------------------------------------------------------------------------------
apt install fail2ban

konfigurace v /etc/fail2ban/jail.conf

ignorování ip adres přes ignoreip = ip1 ip2 ...	

Vytvoření RAIDu
---------------------------------------------------------------------------------------------------
```
apt install mdadm
```

```
mdadm --create --verbose /dev/md0 --level=X --raid-devices=N /dev/sd[abc]
mdadm -Cv /dev/md0 -lX -nN /dev/sd[abc]
```

```
mdadm --create --verbose /dev/md0 --level=10 --raid-devices=2 /dev/sdb /dev/sdc
cat /proc/mdstat (kontrola)
```

LVM
---------------------------------------------------------------------------------------------------
```
apt install lvm2
```

```
pvcreate /dev/sda
wipefs -a /dev/... pokud filter
pvdisplay
```

```
vgcreate group /dev/sda
vgextend group /dev/sd[bc]
vgdisplay
```

```
lvcreate -n drive1 -L 1g group
lvdisplay
```


MKFS a Mount
---------------------------------------------------------------------------------------------------
```
mkfs.ext4 /dev/group/drive1
mount /dev/group/drive1 /mnt/drive1 (pro lvm)
```

```
nano /etc/fstab

/dev/group/drive1	/mnt/drive1	defaults	0	0

mount -a (kontrola)
lsblk
```


DNS
---------------------------------------------------------------------------------------------------
```
apt install bind9
```

zóny v 
/etc/bind/named.conf.local

konfigurace zón v
/etc/bind/zones/db.barticka. ...

```
$TTL    604800
@       IN      SOA     hostname. root.localhost. (
                              7         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      ns.hostname.	; Definovat nameserver
@       IN      A       77.93.216.97	
ns      IN      A       77.93.216.97	; Pro subdoménu ns použít tuto adresu
mail    IN      A       2.2.2.2	; Pro subdoménu mail použít tuto adresu
```

konfigurace zón v /etc/named.conf.local

```
zone "barticka.spos." {			// Takhle definujeme zónu kde jsme master
    type master;
    file  "/etc/bind/db.barticka.spos"
    allow-transfer { adresa; };
};

zone "nekdojinej.spos." {			// Takhle definujeme zónu kde jsme slave
    type slave;
    file  "/etc/bind/slaves/db.barticka.spos"
    masters { ip adresa mastera; };
};

zone "nekdojinej.spos." {				// Takhle definujeme zónu, pro kterou forwardujeme
    type forward;
    forwarders { ip adresa kam forwardujeme; };
}
```


Můžeme definovat ACL, které pak využijeme v match-clients
```
acl "trusted" {							// Definujeme ACL
        77.93.199.0/24;    # ns1 - can be set to localhost		// Povolené adresy
        77.93.216.0/24;    # ns2
        84.42.159.140;  # host1
        185.99.177.253;
        127.0.0.1;
};
```

Options definuje globální nastavení
```
options {
        directory "/var/cache/bind";

        recursion yes;              	// Povolíme rekurzi
        allow-recursion { trusted; };	// Ale jenom pro ACL trusted
        listen-on { localhost; };   	// Posloucháme na tomhle
        allow-transfer { none; };    	// Nepovolíme transfery zón

        forwarders {			// Pokud nevíme, ptáme se tady
                1.1.1.1;
                1.1.2.2;
        };
};
```

Můžeme použít i pohledy
```
view "private" {					// Definujeme pohled
  match-clients { trusted; 192.168.0.0/24; }; // Pro ktere klienty 
  recursion yes;				// Povolime rekurzi
  zone "barticka.spos." {			// Definujeme zony
    type master;
    file  "/etc/bind/private/db.barticka.spos"
    allow-transfer { adresa; };
  };

  zone "nekdojinej.spos." {			
    type slave;
    file  "/etc/bind/slaves/db.barticka.spos"
    masters { ip adresa mastera; };
  };
};

view "public" {				// Musime definovat pohled pro ostatni, jinak je budeme ignorovat
match-clients { "any"; }; 
  recursion no;
  zone "barticka.spos." {
    type master;
    file  "/etc/bind/public/db.barticka.spos"
  };
};
```


Pokud používáme pohledy, musíme je použít i v /etc/bind/named.conf.default-zones
Přidáme proto

```
view "default" {
	match-clients {any;};
	..
	...
	};
```
	
	
Apache2
---------------------------------------------------------------------------------------------------
```
apt install apache2
```


kontrola konfigurací přes
```
apachectl -S a -t	
```

webová stránka ve /var/www/html
konfigurace v /etc/apache2	
sites-available		- dostupné stránky		povolit stránku přes a2ensite/a2dissite
sites-enabled		- běžící stránky
	
změnit 000-default.conf

Pro HTTP

```
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html
        ServerName barticka.spos.n-io.cz
        # ServerAlias web.jindra.spos # možný alias
        # Redirect / https://www.jindra.spos/ # možný redirect


        Alias /.well-known/acme-challenge/ /var/www/dehydrated/.well-known/acme-challenge/	# potřeba pro SSL přes LE

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Pro HTTPS (nový .conf)

```
<VirtualHost _default_:443>
                ServerAdmin webmaster@localhost

                DocumentRoot /var/www/html
                ServerName barticka.spos.n-io.cz

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                SSLEngine on

                SSLCertificateFile      /etc/letsencrypt/live/barticka.spos.n-io.cz/cert.pem		
                SSLCertificateKeyFile   /etc/letsencrypt/live/barticka.spos.n-io.cz/privkey.pem

                #SSLCertificateChainFile /etc/apache2/ssl.crt/server-ca.crt

                #SSLCACertificatePath /etc/ssl/certs/
                #SSLCACertificateFile /etc/apache2/ssl.crt/ca-bundle.crt

                #SSLCARevocationPath /etc/apache2/ssl.crl/
                #SSLCARevocationFile /etc/apache2/ssl.crl/ca-bundle.crl

                #SSLVerifyClient require
                #SSLVerifyDepth  10

                <FilesMatch "\.(cgi|shtml|phtml|php)$">
                                SSLOptions +StdEnvVars
                </FilesMatch>
                <Directory /usr/lib/cgi-bin>
                                SSLOptions +StdEnvVars
                </Directory>
</VirtualHost>
```


povolení PHP
```
apt-get install php libapache2-mod-php php-mysql
a2enmod php7.3
```


LetsEncrypt
```
a2enmod ssl

cd /opt && git clone https://github.com/lukas2511/dehydrated && cd dehydrated
mkdir /etc/dehydrated
cp ./docs/examples/hook.sh /etc/dehydrated/hook.sh && chmod +x /etc/dehydrated/hook.sh
```

nano /etc/dehydrated/config

```
DOMAINS_TXT=/etc/dehydrated/domains
WELLKNOWN="/var/www/dehydrated/.well-known/acme-challenge/"
CERTDIR="/etc/letsencrypt/live/"
CONTACT_EMAIL=barticka@students.zcu.cz
HOOK=/etc/dehydrated/hook.sh
```
```
nano /etc/dehydrated/domains < hostname

mkdir -p /var/www/dehydrated/.well-known/acme-challenge/ /etc/letsencrypt/live/

dehydrated --register --accept-terms

dehydrated --cron
```


MySQL
---------------------------------------------------------------------------------------------------
```
apt-get install mariadb-server
apt install php-mysql
```

konfigurace v /etc/mysql/mariadb.conf.d	- 50-server a 50-client

vytvoření databáze
```
CREATE DATABASE db01 DEFAULT CHARACTER SET utf8  DEFAULT COLLATE utf8_general_ci;
```

přepnutí do databáze 
```
USE db01
```

vytvoření uživatele
```
CREATE USER 'db01'@'localhost' IDENTIFIED BY 'password';		// localhost možno nahradi třeba % pro všechny
```

přihlášení
```
mysql -u db01 -ppassword
```

konfigurace přihlášení v ~/.my.cnf

grant privilegií
```
GRANT ALL PRIVILEGES ON db01.* to 'db01'@'localhost' IDENTIFIED BY 'password';
```

Zálohování přes:

soubory databáze jsou v /var/lib/mysql/jmeno-databaze
pokud není engine InnoDB, jde to překopírováním té složky přes rsync

pokud je InnoDB, jde to přes mysqldump, jako

mysqldump > cílový soubor	- to vygereruje SQLko


Postgres
---------------------------------------------------------------------------------------------------
```
apt install postgresql-11
apt install php-pgsql
```


startování databáze přes pg_ctlcluster verze sekce start

konfigurace ve /etc/postgresql/verze
databázové soubory ve /var/lib/postgresql

přepnutí pod uživatele postgres
```
su - postgres	
```

místo mysql příkaz psql

\l			- vylistovat databáze
\c databaze		- 
\dt 
\d tabulka
\x on | \x off

příkazy viz. github



MAIL
----------------------------------------------------------------------------------------------------
```
apt install postfix
```

logy ve /var/log/mail.log
maily defaultně ve /var/spool/mail/user

```
echo "Test" | mail -s "Testovaci mail" jindra@jindra.spos
```

konfigurace v /etc/postfix/main.cf	
```
home_mailbox = Maildir/
mailbox_command =
mydestination = ...	možno přidat doménu, pak ji bude brát v potaz localhost
```

```
apt install mutt
mutt -f /var/spool/mail/user
mutt -d /home/user/Maildir
```

```
apt-get install dovecot-pop3d
apt-get install dovecot-imapd
```


konfigurace v /etc/dovecot/conf.d/10-mail.conf	- možno nastavit maildir

```

~/.muttrc	autologin

set folder    = imap://jindra:jindra123@localhost:143/
set spoolfile = imap://jindra:jindra123@localhost:143/
```
	
postmap <cesta> (vytvoří nový db file)
případně newalias (vytvoří db z aliasů)


NFS
---------------------------------------------------------------------------------------------------
```
apt install nfs-kernel-server
	
/etc/exports	- nastaveni sdileni
exportfs -a	- provedeni zmen
```
	
namountovani:	
```
mount -t nfs ipadresa:/srv/share /mnt/nfs
```

do fstab:

```	
ipadresa:/srv/share	/mnt/nfs	nfs	defaults	0	0
```

je potřeba změnit oprávnění u /srv/share - chmod 777


Samba
---------------------------------------------------------------------------------------------------

```	
apt-get install samba smbclient cifs-utils	-> NO

smbpasswd -a user		- přidání uživatele
smbclient -L			- vylistování dostupných samba disků
```

konfigurace v /etc/samba/smb.conf
```
interfaces	- kde jsme vidět
workgroup	- skupina
	
přidat share viz. github - ne mezery, ale \n
```
		
```	
systemctl restart smbd	
smbclient //localhost/share1 -U user		- připojení na sdílený disk
```

mountování přes
```
mount -t cifs //localhost/share1 /mnt/share -o username=jindra
```

do fstabu přes
```
//localhost/share1	/mnt/share	cifs	username=user,password=password	0	0
```


NginX
---------------------------------------------------------------------------------------------------
```
apt install nginx
```

NginX běží na stejných portech jako Apache, ten je potřeba přemigrovat na jiné /etc/apache2/ports.conf a stejně tak v sites-enabled

konfigurace v /etc/nginx/sites-enabled

pro SSL přidat 	

```
listen port ssl default_server;
ssl_certificate /fullchain;
ssl_certificate_key /privkey;
```
        			
jako load balancer změnit root aby nebyl stejný jako u apache
       
potom upravit location - tohle provede přesměrování, nic víc
```
location / {
	proxy_pass http://localhost:port
} 			
```
	
	
pro load balancing přidáme upstream
```	
upstream backend  {
        ip_hash;						# hash IP adres - stejné IP adresy posíláme na stejné servery
        server localhost:8080 max_fails=2 fail_timeout=5s;		# seznam serverů
        server localhost:8081 max_fails=2 fail_timeout=5s;
}
```	
a potom v proxy_pass použijeme upstream        
```	
location / {
       	proxy_pass http://upstream;
} 
```
       

Cron
---------------------------------------------------------------------------------------------------

konfigurace v /etc/crontab

obecně není problém, ale defaultně se bere úkol jako "udělat v * * * * * *", ale je možné to změnit na "udělat každých * */x * * *", resp. dát před hodnotu lomítko       

Add key SSH
---------------------------------------------------------------------------------------------------
```
ssh-keygen -b bits -t rsa    výstup v ~/.ssh/id_rsa(.pub)  
ssh-add private key
ssh user@ip
```

Putty login via ssh key:
https://support.hostway.com/hc/en-us/articles/115001509884-How-To-Use-SSH-Keys-on-Windows-Clients-with-PuTTY-


Firewall
---------------------------------------------------------------------------------------------------

https://sleeplessbeastie.eu/2018/09/10/how-to-make-iptables-configuration-persistent/
https://www.digitalocean.com/community/tutorials/iptables-essentials-common-firewall-rules-and-commands
https://upcloud.com/community/tutorials/configure-iptables-debian/

Allow loopback:
```
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT
```
allow established incomming:
```
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```
	
drop invalid:
```
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
```

allow SSH
```
sudo iptables -A INPUT -p tcp --dport ssh -j ACCEPT
```
	
allow http
```
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

allow MySQL from specified IP
```
sudo iptables -A INPUT -p tcp -s 15.15.15.0/24 --dport 3306 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 3306 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```
	
allow postgres from specific IP
```
sudo iptables -A INPUT -p tcp -s 15.15.15.0/24 --dport 5432 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 5432 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```
	
allow postgres to specific network interface:
```
sudo iptables -A INPUT -i eth1 -p tcp --dport 5432 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -o eth1 -p tcp --sport 5432 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

Drop all other:
```
iptables -P INPUT DROP
(changing default chain policy)
```

list policies:
```
iptables -L 
iptables -S
```
	
delete rule:
https://www.digitalocean.com/community/tutorials/how-to-list-and-delete-iptables-firewall-rules
```
sudo iptables -L --line-numbers
poté např. sudo iptables -D INPUT 3
```

persistent:
```
apt install iptables-persistent
```
save: 
```
sudo iptables-save > /etc/iptables/rules.v4
```

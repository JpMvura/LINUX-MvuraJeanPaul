# TP – Administration SSH et Serveur Web Nginx

Machine serveur : VM Ubuntu  
Machine cliente : Hôte  

---

## Partie 2 – Serveur SSH

### Installez le serveur SSH sur la VM.  

#### Commandes utilisées
sudo apt update

sudo apt install -y openssh-server

![alt text](image.png)

### Vérifiez que le service SSH fonctionne et écoute sur un port.

#### Commandes utilisées

systemctl status ssh

![alt text](image-1.png)

ss -tlnp | grep ssh


![alt text](image-2.png)

### Connectez-vous depuis la machine hôte :

#### Commandes utilisées

ssh etudiant@IP_VM

![alt text](image-3.png)

### Générez une clé SSH sur la machine cliente et copiez-la sur le serveur pour tester la connexion sans mot de passe.

#### Commandes utilisées

ssh-keygen -t ed25519 -C "jp-tp-ssh"

![alt text](image-4.png)


ssh-copy-id etudiant@IP_VM

![alt text](image-5.png)

ssh etudiant@IP_VM
![alt text](image-6.png)

## Partie 3 – Sécurisation SSH

Modifiez la configuration SSH sur le serveur pour renforcer la sécurité :

– Interdisez l’accès root.

– Désactivez l’authentification par mot de passe.

– Changez le port par défaut (22) pour réduire les tentatives de brute-force.


#### Commandes utilisées

sudo nano /etc/ssh/sshd_config

Port 2222

PermitRootLogin no

PasswordAuthentication no

PubkeyAuthentication yes

![alt text](image-7.png)


sudo systemctl restart ssh

sudo systemctl status ssh

![alt text](image-8.png)

### Testez la connexion avec le nouveau port depuis la machine cliente.
#### Commandes utilisées
ssh -p 2222 etudiant@IP_VM

![alt text](image-9.png)

### Créez un alias SSH dans ~/.ssh/config pour simplifier les connexions.
#### Commandes utilisées

nano ~/.ssh/config
 Host tp-ssh

     HostName IP_VM

     User etudiant

     Port 2222

     IdentityFile ~/.ssh/id_ed25519

![alt text](image-10.png)


ssh tp-ssh

![alt text](image-11.png)

## Partie 4 – Transfert de fichiers
Transférez un fichier et un dossier depuis la machine cliente vers le serveur :
SCP : scp fichier.txt serveur-tp:/home/etudiant/

#### Commandes utilisées

echo "Bonjour TP SSH" > fichier.txt

![alt text](image-12.png)


scp fichier.txt etudiant@IP_VM:/home/etudiant/

![alt text](image-13.png)

#### Commandes utilisées

ssh tp-ssh

ls -l /home/etudiant/

![alt text](image-14.png)

### SFTP : explorez les commandes put, get, ls pour transférer et naviguer sur le serveur.
#### Commandes utilisées

sftp tp-ssh

 dans SFTP :

 ls

 put fichier.txt

 get fichier.txt

 mkdir dossier_sftp

 cd dossier_sftp

 ls
![alt text](image-15.png)

### RSYNC : synchronisez un dossier entre client et serveur.

#### Commandes utilisées

mkdir -p dossiertp

echo "fichier 1" > dossiertp/f1.txt

echo "fichier 2" > dossiertp/f2.txt

![alt text](image-16.png)



rsync -avz dossiertp/ tp-ssh:/home/etudiant/dossiertp/

![alt text](image-17.png)


ssh tp-ssh

ls -R /home/etudiant/dossiertp/

![alt text](image-18.png)

## Partie 5 – Analyse des logs et sécurité
### Suivez les logs d’authentification pour observer les connexions SSH :

sudo tail -f /var/log/auth.log

#### Commandes utilisées

sudo tail -f /var/log/auth.log

![alt text](image-19.png)

### Installez Fail2Ban et testez un bannissement après plusieurs tentatives échouées.

#### Commandes utilisées
sudo apt install -y fail2ban

![alt text](image-20.png)


sudo systemctl status fail2ban

![alt text](image-21.png)


sudo fail2ban-client status sshd

![alt text](image-22.png)

## Partie 6 – Tunnel SSH
### Créez un tunnel local pour accéder à un service web distant depuis la machine cliente.


ssh -L 8080:localhost:80 tp-ssh

![alt text](image-23.png)


curl http://localhost:8080

![alt text](image-24.png)

### Créez un tunnel distant pour permettre l’accès SSH au client via le serveur.
#### Commandes utilisées

ssh -R 2222:localhost:22 tp-ssh

![alt text](image-25.png)


ssh -p 2222 user_client@localhost

![alt text](image-26.png)

## Partie 7 – Nginx et HTTPS
### Installez Nginx sur la VM.
#### Commandes utilisées

sudo apt update

sudo apt install -y nginx

![alt text](image-27.png)

sudo systemctl status nginx

![alt text](image-28.png)

### Créez un site test dans /var/www/site-tp et un fichier index.html avec un message de bienvenue.
#### Commandes utilisées

sudo mkdir -p /var/www/site-tp

echo "<h1>Bienvenue sur le site TP Nginx</h1>" | sudo tee /var/www/site-tp/index.html

![alt text](image-29.png)

### Configurez Nginx pour servir ce site sur HTTP.
#### Commandes utilisées

sudo nano /etc/nginx/sites-available/site-tp
 
 server {

listen 80;

server_name _;     root /var/www/site-tp;

index index.html;

location / {

try_files $uri $uri/ =404;

}

}

![alt text](image-30.png)


sudo ln -s /etc/nginx/sites-available/site-tp /etc/nginx/sites-enabled/

sudo rm /etc/nginx/sites-enabled/default

sudo nginx -t

sudo systemctl reload nginx

![alt text](image-31.png)


curl http://IP_VM

![alt text](image-32.png)

### Générez un certificat auto-signé pour HTTPS et configurez la redirection HTTP → HTTPS.


#### Commandes utilisées

sudo mkdir -p /etc/nginx/ssl

![alt text](image-33.png)


sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/site-tp.key \
    -out /etc/nginx/ssl/site-tp.crt

![alt text](image-34.png)


sudo nano /etc/nginx/sites-available/site-tp

server {

     listen 80;#     server_name _;

     return 301 https://$host$request_uri;

 }

 server {

     listen 443 ssl;

     server_name _;

     ssl_certificate     /etc/nginx/ssl/site-tp.crt;

     ssl_certificate_key /etc/nginx/ssl/site-tp.key;

     root /var/www/site-tp;

     index index.html;

     location / {

         try_files $uri $uri/ =404;

     }

 }

![alt text](image-35.png)



sudo nginx -t

sudo systemctl reload nginx

![alt text](image-36.png)

### Testez le site depuis le client :

#### Commandes utilisées

curl -k https://IP_VM

![alt text](image-37.png)

## Partie 8 – Firewall et permissions
### Autorisez Nginx dans le firewall (ports HTTP/HTTPS).
Piste : sudo ufw allow 'Nginx Full'

#### Commandes utilisées

sudo ufw allow 'Nginx Full'

sudo ufw enable

sudo ufw status

![alt text](image-38.png)

### Vérifiez les permissions sur /var/www/site-tp pour que Nginx puisse lire les fichiers.

#### Commandes utilisées


ls -ld /var/www/site-tp

ls -l /var/www/site-tp

![alt text](image-39.png)



sudo chown -R www-data:www-data /var/www/site-tp

sudo chmod -R 750 /var/www/site-tp

![alt text](image-40.png)


ls -ld /var/www/site-tp

ls -l /var/www/site-tp

![alt text](image-41.png)

## Partie 9 – Validation finale
### SSH fonctionnel sur port personnalisé et authentification par clé uniquement.
#### Commandes utilisées

ssh -p 2222 etudiant@IP_VM

ssh tp-ssh

![alt text](image-42.png)

### Fail2Ban actif et opérationnel.
#### Commandes utilisées

sudo fail2ban-client status sshd

![alt text](image-43.png)

### Transferts de fichiers fonctionnels (SCP, SFTP, RSYNC).
#### Commandes utilisées

scp fichier.txt tp-ssh:/home/etudiant/

sftp tp-ssh

rsync -avz dossiertp/ tp-ssh:/home/etudiant/dossiertp/

![alt text](image-44.png)

### Nginx accessible en HTTP et HTTPS avec redirection automatique HTTP → HTTPS.
Certificat SSL auto-signé valide.

#### Commandes utilisées

curl -I http://IP_VM

curl -k -I https://IP_VM

![alt text](image-45.png)

### Firewall configuré et permissions correctes sur /var/www/site-tp.
#### Commandes utilisées

sudo ufw status

ls -ld /var/www/site-tp

ls -l /var/www/site-tp

![alt text](image-46.png)
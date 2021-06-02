# https://www.remipoignon.fr/generez-un-certificat-ssl-auto-signe-pour-passer-en-https/ 

# Pourquoi installer le https ?

# Le HTTPS est un protocole qui permet d’encrypter les échanges entre ton serveur et le navigateur web. Ainsi, il est impossible à un pirate de # récupérer les identifiants que tu saisis pour te loguer sur ton site. En HTTP, les identifiants passent en clair et sont facile à intercepter.

# Il nécessite de créer un certificat SSL qui permettra d’initialiser les échanges cryptés. 
# C’est payant ?

# Pour être reconnu par un navigateur web, le certificat SSL doit être signé par un organisme de certification. Ces organisme font payer cette # #certification. Depuis quelques années, Let’s encrypt permet de générer des certificats SSL gratuitement ! C’est une excellente nouvelle pour #l’évolution globale de la sécurité sur internet.

# Tu peux aussi signer toi même ton certificat SSL, il ne sera alors pas connu par les navigateurs et tu aura une page d’erreur du genre :
# Votre connexion n'est pas privée
# Page d’alerte avec un certificat auto-signé

# Dans le cas d’un site en local, ce n’est pas grave, il te suffit de passer cette alerte et tu peux consulter ton site en toute sécurité car la #couche SSL qui crypte les données fait quand même son travail.

# Par contre, si un hacker venait à prendre le contrôle de ton serveur et modifiait ton certificat SSL, il serai alors en mesure de décrypter les # échanges. Mais bon … Je pense qu’une fois le hacker sur ton serveur, il peut aisément avoir accès à tes données.

# Le cas d’un certificat auto-signé peut être intéressant pour la gestion d’un intranet. L’idée serait de générer un certificat maître auto-signé # qui signerait ensuite lui même tous les certificats SSL de l’ensemble des services utilisé dans l’intranet. C’est ce qu’on appelle une PKI # #public Key Infrastructure).

# On peut ensuite installer le certificat maître sur tout les postes des utilisateurs de l’intranet. Ces dernier pourront alors consulter tous # # les sites en HTTPS sans avoir d’alerte de sécurité et l’administrateur de l’intranet garde la main pour pouvoir ajouter de nouveaux certificats # valides.

# La gestion des PKI fera l’objet d’un article que je sortirai bientôt 🙂

# Dans cet article on va s’intéresser à la création d’un certificat simple auto-signé.
# Créer le certificat
# Prérequis

# Nous allons travailler avec un site hébergé sur un serveur Linux (avec accès root). Nous utiliserons Apache2 pour le serveur web.

# Openssl doit être installé pour générer les certificats.

sudo apt-get install openssl

# Génération de la clé privée

# On va créer une clé privée de 4096 bit encrypté avec l’algorithme de cryptage AES 256 bit.

openssl genrsa -aes256 -out certificat.key 4096

# Entres un mot de passe pour ta clé (penses à bien le conserver).

# On a généré une clé privée protégée par mot de passe, on va maintenant générer cette même clé mais déverrouillée.

# Si tu ne souhaites pas la déverrouiller, le mot de passe de la clé sera demandé à chaque redémarrage de Apache2.

# Dans un premier temps on va renommer notre clé :

mv certificat.key certificat.key.lock

# Puis générer notre certificat déverrouillé :

openssl rsa -in certificat.key.lock -out certificat.key

# Saisisses le mot de passe de la clé, le fichier déverrouillé est créé.

# Ta clé privée déverrouillée : certificat.key

# Ta clé privée verrouillée : certificat.key.lock
# Génération du fichier de demande de signature

# Ce fichier va être utile pour obtenir notre certification de l’organisme ou pour auto-signer notre certificat.

openssl req -new -key certificat.key.lock -out certificat.csr

# Renseignes les différents champs demandés comme tu veux.

# Ton fichier de demande de signature : certificat.csr
# Génération du certificat

# Pour auto-signer ton certificat, exécutes la commande suivante :

openssl x509 -req -days 365 -in certificat.csr -signkey certificat.key.lock -out certificat.crt

# Ton certificat : certificat.crt
# Indiquer à Apache d’utiliser le certificat SSL

# Première chose à faire, vérifier que le ssl est activé sur apache :

a2enmod ssl

# Il va falloir ensuite créer un vHost sur le port 443 (port du https) :
nano /etc/apache2/site-availables/site.conf

/etc/apache2/site-availables/site.conf
<VirtualHost *:80>
    ServerName      site.com
    # On redirige le port HTTP vers le port HTTPS
    Redirect        / https://www.site.com
</VirtualHost>
<VirtualHost *:443>
    ServerName      site.com
    DocumentRoot    /var/www/html
        
    SSLEngine on
    SSLCertificateFile    /etc/ssl/www/certificat.crt
    SSLCertificateKeyFile /etc/ssl/www/certificat.key
    # Facultatif, ici, on dit qu'on accepte tout les protocoles SSL sauf SSLv2 et SSLv3 (dont on accepte que le TLS ici)
    SSLProtocol all -SSLv2 -SSLv3
    # Facultatif, on dit que c'est le serveur qui donne l'ordre des algorithmes de chiffrement pendant la négociation avec le client
    SSLHonorCipherOrder on
    # Facultatif, algorithme de chiffrement disponibles (ne pas être trop méchant sinon beaucoup de navigateur un peu ancien ne pourront plus se connecter)
    SSLCipherSuite ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES
</VirtualHost>

# On active le vHost :

a2ensite tuto.conf
service apache2 restart

# C’est tout, ton site est maintenant en HTTPSO

# Incident : <à compléter : ce que les clients ont vu>
## Impact
Les clients ont vu une erreur 403 Forbidden : les pages de la boutique ne s'affichaient pas.
Ils ont aussi vu une erreur 502 Bad Gateway : le stock des produits était indisponible.


- **Date** : 8 novembre 2026
- **Durée** :  environ de 15h00 à 15h20 (heure du serveur), fin confirmée par le redémarrage de Nginx à 15h19
- **Rédigé par** : samba et ton dembele

## Impact

 clients et  l equipe service client , le 403 pour la page de promotion et 502 pour le stock , vers 15h

## Chronologie

<à compléter : l'heure de la publication, des symptômes, de chaque commande de diagnostic et de chaque correction. Une ligne par événement, par exemple : « - 17:16 publication de la version de novembre ».>


- Symptôme : curl -I http://localhost/api/stock renvoie 502 Bad Gateway.
- Diagnostic : lecture des notes de version de Koffi (NOTES-VERSION.md) : la nouvelle API lit le stock dans un fichier indiqué par la variable FICHIER_STOCK, que le service ne fournissait pas.
- Correction : ajout de Environment=FICHIER_STOCK=<chemin> dans mboa-stock.service, puis daemon-reload, reset-failed et restart.
- Vérification : curl http://localhost/api/stock renvoie le stock.

## Causes racines

<à compléter : pour chaque symptôme, le 403 et le 502 : ce que les journaux ont montré, et ce qui a rendu l'erreur possible.>

## Causes racines

**403 Forbidden sur la page des promotions**
- Ce que les journaux ont montré : mboamarket.error.log affichait « Permission denied »
  sur les nouveaux fichiers (promo.html et son affiche).
- Ce qui a rendu l'erreur possible : l'outil mboa-publier crée les fichiers avec un
  umask trop restrictif. Les nouveaux fichiers n'étaient donc pas lisibles par Nginx
  (www-data). Les anciens fichiers, copiés par Nadia, avaient les bons droits.

**502 Bad Gateway sur /api/stock**
- Ce que les journaux ont montré : Nginx n'arrivait pas à joindre l'API sur
  127.0.0.1:8090, car le service mboa-stock ne tournait plus.
- Ce qui a rendu l'erreur possible : la version de novembre de l'API a besoin de la
  variable FICHIER_STOCK, annoncée dans les notes de version de Koffi. L'unité
  systemd ne la fournissait pas, et rien ne vérifiait les notes de version avant
  la publication.
## Correction


<à compléter : ce qui a été changé, exactement : les commandes.>
## Correction

**403**
- Droits des nouveaux fichiers corrigés : lisibles par tous, modifiables par le groupe web.
  sudo chmod 664 /srv/mboamarket/site/promo.html <l'affiche>
- Outil corrigé (OPS-313) : umask 002 dans /usr/local/bin/mboa-publier,
  vérifié avec sudo mboa-publier --essai.
- Vérification : curl -I http://localhost/promo.html renvoie 200.

**502**
- Ligne ajoutée dans /etc/systemd/system/mboa-stock.service, section [Service] :
  Environment=FICHIER_STOCK=<chemin du fichier>
- sudo systemctl daemon-reload
- sudo systemctl reset-failed mboa-stock
- sudo systemctl restart mboa-stock
- Vérification : curl http://localhost/api/stock renvoie le stock.


## Pour que ça ne revienne pas

<à compléter : les actions, chacune avec un responsable.>
## Pour que ça ne revienne pas

- Garder umask 002 dans mboa-publier, et tester chaque publication avec
  sudo mboa-publier --essai avant de publier. Responsable : équipe DevOps.
- Avant chaque publication, lire les notes de version et appliquer les changements
  de configuration demandés (variables d'environnement, etc.). Responsable : la
  personne qui publie.
- Signaler clairement dans les notes de version toute nouvelle variable ou tout
  nouveau fichier nécessaire. Responsable : Koffi, développeur de l'API.
- Après chaque publication, vérifier avec curl -I la page d'accueil, les nouvelles
  pages et /api/stock. Responsable : la personne qui publie.
- Garder la configuration du serveur dans le dépôt mboa-market-serveur, et y
  enregistrer chaque modification. Responsable : équipe DevOps.

Ce post-mortem ne cherche pas de coupable : il cherche ce qui, dans les outils et les habitudes, a laissé passer l'erreur.

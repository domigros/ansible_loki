Role apps_loki
=========

Ce rôle permet d'installer une instance du produit Loki, ainsi l'outil logcli. Le fichier de configuration correspond à un stockage local.

Requirements
------------


Role Variables
--------------
loki_db_dir: "/local/loki/data/db". Nom du répertoire de stockage des données Loki.

Remarque :
----------
Le fichier de configuration se trouve : /etc/loki/loki-local-config.yaml
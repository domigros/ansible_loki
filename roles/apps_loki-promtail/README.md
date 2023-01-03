# Role apps_loki-promtail
Ce rôle permet d'installer l'agent de surveillance  loki/promtail, produisant des données pour Loki et collectable par le produit Prometheus.

Cet agent récupère les binaires de promtail sur l'url suivante : 
"https://xxxxx:xxxxx@nexus.exterieur.dassault-aviation.com:8446/repository/prometheus-packagedelivery/loki"

qui faudra modifier pour spécifier le user/mot de passe du serveur Nexus.
On pourra utiliser pour cela l'option extravars d'Ansible : 
-e -e loki_proxy_url="https://xxxxx:xxxx@nexus.exterieur.dassault-aviation.com:8446/repository/prometheus-packagedelivery/loki"

La configuration dans /etc/promtail/promtail-local-config.yaml est à retoucher pour les besoins spécifiques de la machine.
L'agent dispose d'une IHM de visualisation accessible à l'url http://<machine>:9080. (port par défaut)

## Requirements
RedHat 7/8

## Role Variable
- **loki_promtail_port** : port d'écoute de l'IHM de promtail
- **loki_host** : hostname du serveur loki à qui envoyer les données de log.
- **loki_port** : port du serveur loki à qui envoyer les données de log.
- **loki_tenant** : pour la gestion multi-environnement (non utilisé pour l'instant).
- **loki_promtail_extraconf** : permet de rajouter un paragraphe supplémentaire dans le fichier de conf de promtail, par exemple pour définir un job de scraping supplémentaire. La gestion de ce rajout est contrôlé par la variable **loki_promtail_appli**. Attention : les lignes doivent être indentées de deux caractères pour la déclaration en chaine longue 'yaml'.
- **loki_promtail_appli** : pour la gestion du rajout d'un  paragraphe dans le fichier de conf de promtail

Exemples :
1. Rajout de la remontée syslog en écoute sur le port 1514 :
```
loki_promtail_appli:  "promtail"
loki_promtail_extraconf: |
  - job_name: syslog
    syslog:
      listen_address: 0.0.0.0:1514
      idle_timeout: 60s
      label_structured_data: yes
      labels:
        job: "syslog"
    relabel_configs:
      - source_labels: ['__syslog_message_hostname']
        target_label: 'host'
      - source_labels: ["__syslog_connection_ip_address"]
        target_label: "ip_address"
      - source_labels: ["__syslog_message_severity"]
        target_label: "severity"
      - source_labels: ["__syslog_message_facility"]
        target_label: "facility"
```
    
2. Parsing de fichiers de log Squid : 
Noter l'emploi de \\{\\{ et \\}\\} dans une expression de template pour ne pas interférer avec le templating j2 d'Ansible.
```
loki_promtail_appli:  "promtail"
loki_promtail_extraconf: |+
  - job_name: appli_squid
    pipeline_stages:
    - regex:
        expression: (?P<c_ip>\S+) (?P<c_user_ident>\S+) (?P<c_user_name>\S+) \[(?P<timestamp>.*?)\] "(?P<request_method>\S+) (?P<request_url>\S+) HTTP\/(?P<request_protocol>\S+)" (?P<http_code>\d{3}) (?P<http_size>\d+) (?P<squid_request_status>\S+):(?P<nexthop_status>\S+)  "(?P<user_agent>.*?)"
    - timestamp:
        source: timestamp
        format: "02/Jan/2006:03:04:05 -0700"
    - template:
        source : message
        template: 'c_ip=\{\{ .c_ip \}\} c_user_ident=\{\{ .c_user_ident \}\} c_user_name=\{\{ .c_user_name \}\} request_method=\{\{ .request_method \}\} request_url=\{\{ .request_url \}\} request_protocol=\{\{ .request_protocol \}\} http_code=\{\{ .http_code \}\} http_size=\{\{ .http_size \}\} squid_request_status=\{\{ .squid_request_status \}\} nexthop_status=\{\{ .nexthop_status \}\} user_agent=\{\{ .user_agent \}\}'
    - output:
        source: message
    static_configs:
    - targets:
       - localhost
      labels:
        job: appli_squid
        daapplication: squid
        host:  {{ inventory_hostname }}
        __path__: "/local/squid/data/*.log"
```

## Command
promtail  est contrôlé directement par le service 'promtail'.


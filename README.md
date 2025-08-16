### Accéder à son Home Assistant hébérgé derrière sa box Internet

#### Prerequis :
1- Home assistant installé et fonctionnel depuis votre réseau interne  
2- Serveur distant avec docker , docker-compose et traefik   

### Serveur distant
Nota: un [chart](./chart/) est disponible mais c'est encore du draft   

#### Lancer votre backend derrière traefik
Ajouter dans votre `docker-compose.yml`  (adaptez le chemin `/path/data/hatunnel` )
```yaml
version: '3.7'
services:
# ..... Traefik et autres applications

  hatunnel:
    image: diyfr/hass-tunnel:latest
    container_name: hatunnel
    networks:
      - traefik
    ports:
      - 6222:2222
    restart: always
    volumes:
      - "/path/data/hatunnel:/config"
    environment:
      - PUID=1000  # se référer à la doc de l'image de base 
      - PGID=1000 # se référer à la doc de l'image de base
      - TZ=Europe/Paris # se référer à la doc de l'image de base
      - USER_PASSWORD=achanger  # se référer à la doc de l'image de base 
      - USER_NAME=myUser  # se référer à la doc de l'image de base
      - PASSWORD_ACCESS=true # se référer à la doc de l'image de base
    labels:
      - "traefik.http.routers.hatunnel.rule=Host(`home-assistant.domain.tld`)"
      - "traefik.http.routers.hatunnel.tls=true"
      - "traefik.http.routers.hatunnel.tls.certresolver=letsencrypt"
      - "traefik.http.routers.hatunnel.entrypoints=websecure"
      - "traefik.http.routers.hatunnel.middlewares=security@file" #, compression@file"
      - "traefik.http.services.hatunnel.loadbalancer.server.port=8080"
      - "traefik.http.routers.hatunnel.service=hatunnel"
      - "traefik.docker.network=traefik"
      - "traefik.http.routers.hatunnel-http.rule=Host(`home-assistant.domain.tld`)"
      - "traefik.http.routers.hatunnel-http.middlewares=https-redirect@file"
```
### Depuis votre serveur interne 

#### !IMPORTANT! conf rapide
se placer en root sur votre serveur interne, c'est le compte utilisé pour le service systemd

#### Création de clés et installation sur votre serveur distant
```shell
ssh-keygen -b 4096
```
tout valider par défaut (pas de passphrase)   
Installer votre clé sur le serveur distant 
```shell
ssh-copy-id -p 6222 myUser@home-assistant.domain.tld
```
Saisir le mot de passe injecté en variable d'environnement dans le docker-compose  (ici achanger)  
Test de la connexion  
```shell
ssh -p 6222 myUser@home-assistant.domain.tld
```
La connexion doit se faire sans le mot de passe  

#### Lancer le tunnel SSH 
```
ssh -N -T -R 8080:127.0.0.1:8123 -p 6222 myUser@home-assistant.domain.tld
```
puis tester la connexion http depuis votre serveur distant ici `https://home-assistant.domain.tld`  

On peut ensuite créer un service 
`/etc/systemd/system/run-tunnel-ha.service`
```bash
[Unit]
Description=Start HA tunnel
After=network.target

[Service]
ExecStart=/usr/bin/ssh -N -T -o ServerAliveInterval=60 -o ExitOnForwardFailure=yes -R 8080:127.0.0.1:8123 -p 6222 myUser@home-assistant.domain.tld

# Restart every >2 seconds to avoid StartLimitInterval failure
RestartSec=5
Restart=always

[Install]
WantedBy=multi-user.target
```
activer la prise en compte de ce nouveau service et le redémarrer  
```shell
systemctl daemon-reload 
systemctl enable run-tunnel-ha.service
```

Nota: Un redémarrage du service côté serveur interne me signale que le port 8080 est utilisé sur le serveur distant. J'ai du redémarrer le traefik....  



#### Config sur Home assistant
Ajoutez dans configuration.yaml
```yml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
```









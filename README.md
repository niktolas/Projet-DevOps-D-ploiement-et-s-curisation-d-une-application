# Truitteur - Infrastructure DevSecOps & Application Réseau Social

> **Projet de fin d'études — ESIEE Paris (Filière Cybersécurité, 2025-2026)**
> Conception et déploiement d'une infrastructure cloud entièrement sécurisée (*Security by Design*) et d'une chaîne CI/CD automatisée pour une application microservices Java / Spring Boot & Angular.
> 
> 

---

## Présentation du Projet

**Truitteur** est une plateforme de réseau social développée sur une architecture microservices. L'objectif principal n'était pas le développement applicatif, mais la mise en place d'une infrastructure DevSecOps moderne, résiliente et hautement sécurisée à partir de deux machines virtuelles vierges sur AWS.

Le projet s'articule autour de trois piliers fondamentaux :

1. **Ops :** Déploiement et durcissement des serveurs de production et de compilation.


2. **Sec :** Approche *Security by Design*, filtrage réseau strict, isolation des privilèges et conformité aux guides ANSSI.


3. **Dev :** Automatisation du cycle de vie du logiciel via un pipeline CI/CD intégrant des contrôles de qualité et de sécurité (SAST).



---

## Architecture System & Cartographie des Services

L'infrastructure s'appuie sur **2 instances virtuelles AWS EC2** distinctes pour garantir l'isolation des environnements de build et de production.

```
+-------------------------------------------------------+      +-------------------------------------------------------+
|              SERVEUR DE COMPILATION (CI/CD)           |      |                 SERVEUR DE PRODUCTION                 |
|                                                       |      |                                                       |
|  [ Nginx Reverse-Proxy (HTTPS / TLS 1.2 & 1.3) ]      |      | [ Nginx Reverse-Proxy (HTTPS / TLS 1.2 & 1.3) ]       |
|                         |                             |      |                         |                             |
|       +-----------------+-----------------+           |      |        +----------------+----------------+            |
|       |                 |                 |           |      |        |                |                |            |
|  [ Jenkins ]     [ SonarQube ]      [ Nexus 3 ]       |      |  [ Angular ]     [ Gateway ]    [ Auth / Post ]       |
|  (Port 8080)     (PostgreSQL BDD)   (Dépôt Artifacts) |      |  (Port 3000)     (Port 8080)    (Ports 8081/8082)     |
|                         |                             |      |                         |                             |
|            (Réseau Podman: devops-net)                |      |  [ Discovery Eureka ]        [ PostgreSQL 15 ]        |
|                                                       |      |  (Port 8761)                 (PostgreSQL BDD)         |
+-------------------------------------------------------+      +-------------------------------------------------------+

```

### Cartographie des Ports Applicatifs (Serveur de Production)

Pour éviter les conflits de liaison d'adresses (`bind: address already in use`) tout en maintenant un niveau d'isolation strict, les microservices s'exécutent dans des conteneurs Podman indépendants :

| Service | Port Interne (Conteneur) | Port Hôte (Mappage Externe) | Rôle & Exposition |
| --- | --- | --- | --- |
| **Gateway** | `8080` | `8080` | Point d'entrée unique public géré par Spring Cloud Gateway.

 |
| **Authentication** | `8080` | `8081` | Service de gestion des comptes et jetons (Accessible via Gateway).

 |
| **Post / Tweets** | `8080` | `8082` | Service de publication de tweets (Accessible via Gateway).

 |
| **Front Angular** | `80` | `3000` | Interface utilisateur statique Angular servie par Nginx.

 |
| **Discovery (Eureka)** | `8761` | `8761` | Annuaire d'enregistrement dynamique des services Spring.



---

## Socle de Sécurité & Ops (*Security by Design*)

### 1. Durcissement des Accès Système (SSH & ANSSI)

* **Suppression de la surface d'attaque directe :** Désactivation totale du compte `root` (`PermitRootLogin no`) et interdiction stricte de l'authentification par mot de passe (`PasswordAuthentication no`).


* **Authentification sur biclé RSA :** Accès nominatif via clé SSH privée (RSA 4096-bit). Redondance d'accès gérée par concaténation des clés publiques du groupe dans `authorized_keys`.


* **Changement du port par défaut :** Migration du service SSH sur le port **52819** (plage dynamique) pour éviter le bruit des scans automatiques de bots.


* **Gestion des privilèges :** Création d'utilisateurs système dédiés sans shell d'exécution (`javauser`, `nexus`) et accès d'administration restreint au groupe `asystes` via `sudo`.



### 2. Filtrage Réseau Strict (Pare-feu Iptables)

Application d'une politique **Default DROP** sur l'ensemble des chaînes (`INPUT`, `FORWARD`, `OUTPUT`) :

* **Flux Entrants (`INPUT`) :** Seuls les ports SSH `52819`, HTTP `80` et HTTPS `443` sont autorisés depuis l'extérieur. La loopback (`lo`) est ouverte pour la communication inter-processus locale.


* **Flux Sortants (`OUTPUT`) :** Seuls les flux vitaux sont whitelistés : DNS (`UDP/TCP 53`), HTTP/HTTPS sortants (`80/443` pour les paquets système et dépendances), et NTP (`UDP 123` pour la synchronisation de l'heure et la validité des certificats).


* **Suivi de Connexion :** Module `conntrack` placé en priorité pour autoriser la réception des paquets issus de connexions déjà établies (`ESTABLISHED, RELATED`).



### 3. Exposition Sécurisée (Nginx Reverse-Proxy)

* **Terminaison SSL/TLS :** Seuls les protocoles **TLSv1.2** et **TLSv1.3** sont autorisés. Désactivation des algorithmes dépréciés.


* **Redirection HTTPS Forcée :** Redirection permanente `301` du port `80` vers le port `443`.


* **En-têtes de Sécurité (OWASP Guidelines) :** Injection de `Strict-Transport-Security` (HSTS), `X-Frame-Options: SAMEORIGIN` (anti-clickjacking), `X-Content-Type-Options: nosniff`, et `X-XSS-Protection`.


* **Anti-DoS & Rate Limiting :** Restriction de la taille du corps de requête (`client_max_body_size 8m` sur la prod, ajusté à `1G` sur Nexus pour les artefacts `.jar`) et masquage complet de la signature Nginx (`server_tokens off`).



---

## Chaîne d'Intégration et de Déploiement Continus (CI/CD)

Le serveur de compilation orchestre un pipeline **Jenkins** entièrement conteneurisé.

```
  +------------+      +-------------------+      +---------------+      +-------------------+
  | 1. CHECKOUT| ---> | 2. NEXUS CHECK    | ---> | 3. BUILD      | ---> | 4. SAST SCAN      |
  | Git Clone  |      | Verification ver. |      | Maven / NPM   |      | SonarQube         |
  +------------+      +-------------------+      +---------------+      +-------------------+
                                                                                  |
  +------------+      +-------------------+      +---------------+                v
  | 7. DEPLOY  | <--- | 6. PUBLISH        | <--- | 5. QUALITY    | <-----------------+
  | SCP + Pod  |      | Nexus Snapshot    |      |    GATE       |
  +------------+      +-------------------+      +---------------+

```

### Étape par Étape du Pipeline (`Jenkinsfile`)

1. **Importation Git :** Récupération dynamique du code source depuis le dépôt Git avec des identifiants stockés dans le coffre Jenkins.


2. **Vérification de Version dans Nexus :** Exécution d'une requête REST / `curl` avec `mvn help:evaluate`. Si l'artefact existe déjà en version `RELEASE` dans Nexus, le build est contourné pour économiser des ressources.


3. **Build & Package :** Compilation du code source Java (`mvn clean package -DskipTests`) ou transpilation Angular avec support d'anciennes clés crypto (`--ssl-legacy-provider`).


4. **Analyse de Code Statique (SAST) :** Inspection automatique par **SonarScanner** pour remonter la dette technique, les bugs, les *code smells* et les vulnérabilités.


5. **Quality Gate & Webhooks :** Suspension du pipeline via `waitForQualityGate`. SonarQube notifie Jenkins par Webhook dès que l'analyse est validée.


6. **Publication d'Artefact :** Envoi du livrable `.jar` vers le dépôt `maven-snapshots` de **Sonatype Nexus 3**.


7. **Déploiement Continu en Production :**
* Connexion SSH à la machine de production sur le port `52819`.


* Transfert du fichier `.jar` compilé via `scp`.


* Redémarrage propre du conteneur applicatif **Podman** correspondant et injection des variables d'environnement réseau et de base de données.





---

## Conteneurisation & Isolation (Podman)

Pour éviter les failles de sécurité liées au démon central de Docker, l'ensemble des conteneurs est exécuté via **Podman** en mode **Rootless** :

* **Serveur CI/CD :** Les conteneurs `jenkins`, `sonarqube`, `sonar-db` (PostgreSQL) et `nexus` sont connectés sur un réseau privé interne Podman (`devops-net`).


* **Serveur de Production :** Les microservices (`gateway`, `authentication`, `post`, `angular-front`, `registry`) communiquent de manière isolée via le réseau bridge Podman (`esiee-network`).


* **Résolution des Services :** Utilisation de l'option `EUREKA_INSTANCE_PREFER_IP_ADDRESS=true` pour forcer l'annuaire Eureka à router le trafic inter-conteneurs par leurs adresses IP routables.



---

## Membres de l'Équipe ("Les Asystes")

* **Nicolas WILLIAME** — Configuration Ops, Déploiements & Sécurisation Java


* **Rayan ZABAÏOU** — Durcissement Réseau, SSH, Iptables & Rédaction du rapport


* **Zélia TITON** — Administration Système & Pipelines CI/CD


* **Théo ZHOU** — Architecture Jenkins, Podman, SonarQube & Configuration Nexus



---

## Documentation & Fichiers Utiles

*  **Rapport de projet complet :** [`Projet-Truitteur.pdf`](https://www.google.com/search?q=./Projet-Truitteur.pdf) (Contient le détail des configurations Iptables, Nginx, Jenkinsfile et le retour d'expérience complet).


*  **Configurations d'exemple :** Consulter la section *Annexes* du rapport pour voir le détail des scripts de déploiement et des configurations de sécurité (`sshd_config`, `nginx.conf`, `config.xml`).

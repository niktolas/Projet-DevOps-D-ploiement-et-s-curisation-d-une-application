# Projet Truitteur - Infrastructure DevSecOps

> **Projet de 4ème année à l'ESIEE Paris en filière Cybersécurité.**
> --> Mise en place d'une infrastructure complète "Security by Design" pour une application Java/Spring Boot.

---

## 📑 Présentation du Projet
Truitteur est un projet visant à déployer et sécuriser une application de réseau social à partir de zéro sur une infrastructure cloud. L'objectif principal était de construire un socle robuste en suivant les principes de **minimisation de la surface d'attaque** et d'**automatisation sécurisée**.

### 🏗️ Architecture & Stack Technique
L'infrastructure repose sur deux environnements distincts hébergés sur **AWS** :

* **Serveur de Compilation (CI/CD) :** Jenkins, SonarQube, Sonatype Nexus.
* **Serveur de Production :** Nginx (Reverse-Proxy), Java 17 JRE Headless, PostgreSQL 15.
* **Conteneurisation :** Orchestration via **Podman** en mode Rootless.

---

## 🔒 Sécurisation du Socle (Ops)
Le projet applique les recommandations de l'**ANSSI** pour le durcissement système :

* **Accès SSH durci :** Désactivation du compte `root` et de l'authentification par mot de passe. Connexion exclusive par clés RSA sur un port personnalisé.
* **Filtrage Réseau :** Configuration d'un pare-feu **Iptables** extrêmement restrictif (Politique par défaut : DROP). Seuls les flux vitaux (HTTPS, DNS, NTP) sont autorisés.
* **Reverse-Proxy Nginx :** Terminaison SSL/TLS (v1.2 & v1.3 uniquement), protection contre le clickjacking et injection d'en-têtes de sécurité (HSTS, X-Content-Type-Options).

---

## 🚀 Pipeline CI/CD (DevSecOps)
La chaîne de livraison automatise la sécurité à chaque étape :

1.  **Build :** Compilation Maven.
2.  **Analyse Statisique (SAST) :** Scan de vulnérabilités via **SonarQube**. 
3.  **Gestion d'Artefacts :** Stockage sécurisé dans **Nexus**.
4.  **Déploiement Continu :** Transfert automatique via SSH/SCP et déploiement dans des conteneurs isolés sur le serveur de production.

---

## 👥 Équipe "Les Asystes"
* Nicolas WILLIAME
* Rayan ZABAÏOU
* Zélia TITON
* Théo ZHOU

---

📄 *Pour plus de détails techniques, consultez le [Rapport Complet](./Projet-Truitteur.pdf).*

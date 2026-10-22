# PHP2Shell — Vérification et mitigation d'un home lab

> Réponse à **CVE-2026-109618** et **CVE-2026-109619** (« PHP2Shell »), divulguées le 21 octobre 2026.
> Ce dépôt documente comment j'ai vérifié mon home lab, et propose une checklist simple que tout le monde peut suivre.

---

## 🚨 C'est quoi, PHP2Shell ?

Deux vulnérabilités critiques (CVSS v4.0 **9.2**) dans le moteur PHP lui-même, qui permettent une **exécution de code à distance sans authentification**.

| CVE | Type | Impact |
|---|---|---|
| CVE-2026-109618 | Corruption mémoire lors d'une erreur *out-of-memory* | RCE avec les privilèges du processus web |
| CVE-2026-109619 | État global périmé (*use-after-free*) | Fuite de mémoire et RCE |

**Pourquoi c'est grave :**
- Toute installation PHP qui **sert des requêtes web** est touchée : PHP-FPM, Apache `mod_php`, `php-cgi`, FrankenPHP, LiteSpeed.
- Pratiquement toutes les versions des dix dernières années sont vulnérables.
- Avec `mod_php`, l'attaque peut se déclencher **avant même que le code de l'application s'exécute**. Une application « bien codée » n'est donc pas protégée.
- Les détails ont fuité avant la date prévue, et une **exploitation active** a été observée dès la sortie du correctif.

### Versions corrigées

| Branche | Version corrigée |
|---|---|
| 8.5 | **8.5.12** |
| 8.4 | **8.4.27** |
| 8.3 | **8.3.36** |
| 8.2 | **8.2.35** |
| ≤ 8.1 | Plus supportée : mettre à niveau, ou retirer du réseau |

---

## 🏠 Mon home lab : inventaire et vérification

### Inventaire

| Composant | Hébergement | Utilise PHP ? | Exposé à Internet ? |
|---|---|---|---|
| Raspberry Pi 5 (OS hôte) | Physique | Non (à vérifier) | Non |
| Pi-hole | Docker | Non (v6) / Oui (v5) | Non |
| Portainer | Docker | Non (Go) | Non |
| wg-easy (WireGuard) | Docker | Non (Node.js) | **Oui** (UDP WireGuard seulement) |
| Uptime Kuma | Docker | Non (Node.js) | Non |
| Homepage | Docker | Non (Next.js) | Non |
| VM GLPI (labo Ansible) | VM | **Oui** | Non |

### Vérifications

- [ ] Aucun conteneur ne contient PHP (commande ci-dessous)
- [ ] Version de Pi-hole confirmée (v6 = pas de PHP)
- [ ] OS du Pi à jour (`apt upgrade`)
- [ ] Images Docker mises à jour (`docker compose pull`)
- [ ] Redirections de ports du routeur : seulement le port WireGuard
- [ ] VM GLPI mise à jour, ou éteinte en attendant
- [ ] Version de PHP confirmée ≥ version corrigée partout où PHP existe

**Résultat :** _à compléter_

---

## ✅ Checklist pour tout le monde

### 1. Trouver PHP partout où il se cache

PHP est souvent intégré dans d'autres logiciels (WordPress, GLPI, iTop, Nextcloud, phpMyAdmin…).

**Linux / VM :**
```bash
php -v
dpkg -l | grep -i php          # Debian / Ubuntu
rpm -qa | grep -i php          # RHEL / Rocky / Alma
```

**Conteneurs Docker :**
```bash
for c in $(docker ps -q); do
  echo "$(docker inspect -f '{{.Name}}' $c): $(docker exec $c php -v 2>/dev/null | head -1)"
done
```

**Windows (XAMPP, WAMP, Laragon) :**
```powershell
php -v
Test-Path C:\xampp\php
```

### 2. Mettre à jour

**Debian / Ubuntu / Raspberry Pi OS :**
```bash
sudo apt update && sudo apt upgrade -y
sudo systemctl restart apache2        # si Apache
sudo systemctl restart php*-fpm       # si PHP-FPM
php -v                                # confirmer la version
```

**Docker :**
```bash
docker compose pull
docker compose up -d --force-recreate
```
> Une image construite avec un `Dockerfile` (`FROM php:...`) doit être **reconstruite** : `docker compose build --pull`.

**XAMPP / WAMP / Laragon :** installer la version qui inclut un PHP corrigé. Si elle n'est pas encore disponible, **arrêter Apache** en attendant.

> Les distributions Linux ont reçu les correctifs avant la divulgation publique. Passez par les paquets officiels de votre distribution.

### 3. Réduire l'exposition

- Vérifier les **redirections de ports** du routeur : aucun 80/443 vers une machine non patchée.
- Si un service PHP exposé à Internet ne peut pas être patché tout de suite : **le mettre hors ligne**.
- Un WAF peut aider temporairement, mais **la mise à jour est le seul vrai correctif**.

### 4. Vérifier qu'on n'a pas déjà été touché (systèmes exposés)

- Chercher des fichiers `.php` récents ou inconnus dans les dossiers web.
- Vérifier les processus lancés par l'utilisateur web (`www-data`, `apache`, `nginx`).
- Regarder les logs pour des User-Agents contenant `php2shell`.

```bash
sudo grep -i "php2shell" /var/log/apache2/access.log* /var/log/nginx/access.log* 2>/dev/null
sudo find /var/www -name "*.php" -mtime -3
ps -u www-data -f
```

---

## 📚 Sources

- [CISA — Critical Vulnerabilities in PHP Require Immediate Patching](https://www.cisa.gov/news-events/alerts/2026/10/21/critical-vulnerabilities-php-require-immediate-patching)
- [Wiz Research — PHP2Shell](https://www.wiz.io/blog/php2shell-php-rce)
- [BleepingComputer](https://www.bleepingcomputer.com/news/security/php-ships-emergency-patches-for-critical-php2shell-flaws/)
- [Help Net Security](https://www.helpnetsecurity.com/2026/10/21/php2shell-php-vulnerabilities-patched-cve-2026-109618-cve-2026-109619/)
- [Armis — How to Find, Prioritize, and Patch](https://www.armis.com/blog/php2shell-how-to-find-prioritize-and-patch-affected-systems/)

---

*Rédigé par [Chahd](https://github.com/Chahdmk), étudiante en réseautique et cybersécurité.*

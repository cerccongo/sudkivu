# Site Web Officiel — Province du Sud-Kivu

Site web officiel du Gouvernement Provincial du Sud-Kivu, République Démocratique du Congo.

🌐 **URL** : [https://sudkivu.cd](https://sudkivu.cd)

---

## À propos

Ce dépôt contient le code source du site web officiel de la Province du Sud-Kivu. Le site est un ensemble de pages HTML statiques couvrant les institutions, le gouvernement provincial, les territoires, les actualités et les services aux citoyens.

---

## Structure du site

| Section | Description |
|---------|-------------|
| Accueil | Page principale (`index.html`) |
| Gouverneur | Profil et agenda du Gouverneur |
| Vice-Gouverneur | Profil et activités du Vice-Gouverneur |
| Gouvernement | Membres du gouvernement provincial |
| Assemblée | Assemblée provinciale |
| Ministères | Pages des 10 ministères provinciaux |
| Territoires | 8 territoires, villes et communes |
| Chefferies & Secteurs | Entités administratives locales |
| Actualités | Communiqués et dépêches officielles |
| Investissement | Opportunités d'investissement |
| Contact | Coordonnées et formulaire de contact |

---

## Déploiement (Cloudflare Workers)

Le site est hébergé sur **Cloudflare Workers** via [Wrangler](https://developers.cloudflare.com/workers/wrangler/).

### Prérequis

- [Node.js](https://nodejs.org/) (v18+)
- Compte [Cloudflare](https://cloudflare.com)
- Wrangler CLI

```bash
npm install -g wrangler
```

### Déployer

```bash
# Authentification Cloudflare
wrangler login

# Déploiement en production
wrangler deploy
```

La configuration est définie dans `wrangler.toml`. Le site publie automatiquement tous les fichiers du dépôt en tant qu'assets statiques.

### Déploiement continu (CI/CD)

Chaque push sur la branche `main` déclenche un déploiement automatique via GitHub Actions.

---

## Développement local

```bash
# Prévisualiser le site localement
wrangler dev
```

Le site sera accessible sur `http://localhost:8787`.

---

## Mise à jour du contenu

Pour modifier une page existante :
1. Éditez le fichier `.html` correspondant
2. Faites un commit et poussez sur `main`
3. Le redéploiement s'effectue automatiquement

---

## Déploiement alternatif — Digital Ocean Droplet

Le site peut également être hébergé sur un **Droplet Digital Ocean** en utilisant **Nginx** comme serveur web.

### Prérequis

- Un compte [Digital Ocean](https://www.digitalocean.com/)
- Un Droplet Ubuntu 22.04 LTS (ou supérieur) créé et accessible
- Accès SSH au Droplet
- `rsync` installé sur votre machine locale

### 1. Créer un Droplet

Depuis le panneau de contrôle Digital Ocean :

1. Cliquez sur **Create → Droplets**
2. Choisissez l'image **Ubuntu 22.04 LTS**
3. Sélectionnez le plan souhaité (le plan de base à 6 $/mois suffit pour un site statique)
4. Ajoutez votre clé SSH publique pour l'authentification
5. Cliquez sur **Create Droplet**

Notez l'adresse IP publique du Droplet (ex. `203.0.113.10`).

### 2. Configurer le serveur Nginx

Connectez-vous en SSH et installez Nginx :

```bash
ssh root@<IP_DU_DROPLET>

# Mettre à jour le système et installer Nginx
apt update && apt upgrade -y
apt install -y nginx

# Activer et démarrer Nginx
systemctl enable nginx
systemctl start nginx
```

Créez le répertoire racine du site :

```bash
mkdir -p /var/www/sudkivu
chown -R www-data:www-data /var/www/sudkivu
```

Créez la configuration Nginx :

```bash
nano /etc/nginx/sites-available/sudkivu
```

Collez la configuration suivante (remplacez `sudkivu.cd` par votre domaine) :

```nginx
server {
    listen 80;
    server_name sudkivu.cd www.sudkivu.cd;

    root /var/www/sudkivu;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # Mise en cache des assets statiques
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|mp4)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }
}
```

Activez le site et rechargez Nginx :

```bash
ln -s /etc/nginx/sites-available/sudkivu /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

### 3. Déployer les fichiers

Depuis votre machine locale, utilisez `rsync` pour transférer les fichiers du dépôt vers le Droplet :

```bash
# Cloner le dépôt (si ce n'est pas déjà fait)
git clone https://github.com/cerccongo/sudkivu.git
cd sudkivu

# Synchroniser les fichiers avec le serveur
rsync -avz --exclude='.git' --exclude='.github' \
  ./ root@<IP_DU_DROPLET>:/var/www/sudkivu/
```

Le site sera accessible à `http://<IP_DU_DROPLET>` ou à votre domaine une fois le DNS configuré.

### 4. Configurer HTTPS avec Let's Encrypt (recommandé)

```bash
ssh root@<IP_DU_DROPLET>

apt install -y certbot python3-certbot-nginx
certbot --nginx -d sudkivu.cd -d www.sudkivu.cd
```

Certbot configure automatiquement le renouvellement automatique du certificat SSL.

### 5. Déploiement continu via GitHub Actions (optionnel)

Ajoutez les secrets suivants dans les paramètres du dépôt GitHub :
- `DO_HOST` : adresse IP du Droplet
- `DO_SSH_KEY` : clé SSH privée (contenu du fichier `~/.ssh/id_rsa`)

Créez le fichier `.github/workflows/deploy-droplet.yml` :

```yaml
name: Deploy to Digital Ocean Droplet

on:
  push:
    branches: ["main"]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy via rsync
        uses: easingthemes/ssh-deploy@v5
        with:
          SSH_PRIVATE_KEY: ${{ secrets.DO_SSH_KEY }}
          REMOTE_HOST: ${{ secrets.DO_HOST }}
          REMOTE_USER: root
          TARGET: /var/www/sudkivu/
          EXCLUDE: ".git/, .github/"
```

Chaque push sur `main` déploiera automatiquement le site sur le Droplet.

---

## Technologies utilisées

- HTML5 / CSS3 / JavaScript (vanilla)
- [Leaflet.js](https://leafletjs.com/) — cartes interactives
- [Google Fonts](https://fonts.google.com/) — typographie
- [Cloudflare Workers](https://workers.cloudflare.com/) — hébergement et CDN
- [Digital Ocean](https://www.digitalocean.com/) — hébergement alternatif sur Droplet

---

*Gouvernement Provincial du Sud-Kivu — RDC*
*Mandature 2024–2028*

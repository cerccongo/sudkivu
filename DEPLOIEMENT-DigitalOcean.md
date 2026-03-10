# Guide de Déploiement — Site Web Province du Sud-Kivu
## Hébergement sur Digital Ocean

---

## Option 1 : Digital Ocean App Platform (Recommandée — la plus simple)

### Étape 1 – Préparer le dépôt GitHub
1. Créez un compte GitHub (gratuit) sur https://github.com
2. Créez un nouveau dépôt public ou privé nommé `sudkivu-gov`
3. Uploadez le fichier `sudkivu-gov.html` dans ce dépôt
4. Renommez-le en `index.html`

### Étape 2 – Créer une App sur Digital Ocean
1. Connectez-vous sur https://cloud.digitalocean.com
2. Cliquez sur **"Create"** → **"Apps"**
3. Sélectionnez **GitHub** comme source
4. Autorisez Digital Ocean à accéder à votre dépôt
5. Choisissez le dépôt `sudkivu-gov` et la branche `main`

### Étape 3 – Configurer l'App
- **Type** : Static Site
- **Build Command** : (laisser vide)
- **Output Directory** : `/`
- **Index Document** : `index.html`
- **Plan** : Starter (GRATUIT — $0/mois pour les sites statiques)

### Étape 4 – Déployer
1. Cliquez sur **"Next"** puis **"Create Resources"**
2. Digital Ocean déploie automatiquement en ~2 minutes
3. Votre site est accessible via l'URL fournie (ex: `https://sudkivu-gov-xxxxx.ondigitalocean.app`)

---

## Option 2 : Digital Ocean Droplet (Serveur dédié)

### Créer un Droplet Ubuntu
```bash
# Sur Digital Ocean, créer un Droplet :
# OS : Ubuntu 22.04 LTS
# Plan : Basic — $4/mois (512 MB RAM, 10 GB SSD)
# Datacenter : Frankfurt (Europe) ou Singapore (Afrique de l'Est)
```

### Installer Nginx
```bash
ssh root@VOTRE_IP_DROPLET
apt update && apt upgrade -y
apt install nginx -y
systemctl enable nginx
systemctl start nginx
```

### Déployer le site
```bash
# Copier le fichier HTML
cp sudkivu-gov.html /var/www/html/index.html
chmod 644 /var/www/html/index.html

# Configurer Nginx
cat > /etc/nginx/sites-available/sudkivu << 'EOF'
server {
    listen 80;
    server_name votre-domaine.cd www.votre-domaine.cd;
    root /var/www/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
    
    gzip on;
    gzip_types text/html text/css application/javascript;
}
EOF

ln -s /etc/nginx/sites-available/sudkivu /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

### Configurer HTTPS (SSL gratuit avec Let's Encrypt)
```bash
apt install certbot python3-certbot-nginx -y
certbot --nginx -d votre-domaine.cd -d www.votre-domaine.cd
```

---

## Option 3 : Digital Ocean Spaces (CDN statique)

1. Créer un **Space** dans Digital Ocean
2. Activer **"Static website"** dans les paramètres
3. Uploader `index.html`
4. Activer le **CDN** pour des performances mondiales
5. Configurer un domaine personnalisé

---

## Domaine Personnalisé (Recommandé)

Pour un domaine `.cd` (Congo) :
- Enregistrez votre domaine sur https://nic.cd
- Ou utilisez un domaine international : `.org`, `.gov`, `.info`
- Ajoutez-le dans Digital Ocean sous **"Networking"** → **"Domains"**

Exemple de domaine suggéré : `gouvernement.sud-kivu.cd`

---

## Coûts Estimés

| Option | Coût mensuel |
|--------|-------------|
| App Platform (site statique) | **GRATUIT** |
| Droplet Basic (1 CPU, 512MB) | ~$4 USD/mois |
| Droplet Standard (1 CPU, 1GB) | ~$6 USD/mois |
| Spaces (CDN) + 250 GB | ~$5 USD/mois |
| Domaine .cd/an | ~$10–20 USD/an |

---

## Support et Maintenance

- **Mises à jour du contenu** : Modifier `index.html` et pousser sur GitHub (redéploiement automatique)
- **Sauvegardes** : Activez les sauvegardes automatiques dans Digital Ocean ($1/mois)
- **Monitoring** : Configurer des alertes de disponibilité gratuitement

---

*Document préparé pour le Gouvernement Provincial du Sud-Kivu — RDC*
*Mandature 2024–2028*

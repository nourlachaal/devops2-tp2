# TP2 — Orchestrer une application multi-conteneurs avec Docker Compose

**Module :** DevOps 2 (ECUE733)
**Étudiante :** Nour Lachaal

---

## Partie 1 — Découvrir les deux applications

### Q1. Variables d'environnement attendues par Strapi pour PostgreSQL

Dans `backend/config/database.js`, Strapi lit ces variables d'environnement :

- `DATABASE_CLIENT` : le type de base de données (ici `postgres`)
- `DATABASE_HOST` : l'adresse du serveur de base de données
- `DATABASE_PORT` : le port (5432 pour PostgreSQL)
- `DATABASE_NAME` : le nom de la base
- `DATABASE_USERNAME` : l'utilisateur
- `DATABASE_PASSWORD` : le mot de passe
- `DATABASE_SSL` (et d'autres options facultatives comme le schéma ou le pool de connexions)

**Valeur de `DATABASE_HOST` : `postgres`**, c'est-à-dire le nom du service PostgreSQL dans `docker-compose.yml`.

**Pourquoi pas `localhost` ?** Dans un conteneur, `localhost` désigne le conteneur lui-même, pas la machine hôte ni les autres conteneurs. Strapi tourne dans son propre conteneur, et PostgreSQL dans un autre. Docker Compose fournit un DNS interne qui résout le nom de chaque service vers l'adresse IP de son conteneur. Il faut donc utiliser le nom du service (`postgres`).

### Q2. Rôle de `backend/.env.example` et pourquoi ne jamais commiter `.env`

`.env.example` est un **modèle** qui documente les variables nécessaires à l'application, avec des valeurs d'exemple ou factices. Il est versionné pour que toute personne qui clone le projet sache quelles variables définir : elle le copie en `.env` et le complète.

Le vrai fichier `.env` contient des **secrets** (mots de passe, clés JWT, sels de chiffrement, tokens). Il ne doit jamais être commité car :

- les secrets deviendraient visibles par toute personne ayant accès au dépôt (surtout s'il est public) ;
- ils resteraient dans l'historique Git même après suppression du fichier ;
- chaque environnement (développement, production) doit avoir ses propres secrets.

C'est pourquoi `.env` est listé dans `.gitignore`.

---

## Partie 2 — PostgreSQL

### Q3. Différence entre `docker compose down` et `docker compose down -v`

- `docker compose down` supprime les **conteneurs** et les **réseaux** de la stack, mais **conserve les volumes nommés**. Les données persistent.
- `docker compose down -v` supprime en plus les **volumes** (ici `pgdata`). Les données sont **définitivement perdues**.

**Test effectué :**

1. J'ai démarré PostgreSQL puis créé une table `test` contenant une ligne.
2. Après `docker compose down` puis `docker compose up -d postgres`, la commande `\dt` affichait toujours la table `test` : les données avaient survécu grâce au volume `pgdata`.
3. Après `docker compose down -v` puis `docker compose up -d postgres`, la commande `\dt` retournait « Did not find any relations » : le volume avait été supprimé, la base était vide.

### Q4. Pourquoi `$${POSTGRES_USER}` avec deux `$` dans le healthcheck ?

Docker Compose interprète lui-même la syntaxe `${VARIABLE}` et la remplace par la valeur du fichier `.env` **au moment de lire** le `docker-compose.yml`.

Dans le healthcheck, on veut que la variable soit résolue **à l'intérieur du conteneur**, par le shell, au moment où la commande `pg_isready` s'exécute. Le double `$$` est un **échappement** : Compose le transforme en un seul `$` littéral et ne le remplace pas. Le conteneur reçoit donc `pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}` et son shell utilise les variables d'environnement du conteneur.

Avec un seul `$`, c'est Compose qui substituerait la valeur du `.env` dès le départ, ce qui ne garantit pas que ce soit la même valeur que celle vue dans le conteneur.
## Partie 3 — pgAdmin

### Q5. `"Host": "postgres"`, `:ro` et `${PGADMIN_PORT:-5050}`

- **`"Host": "postgres"`** : pgAdmin tourne dans son propre conteneur. Il joint PostgreSQL via le réseau Docker, où le nom du service (`postgres`) est résolu par le DNS interne de Compose. `localhost` désignerait le conteneur pgAdmin lui-même.
- **`:ro`** (*read-only*) : le fichier `servers.json` est monté en **lecture seule** dans le conteneur. pgAdmin peut le lire pour pré-enregistrer le serveur, mais ne peut pas le modifier, ce qui protège le fichier de l'hôte.
- **`${PGADMIN_PORT:-5050}`** : syntaxe de valeur par défaut. Compose utilise la valeur de `PGADMIN_PORT` si elle est définie et non vide dans `.env`, sinon il utilise `5050`. Le port 5050 de l'hôte est redirigé vers le port 80 du conteneur.
## Partie 4 — Strapi

### Q6. Pourquoi copier `package*.json` et lancer `npm ci` avant le reste du code ?

Docker construit l'image par **couches** et met chaque couche en **cache** tant que ses entrées n'ont pas changé. En copiant d'abord uniquement `package.json` et `package-lock.json`, la couche `npm ci` (longue, qui télécharge les dépendances) n'est rejouée que si les dépendances changent. Si on ne modifie que le code source, `npm ci` est réutilisé depuis le cache et le build est beaucoup plus rapide.

**Test effectué :** j'ai ajouté un commentaire dans `backend/config/server.js` et relancé le build.
- Étapes en cache (`CACHED`) : [image de base, WORKDIR, COPY package*.json, RUN npm ci, ...]
- Étapes rejouées : [COPY . ., RUN npm run build, COPY --from=build ...]

### Q7. À quoi servent `USER node` et `--chown=node:node` ?

- `USER node` : le processus Strapi s'exécute avec l'utilisateur non privilégié `node` au lieu de `root`. C'est le principe du **moindre privilège** : si l'application est compromise, l'attaquant a des droits limités dans le conteneur.
- `--chown=node:node` : les fichiers copiés depuis l'étape de build appartiennent par défaut à `root`. Cette option les donne à l'utilisateur `node`, qui peut ainsi les lire et écrire (par exemple les dossiers temporaires et les uploads) sans être root.

### Q8. Que fait `depends_on: condition: service_healthy` ?

Il indique à Compose de **ne démarrer `strapi` que lorsque le healthcheck de `postgres` est au statut `healthy`**, c'est-à-dire quand la base accepte réellement les connexions, et pas seulement quand le conteneur est lancé.

Sans cette condition (ou avec un simple `depends_on` sans condition), Compose attendrait juste que le conteneur PostgreSQL soit démarré. Strapi risquerait de tenter de se connecter alors que la base s'initialise encore, ce qui provoque des erreurs de connexion et un plantage ou des redémarrages.

### Q9. Table `articles` dans pgAdmin

Dans pgAdmin (Databases → strapi → Schemas → public → Tables → articles), j'ai trouvé [N] lignes pour un article publié. [Explication : avec Draft & Publish de Strapi 5, une ligne correspond à la version brouillon et une autre à la version publiée du même document (même `document_id`).]

Capture : `screenshots/pgadmin-articles.png` 
## Partie 5 — Next.js

### Q10. Pourquoi `STRAPI_URL=http://localhost:1337` ne fonctionnerait-il pas dans le conteneur ?

Le code de `page.tsx` s'exécute **côté serveur, dans le conteneur `frontend`**. Dans ce conteneur, `localhost` désigne le conteneur frontend lui-même, où rien n'écoute sur le port 1337. Strapi tourne dans un autre conteneur. Il faut utiliser le **nom du service** (`http://strapi:1337`), que le DNS interne de Docker Compose résout vers l'adresse du conteneur Strapi.

### Q11. Dockerfile multi-stage et sortie `standalone`

- **Nombre d'étapes :** 3 `FROM` (`deps`, `builder`, `runner`).
- **Contenu de l'image finale :** uniquement le dossier `public/`, le serveur autonome `.next/standalone` (serveur Node minimal avec seulement les dépendances nécessaires) et `.next/static`. Les `node_modules` complets, les sources et les outils de build ne sont pas inclus.
- **Taille de l'image finale :** [taille relevée avec `docker images tp2/frontend`].
- **Sans `output: "standalone"` :** le `next build` réussit mais le build Docker échoue à l'étape `COPY --from=builder /app/.next/standalone ./` : [message d'erreur relevé]. En effet, le dossier `.next/standalone` n'est généré que lorsque cette option est activée.

Capture : `screenshots/frontend-articles.png`
## Partie 6 — Réseaux et sécurité

### Q12. Pourquoi `strapi` est-il dans les deux réseaux ? Pourquoi ne publier aucun port pour `postgres` ?

- **Strapi dans les deux réseaux :** il fait le pont entre les deux zones. Il doit joindre PostgreSQL (réseau `backend`) et être joignable par le frontend (réseau `frontend`). Le frontend, lui, n'est que dans `frontend` : il ne peut parler qu'à Strapi et ne voit pas la base de données.
- **Aucun port publié pour `postgres` :** la base n'est accessible que par les conteneurs du réseau `backend`, et pas depuis la machine hôte ni depuis l'extérieur. Cela réduit la **surface d'attaque** : on ne peut pas tenter de se connecter directement à la base depuis le réseau.

**Vérification :** depuis le conteneur `frontend`, la connexion à `postgres:5432` échoue ([sortie relevée]), alors que `http://strapi:1337/_health` répond OK.

### Q13. Deux mesures pour durcir la stack en production

1. **Gérer les secrets proprement** : régénérer chaque secret (`openssl rand -base64 16`) et utiliser des Docker secrets ou un gestionnaire de secrets plutôt qu'un fichier `.env` en clair.
2. **Placer un reverse proxy HTTPS** (Nginx, Traefik) devant la stack, ne plus exposer directement Strapi et pgAdmin, et chiffrer le trafic avec TLS.

D'autres mesures possibles : épingler les versions des images (pas de `latest`), scanner les images, définir des limites de ressources, ne pas exposer pgAdmin.
OK
50b60f145e0c   tp2_backend             bridge    local
2a8b9d2b0006   tp2_default             bridge    local
7968ac66aa3d   tp2_frontend            bridge    local
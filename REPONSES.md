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
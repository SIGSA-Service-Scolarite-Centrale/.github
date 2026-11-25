# Scolarité Centrale — Commandes du projet

Ce document regroupe les principales commandes utilisées pendant le
développement et la maintenance des microservices de la Scolarité Centrale.

## 0. Vue d'ensemble des services

Le projet fonctionne comme un espace de travail de type monorepo : chaque service
est un dossier autonome avec son propre `package.json` et ses propres
`node_modules`. **Lancez toujours les commandes depuis le dossier du service sur
lequel vous travaillez.**

| Dossier                     | Nom dans `package.json`         | Type             | Remarques                                       |
| --------------------------- | ------------------------------- | ---------------- | ----------------------------------------------- |
| `apigateway/`               | `apigateway`                    | NestJS + GraphQL | Passerelle Apollo GraphQL, Apollo Federation    |
| `authentification-service/` | `authentification-service`      | NestJS + REST    | JWT, Passport, mails, stockage objet (MinIO)    |
| `personne-service/`         | `registry-service`              | NestJS + REST    | Dossiers civils et registres académiques        |
| `structure-service/`        | `structure-service`             | NestJS + REST    | Structures / établissements, upload MinIO       |
| `catalogue-service/`        | `curriculum-service`            | NestJS + REST    | Curriculum (CDC v4, §7.4), tests de fumée en JS |
| `seance-service/`           | `seance-service`                | NestJS + REST    | Séances                                         |
| `progression-service/`      | `analytics-progression-service` | NestJS + REST    | Analytics et progression (throttler, helmet)    |
| `rapport-service/`          | `rapport-service`               | NestJS + GraphQL | Sous-graphe Apollo, rapports, MinIO             |
| `notification-service/`     | `notification-service`          | NestJS + REST    | Alertes / notifications, modèles de mails       |
| `frontend/`                 | `frontend`                      | React 19 + Vite  | SPA (Tailwind, TanStack Query, GraphQL request) |

> `apigateway/apollo-inspect/package/package.json` appartient à une copie
> embarquée de `@nestjs/apollo`, utilisée uniquement pour l'inspection — ne
> l'installez pas et ne l'exécutez pas.

Socle technique commun aux services NestJS : Node.js 20+, TypeScript 5.7+,
NestJS 11, TypeORM 0.3, PostgreSQL (`pg`), RabbitMQ (`amqplib` +
`amqp-connection-manager`), Swagger, Jest 30 + ts-jest, ESLint + Prettier.

---

## 1. Prérequis

Assurez-vous que les éléments suivants sont installés :

- Node.js
- npm
- PostgreSQL
- RabbitMQ
- MinIO lorsque le service utilise le stockage objet
- Git

Vérifier les versions installées :

```bash
node -v
npm -v
git --version
```

Vérifier la version de Node.js active avec NVM :

```bash
nvm list
node -v
```

Changer de version de Node.js :

```bash
nvm use 20.20.2
```

---

## 2. Installation du projet

Cloner le dépôt :

```bash
git clone <repository-url>
```

Se placer à la racine du projet :

```bash
cd <project-root>
```

Il **n'existe pas de `package.json` à la racine** : les dépendances s'installent
service par service. Installez celles du service sur lequel vous travaillez :

```bash
cd <service-folder>
npm install
```

Par exemple :

```bash
cd authentification-service
npm install

cd ../apigateway
npm install

cd ../frontend
npm install
```

Si le service possède un fichier de verrouillage et que vous souhaitez une
installation propre et reproductible :

```bash
npm ci
```

---

## 3. Fichiers d'environnement

Chaque service charge sa configuration via `@nestjs/config` + une validation
`joi` : un fichier `.env` doit donc exister **dans le dossier de chaque service**
avant son démarrage. Variables communes aux services :

```text
DB_HOST, DB_PORT, DB_USERNAME, DB_PASSWORD, DB_NAME
RABBITMQ_URI (ou RABBITMQ_URL)
JWT_PUBLIC_KEY / JWT_PRIVATE_KEY (RS256)
MINIO_ENDPOINT, MINIO_PORT, MINIO_ACCESS_KEY, MINIO_SECRET_KEY, MINIO_BUCKET
```

Les clés JWT peuvent être générées pour le développement local :

| Service                    | Commande                |
| -------------------------- | ----------------------- |
| `authentification-service` | `npm run generate:keys` |

Le script `generate:keys` crée la paire de clés RS256 (`JWT_PRIVATE_KEY` /
`JWT_PUBLIC_KEY`) utilisée pour signer et vérifier les jetons. Lancez-le depuis
`authentification-service`, après avoir créé le fichier `.env` :

```bash
cd authentification-service
npm run generate:keys
```

Les règles de journalisation (audit logs) de la base de données s'appliquent avec
un second script, également propre à `authentification-service` :

```bash
cd authentification-service
npm run db:logs
```

`db:logs` exécute `scripts/apply-log-rules.ts`, qui charge le SQL de
`src/database/db-rules/logs.sql` sur la base configurée dans le `.env`. Ce SQL
rend la table `logs` **append-only** : tout `UPDATE` ou `DELETE` est rejeté par
les triggers `trg_logs_prevent_update` et `trg_logs_prevent_delete`. Lancez-le
après `npm run migration:run`, et relancez-le sans risque après une mise à jour
du fichier SQL (les triggers sont recréés à chaque exécution).

Le stockage objet (MinIO) peut être démarré localement depuis les services qui
fournissent un script :

```bash
cd structure-service
npm run start:minio
```

---

## 4. Démarrer l'application

Lancez ces commandes **depuis le dossier du service**.

### Mode développement

```bash
npm run start
```

### Mode développement avec rechargement automatique

```bash
npm run start:dev
```

### Mode débogage

```bash
npm run start:debug
```

### Mode production

```bash
npm run start:prod
```

### Frontend

```bash
cd frontend
npm run dev      # serveur de développement Vite avec HMR
npm run preview  # prévisualisation du build de production
```

> Les scripts `npm run start*` n'existent que dans les services NestJS. Le
> frontend utilise `dev`, `build`, `preview` et `lint` à la place.

---

## 5. Compilation de l'application

Compiler le service (valable pour tous les services NestJS) :

```bash
npm run build
```

L'application compilée est générée dans le dossier du service :

```text
<service-folder>/dist/
```

Exécuter l'application compilée :

```bash
npm run start:prod
```

Compilation du frontend (exécute `tsc -b` puis `vite build`) :

```bash
cd frontend
npm run build
```

---

## 6. Validation TypeScript

Vérifier TypeScript sans générer de fichiers :

```bash
npx tsc --noEmit
```

Cette commande doit être lancée avant de considérer une modification comme
terminée.

Un résultat sans erreur signifie que TypeScript n'a détecté aucun problème de
compilation.

---

## 7. ESLint

Tous les services (y compris `frontend`) exposent exactement les mêmes scripts
`lint` et `lint:fix` :

Vérifier le code :

```bash
npm run lint
```

Corriger automatiquement les erreurs et avertissements détectés :

```bash
npm run lint:fix
```

| Service                    | `lint`                              | `lint:fix`                                |
| -------------------------- | ----------------------------------- | ----------------------------------------- |
| `apigateway`               | `eslint "src/**/*.{js,jsx,ts,tsx}"` | `eslint "src/**/*.{js,jsx,ts,tsx}" --fix` |
| `authentification-service` | `eslint "src/**/*.{js,jsx,ts,tsx}"` | `eslint "src/**/*.{js,jsx,ts,tsx}" --fix` |
| `personne-service`         | `eslint "src/**/*.{js,jsx,ts,tsx}"` | `eslint "src/**/*.{js,jsx,ts,tsx}" --fix` |
| `structure-service`        | `eslint "src/**/*.{js,jsx,ts,tsx}"` | `eslint "src/**/*.{js,jsx,ts,tsx}" --fix` |
| `catalogue-service`        | `eslint "src/**/*.{js,jsx,ts,tsx}"` | `eslint "src/**/*.{js,jsx,ts,tsx}" --fix` |
| `seance-service`           | `eslint "src/**/*.{js,jsx,ts,tsx}"` | `eslint "src/**/*.{js,jsx,ts,tsx}" --fix` |
| `progression-service`      | `eslint "src/**/*.{js,jsx,ts,tsx}"` | `eslint "src/**/*.{js,jsx,ts,tsx}" --fix` |
| `rapport-service`          | `eslint "src/**/*.{js,jsx,ts,tsx}"` | `eslint "src/**/*.{js,jsx,ts,tsx}" --fix` |
| `notification-service`     | `eslint "src/**/*.{js,jsx,ts,tsx}"` | `eslint "src/**/*.{js,jsx,ts,tsx}" --fix` |
| `frontend`                 | `eslint .`                          | `eslint . --fix`                          |

- Les services NestJS utilisent un `eslint.config.mjs` en configuration « flat »
  (ESLint 9/10 + `typescript-eslint` + `eslint-plugin-prettier`).
- `frontend` utilise `eslint.config.js` avec les plugins React Hooks et React
  Refresh ; son périmètre est la racine du projet (`eslint .`).
- `npm run lint` ne modifie jamais les fichiers : utilisez `npm run lint:fix`
  pour appliquer les corrections automatiques.

---

## 8. Formatage (Prettier)

La procédure de formatage est identique dans tous les services, `frontend`
compris (Prettier est déclaré dans chaque `package.json`) :

Vérifier le formatage sans modifier les fichiers :

```bash
npm run format:check
```

Appliquer le formatage :

```bash
npm run format:fix
```

| Groupe de services    | `format:check`                                            | `format:fix`                                              |
| --------------------- | --------------------------------------------------------- | --------------------------------------------------------- |
| Les 9 services NestJS | `prettier --check "src/**/*.{js,jsx,ts,tsx,css,json,md}"` | `prettier --write "src/**/*.{js,jsx,ts,tsx,css,json,md}"` |
| `frontend`            | `prettier --check .`                                      | `prettier --write .`                                      |

- Chaque service NestJS contient un `.prettierrc` (`singleQuote: true`,
  `trailingComma: "all"`) ; Prettier doit donc être lancé depuis le dossier du
  service pour que ce fichier soit pris en compte.
- ESLint est configuré avec `eslint-plugin-prettier` : un écart de formatage est
  remonté comme une erreur ESLint. `npm run lint` et `npm run format:check`
  doivent donc être cohérents.
- `npm run format:check` est la commande à utiliser en CI et avant un commit ;
  `npm run format:fix` réécrit les fichiers.

---

## 9. Validation recommandée avant de commiter

Lancez ces commandes dans l'ordre, depuis le service que vous avez modifié :

```bash
npm run lint
npx tsc --noEmit
npm test
npm run build
```

Minimum recommandé :

```bash
npm run lint
npx tsc --noEmit
```

Ne poursuivez pas une fonctionnalité si TypeScript remonte des erreurs.

> Le frontend n'a pas de script `test` : utilisez-y `npm run lint` et
> `npm run build` (qui exécute déjà `tsc -b`).

---

## 10. Tests

Les scripts `test*` ne sont **pas identiques d'un service à l'autre**. Utilisez
le tableau ci-dessous.

### Disponibilité des scripts

| Script       | `apigateway` | `authentification` | `personne` | `notification` | `progression` | `rapport` | `seance` | `structure` | `catalogue` | `frontend` |
| ------------ | :----------: | :----------------: | :--------: | :------------: | :-----------: | :-------: | :------: | :---------: | :---------: | :--------: |
| `test`       |      ✓       |         ✓          |     ✓      |       ✓        |       ✓       |     ✓     |    ✓     |      ✓      |      ✓      |     —      |
| `test:watch` |      ✓       |         ✓          |     ✓      |       ✓        |       —       |     ✓     |    —     |      ✓      |      —      |     —      |
| `test:cov`   |      ✓       |         ✓          |     ✓      |       ✓        |       —       |     ✓     |    —     |      ✓      |      —      |     —      |
| `test:debug` |      ✓       |         ✓          |     ✓      |       ✓        |       —       |     ✓     |    —     |      ✓      |      —      |     —      |
| `test:e2e`   |      ✓       |         ✓          |     ✓      |       ✓        |       ✓       |     ✓     |    ✓     |      ✓      |      —      |     —      |
| `test:smoke` |      —       |         —          |     —      |       —        |       —       |     —     |    —     |      —      |      ✓      |     —      |

### Lancer les tests unitaires

```bash
npm run test
```

### Lancer les tests en mode watch

```bash
npm run test:watch
```

### Lancer les tests avec couverture

```bash
npm run test:cov
```

La couverture est écrite dans `<service-folder>/coverage/`.

### Lancer les tests end-to-end

```bash
npm run test:e2e
```

### Lancer un fichier de test précis

```bash
npx jest src/example/example.service.spec.ts
```

### Remarques Jest

- Les tests unitaires sont reconnus par le motif `.*\.spec\.ts$` sous `src/`.
- `authentification-service` et `notification-service` remplacent `@nestjs/jwt`
  par un bouchon (stub) ; `authentification-service` charge aussi
  `jest.setup.js`.

---

## 11. Scripts d'environnement et de peuplement

Chaque script appartient à un seul service — lancez-le depuis le dossier de ce
service.

| Service                    | Script                  | Rôle                                                  |
| -------------------------- | ----------------------- | ----------------------------------------------------- |
| `authentification-service` | `npm run seed:run`      | Données initiales (`src/database/seeds/seed.ts`)      |
| `authentification-service` | `npm run db:logs`       | Applique les règles de journalisation                 |
| `authentification-service` | `npm run generate:uml`  | Génère le document UML                                |
| `authentification-service` | `npm run generate:keys` | Génère les clés JWT (script JS)                       |
| `rapport-service`          | `npm run seed:run`      | Données initiales (`src/database/migrations/seed.ts`) |
| `structure-service`        | `npm run start:minio`   | Démarre MinIO via `scripts/start-minio.ps1`           |

---

## 12. Migrations TypeORM

> Utilisez le script npm `typeorm` de chaque service plutôt que la CLI globale :
> il enregistre `ts-node` + `tsconfig-paths` (DataSource en TypeScript).
>
> `apigateway` ne définit **aucun** script TypeORM.

### DataSource

Les 9 services utilisent **le même DataSource**, au même emplacement :

```text
<service-folder>/src/database/data-source.ts
```

Il est enregistré par le script `typeorm` (`ts-node` + `tsconfig-paths`) et
pointe vers `src/database/migrations/*.ts`.

### Créer une migration

```bash
npm run migration:generate -- src/database/migrations/<MigrationName>
```

Le chemin est identique dans tous les services.

### Appliquer les migrations

```bash
npm run migration:run
```

### Annuler la dernière migration

```bash
npm run migration:revert
```

### Afficher l'état des migrations

Disponible uniquement dans `notification-service` et `progression-service` :

```bash
npm run migration:show
```

---

## 13. Workflow de développement recommandé

Pour une nouvelle fonctionnalité, **commencez toujours par vous placer dans le
dossier du service concerné** :

```bash
cd <service-folder>
```

### 1. Mettre à jour le dépôt

```bash
git status
git pull
```

### 2. Créer une branche

```bash
git switch -c feature/<feature-name>
```

### 3. Installer les dépendances si nécessaire

```bash
npm install
```

### 4. Démarrer l'infrastructure

```text
PostgreSQL
RabbitMQ
MinIO
```

### 5. Démarrer le service

```bash
npm run start:dev
```

### 6. Développer et tester

```bash
npm run test
```

### 7. Vérifier TypeScript

```bash
npx tsc --noEmit
```

### 8. Vérifier ESLint

```bash
npm run lint
```

### 9. Compiler

```bash
npm run build
```

### 10. Relire les modifications Git

```bash
git status
git diff
```

### 11. Commiter

```bash
git add .
git commit -m "feat: description"
```

### 12. Pousser

```bash
git push -u origin <branch-name>
```

---

## 14. Validation finale recommandée

Lancez ces commandes dans le service que vous avez modifié, avant de pousser :

```bash
npm run lint
```

puis :

```bash
npx tsc --noEmit
```

puis :

```bash
npm test
```

puis :

```bash
npm run build
```

Si une commande échoue, corrigez le problème avant de passer à la suivante.

Si vous avez touché plusieurs services, répétez la séquence dans **chacun**
d'entre eux (par exemple
`cd apigateway && npm run lint && npx tsc --noEmit && npm test && npm run build`).

---

## 15. Référence rapide des commandes

Lancez chaque commande ci-dessous **depuis le dossier du service** auquel elle
s'applique.

| Tâche                     | Commande                                     | Disponibilité                        |
| ------------------------- | -------------------------------------------- | ------------------------------------ |
| Installer les dépendances | `npm install`                                | tous                                 |
| Installation propre       | `npm ci`                                     | tous (nécessite un lockfile)         |
| Démarrer                  | `npm run start`                              | services NestJS                      |
| Développement             | `npm run start:dev`                          | services NestJS                      |
| Débogage                  | `npm run start:debug`                        | services NestJS                      |
| Production                | `npm run start:prod`                         | services NestJS                      |
| Serveur de dev frontend   | `npm run dev`                                | `frontend`                           |
| Prévisualisation frontend | `npm run preview`                            | `frontend`                           |
| Compiler                  | `npm run build`                              | tous                                 |
| Lint                      | `npm run lint`                               | tous                                 |
| Lint + correction         | `npm run lint:fix`                           | tous                                 |
| Vérification TypeScript   | `npx tsc --noEmit`                           | tous                                 |
| Tests                     | `npm run test`                               | services NestJS                      |
| Tests (watch)             | `npm run test:watch`                         | voir le tableau du §10               |
| Couverture de tests       | `npm run test:cov`                           | voir le tableau du §10               |
| Débogage des tests        | `npm run test:debug`                         | voir le tableau du §10               |
| Tests E2E                 | `npm run test:e2e`                           | voir le tableau du §10               |
| Test de fumée             | `npm run test:smoke`                         | `catalogue-service`                  |
| Serveur registre simulé   | `npm run mock:registry`                      | `catalogue-service`                  |
| Vérifier le formatage     | `npm run format:check`                       | tous                                 |
| Appliquer le formatage    | `npm run format:fix`                         | tous                                 |
| Lancer les seeds          | `npm run seed:run`                           | `authentification`, `rapport`        |
| Générer les clés JWT      | `npm run generate:keys`                      | `authentification-service`           |
| Règles de journalisation  | `npm run db:logs`                            | `authentification-service`           |
| Générer les clés de dev   | `npm run keys:dev`                           | `personne`, `seance`, `notification` |
| Démarrer MinIO            | `npm run start:minio`                        | `structure-service`                  |
| Générer une migration     | `npm run migration:generate -- <Path/Name>`  | tous sauf `apigateway`               |
| Appliquer les migrations  | `npm run migration:run`                      | tous sauf `apigateway`               |
| Annuler une migration     | `npm run migration:revert`                   | tous sauf `apigateway`               |
| État des migrations       | `npm run migration:show`                     | `notification`, `progression`        |
| CLI Nest                  | `npx @nestjs/cli`                            | services NestJS                      |
| Générer un module         | `npx nest g module <name>`                   | services NestJS                      |
| Générer un contrôleur     | `npx nest g controller <name>`               | services NestJS                      |
| Générer un service        | `npx nest g service <name>`                  | services NestJS                      |
| État Git                  | `git status`                                 | racine du dépôt                      |
| Historique Git            | `git log --oneline`                          | racine du dépôt                      |
| Branches Git              | `git branch -a`                              | racine du dépôt                      |
| Récupération Git          | `git fetch origin`                           | racine du dépôt                      |
| Mise à jour Git           | `git pull`                                   | racine du dépôt                      |
| Remisage Git              | `git stash`                                  | racine du dépôt                      |
| Différences Git           | `git diff`                                   | racine du dépôt                      |
| Commit Git                | `git commit -m "message"`                    | racine du dépôt                      |
| Push Git                  | `git push`                                   | racine du dépôt                      |
| PostgreSQL                | `psql -U <user> -d <database>`               | –                                    |
| CLI TypeORM (directe)     | `npx typeorm migration:run -d <data-source>` | uniquement si `ts-node` est résolu   |

---

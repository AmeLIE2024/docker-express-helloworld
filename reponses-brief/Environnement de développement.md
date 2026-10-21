---
tags:
  - Backend
  - Docker
created: 2026-10-20
---
## Qu'arrive-t-il aux fichiers écrits dans un conteneur lorsqu'on l'arrête, puis lorsqu'on le supprime et le recrée ?

Quand on arrête un conteneur Docker, les fichiers ne sont pas perdus mais restent stockés dans sa couche inscriptible.
Si on redémarre le conteneur, on retrouve exactement le même état et avec les mêmes fichiers. 

Quand on supprime et recrée un conteneur les fichiers sont perdus sauf s'ils sont dans un volume ou un bind mount.
La suppression du conteneur détruit sa couche inscriptible. Lorsqu'on recrée un nouveau conteneur à partir de la même image, Docker crée une toute nouvelle couche d'écriture vide. Le nouveau conteneur repart de 0, à l'état initial.

### Liste des commandes

| Action                         | Docker                         | Docker Compose           | Effet sur les fichiers                                                                                                                                      |
| ------------------------------ | ------------------------------ | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Arrêter                        | `docker stop <conteneur>`      | `docker compose stop`    | Conservés (couche inscriptible intacte)                                                                                                                     |
| Redémarrer                     | `docker start <conteneur>`     | `docker compose start`   | Retrouvés à l'identique                                                                                                                                     |
| Supprimer un conteneur arrêté  | `docker rm <conteneur>`        | `docker compose rm`      | Perdus (couche inscriptible détruite)                                                                                                                       |
| Créer et démarrer              | `docker run <image>`           | `docker compose up -d`   | `docker run` : nouveau conteneur, état initial de l’image. `docker compose up -d` : crée les conteneurs absents (état initial), réutilise ceux qui existent |
| Arrêter et supprimer           | `docker stop` puis `docker rm` | `docker compose down`    | Perdus, sauf volumes et bind mounts                                                                                                                         |


> [!info] Sources
>  - https://www.stephane-robert.info/docs/conteneurs/moteurs-conteneurs/docker/volumes/
> - https://www.ssdnodes.com/learn/lang/fr/docker-compose-down-vs-stop
> - https://www.theodo.com/blog/docker-compose-stop-vs-down

## Qu'est-ce qu'un volume Docker et comment permet-il de conserver des données ?

Un volume Docker est un répertoire ou un espace de stockage dédié qui peut être partagé entre les conteneurs et persiste même lorsque le conteneur associé est arrêté ou supprimé. 
En d'autres termes, il offre un mécanisme de persistance des données indépendant du cycle de vie des conteneurs. 

Un volume standards (géré par Docker) est un espace de stockage dédié, crée et géré par Docker dans un répertoire spécial ( `/var/lib/docker/volumes/`). 

> [!INFO] Sources
> - https://blog.alphorm.com/volumes-docker-guide-complet#questce-quun-volume-docker
> - https://www.stephane-robert.info/docs/conteneurs/moteurs-conteneurs/docker/volumes/

## Qu'est-ce qu'un bind mount, quelle est la différence avec un volume et dans quels cas utiliser chacun ?

Le Bind Mount crée un lien direct entre un dossier de la machine et un dossier dans le conteneur. Contrairement aux volumes standards, le bind mount pointe vers n'importe quel chemin de notre système de fichiers. Le conteneur voit et modifie directement nos fichiers, sans l'intermédiaire des volumes standards.

**Exemple de mise en place :**

```bash
# Syntaxe -v (chemin absolu recommandé)
docker run -v /home/user/projet:/app nginx
# Syntaxe --mount (plus explicite)
docker run --mount type=bind,source=/home/user/projet,target=/app nginx
# Astuce : utiliser $(pwd) pour le répertoire courant
docker run -v $(pwd):/app nginx
```

Le dossier se trouve sur l’hôte et n’est pas géré par Docker. Le bind mount est idéal en développement.

**Quand utiliser les volumes standards ?**

- Bases de données : PostgreSQL, MySQL, MongoDB stockent leurs données dans des volumes
- Données applicatives : uploads utilisateurs, fichiers générés
- Partage entre conteneurs : plusieurs conteneurs accédant aux mêmes données
- Production : privilégier les volumes pour les données

**Quand utiliser les bind mount ?**
- en développement
- pour injecter des fichiers de configuration depuis l'hôte

**Tableau comparatif :**

| Critère         | Volume Standard            | Bind Mount                 |
| --------------- | -------------------------- | -------------------------- |
| Stockage        | `/var/lib/docker/volumes/` | Chemin hôte quelconque     |
| Persistance     | ✅ Oui                      | ✅ Oui                      |
| Géré par Docker | ✅ Oui                      | ❌ Non                      |
| Performance     | Bonne                      | Dépend du disque de l'hôte |
| Sécurité        | ✅ Isolé                    | ⚠️ Expose l'hôte           |
| Portabilité     | ✅ Facile à migrer          | ❌ Dépend de l'hôte         |
| Cas d'usage     | Production, BDD            | Développement, hot-reload  |

> [!INFO] Sources
>- https://www.geeksforgeeks.org/devops/docker-volume-vs-bind-mount/
> - https://www.stephane-robert.info/docs/conteneurs/moteurs-conteneurs/docker/volumes/#o%C3%B9-sont-stock%C3%A9es-vos-donn%C3%A9es-
> - https://docs.docker.com/engine/storage/bind-mounts/

## Exercice : persistance avec un volume
https://docs.docker.com/get-started/docker-concepts/running-containers/persisting-container-data/

```Powershell

PS C:\Users\97692T\WSSimplon\persiting-container-docker> docker run --name=db -e POSTGRES_PASSWORD=secret -d -v postgres_data:/var/lib/postgresql postgres:18
aa2dfc895d080d7439c1a97be546ce007db2b68ffc5f3fd51958bc68805fc9be

PS C:\Users\97692T\WSSimplon\persiting-container-docker> docker exec -ti db psql -U postgres
psql (18.6 (Debian 18.6-1.pgdg13+2))
Type "help" for help.

postgres=# CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    description VARCHAR(100)
);
INSERT INTO tasks (description) VALUES ('Finish work'), ('Have fun');
CREATE TABLE
INSERT 0 2
postgres=# SELECT * FROM tasks;
 id | description
----+-------------
  1 | Finish work
  2 | Have fun
(2 rows)

postgres=# \q

PS C:\Users\97692T\WSSimplon\persiting-container-docker> docker stop db
db
PS C:\Users\97692T\WSSimplon\persiting-container-docker> docker rm db
db

PS C:\Users\97692T\WSSimplon\persiting-container-docker> docker run --name=new-db -d -v postgres_data:/var/lib/postgresql postgres:18

PS C:\Users\97692T\WSSimplon\persiting-container-docker> docker exec -ti new-db psql -U postgres -c "SELECT * FROM tasks"
 id | description
----+-------------
  1 | Finish work
  2 | Have fun
(2 rows)

PS C:\Users\97692T\WSSimplon\persiting-container-docker> docker rm -f new-db
new-db
PS C:\Users\97692T\WSSimplon\persiting-container-docker> docker volume rm postgres_data
postgres_data
PS C:\Users\97692T\WSSimplon\persiting-container-docker>

```

La table est retrouvée dans `new-db` alors que `db` a été supprimé.

Les données sont bien dans le volume `postgres_data`
## Notes supplémentaires

### Comment rendre un dossier de ton environnement de travail accessible dans un conteneur

On peut utiliser `--mount` ou `--volume`
Quand la source n'existe pas : avec `-v`, Docker crée le dossier sans prévenir, avec `--mount` il renvoie une erreur.
`--mount` est préféré en général, car plus explicite et plus complet. 

```powershell
docker run --mount type=bind,src=<host-path>,dst=<container-path>
docker run --volume <host-path>:<container-path>
```

### Comment choisir le chemin source et le chemin cible du montage
- **Source (`source` ou `src`)** : l’emplacement du fichier ou dossier sur l’hôte, en chemin absolu ou relatif. C’est le dossier du projet qu'on veut partager. 
- **Cible (`target`, `dst` ou `destination`)** : l’endroit où il est monté dans le conteneur, obligatoirement un chemin absolu. Ce choix est surtout pratique : prendre le dossier où l’application s’attend à trouver ses fichiers (par exemple `/app`). 

> [!WARNING] Risques de sécurité des bind mounts
> Un conteneur avec un **bind mount** vers `/` (racine) aurait accès à **tout le système de fichiers** de l'hôte ! Limitez toujours les bind mounts aux répertoires **strictement nécessaires** et évitez-les en **production** pour les données. Utiliser `:ro` quand le conteneur n'a pas besoin d'écrire.

### Pourquoi les fichiers déjà présents dans le dossier cible de l'image sont masqués par le montage
Quand on monte un dossier sur un répertoire du conteneur qui contient déjà des fichiers, ces fichiers sont masqués par le montage.
Les fichiers de l’image ne sont donc ni supprimés ni modifiés : le conteneur voit à cet endroit le contenu du dossier de l’hôte à la place. Il n’existe pas de moyen simple de retirer un montage pour les révéler à nouveau : le plus simple est de recréer le conteneur sans le montage.
### Pourquoi un bind mount ne redémarre pas automatiquement une application lorsqu'un fichier change
Un bind mount ne fait que **partager des fichiers** : le conteneur voit les modifications de l’hôte en direct. Pour avoir un rechargement automatique, il faut un outil qui surveille les fichiers et relance ou recharge l'application, par exemple le mode dev de Quarkus, SpringBootDevTools ou, nodemon pour Node.js. Docker Compose a aussi une fonctionnalité « Compose Watch ».


## Commande de l'exercice de mise en application

```powershell
docker run --rm -it -p 3000:3000 -v "${PWD}:/app" -w /app node:24-alpine sh -c "npm install && npm run dev"                               
```

### Explications : 
#### Les options
`--rm` supprime automatiquement le conteneur quand il s'arrête. 
`-it` pour voir les logs en direct et pouvoir arrêter avec `ctrl+c` 
`-p 3000:3000` publie un port, au format `port-hôte:port-conteneur`
`-v "${PWD}:/app"` crée le bind mount, au format `chemin-hôte:chemin-conteneur` :
- `${PWD}` est une variable PowerShell qui vaut le dossier courant, donc mon dossier de projet. 
- `/app` est le dossier du conteneur où mon projet apparaît ;
- les guillemets protègent le chemin s’il contient des espaces.
`-w /app` (working directory) définit le dossier de travail dans le conteneur. 

#### L’image
`node:24-alpine` se lit `nom:tag`. L’image est Node.js, et le tag `24-alpine` signifie Node 24 sur Alpine Linux.

#### La commande exécutée dans le conteneur
**`sh -c "npm install && npm run dev"`** :

- `sh` est le shell d’Alpine (il n’a pas `bash` par défaut). Nécessaire car il y a deux commandes enchainées;
- `-c` pour lui demander d’exécuter la chaîne qui suit ;
- `npm install` installe les dépendances listées dans `../package.json` (Express, nodemon) ;
- `&&` n’exécute la commande suivante que si la précédente a réussi ;
- `npm run dev` lance le script `dev` du `../package.json`, soit `nodemon --legacy-watch --watch src src/index.js`

> [!INFO] Info
> `--legacy-watch` permet de détecter les changements de fichiers faits depuis l'hôte dans le conteneur.
### Réponse 

```powershell
[nodemon] starting `node src/index.js`  
Example app listening on port 3000  

# modification du fichier index.js

[nodemon] restarting due to changes...  
[nodemon] starting `node src/index.js`  
Example app listening on port 3000

# prise en compte des modifications apportées sur le localhost:3000 sans reconstruction de l'image ni recréation du conteneur.
```


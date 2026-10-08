# Examen CinéK8s — COSSAIS Simon

## Partie 1
**Q1.1**
- Propriété Spring : `movie.url` (injectée via `@Value("${movie.url}")` dans `MovieClient`).
- Variable d'environnement : `MOVIE_URL` (grâce au *relaxed binding* de Spring Boot qui transforme les points en underscores et met en majuscules).

**Q1.2**
- (a) Le film demandé n'existe pas : **422 Unprocessable Entity** (`ResponseStatusException(HttpStatus.UNPROCESSABLE_ENTITY, ...)`).
- (b) Il reste moins de places que demandé : **409 Conflict** (`ResponseStatusException(HttpStatus.CONFLICT, ...)`).
- (c) `movie-service` ne répond pas du tout : **503 Service Unavailable** (`ResponseStatusException(HttpStatus.SERVICE_UNAVAILABLE, ...)` interceptant `ResourceAccessException`).

**Q1.3**
Ligne complétée dans `ticket-service/src/main/resources/application.yaml` :
```yaml
          include: readinessState,movie
```
*Explication* : En cas d'indisponibilité de `movie-service`, si cette dépendance était dans la `livenessProbe`, le kubelet redémarrerait en boucle le conteneur du Pod `ticket` (CrashLoop), ce qui ne réglerait pas la panne externe et consommerait des ressources inutilement. À l'inverse, l'échec d'une `readinessProbe` retire simplement le Pod des endpoints du Service Kubernetes (aucun trafic ne lui est routé tant que `movie-service` est indisponible) sans tuer ni redémarrer le conteneur.

**Q1.4**
| Endpoint | Probe(s) Kubernetes qui l'utilisent | Conséquence d'un échec de la probe |
|----------|-------------------------------------|-----------------------------------|
| `/actuator/health/liveness` | `livenessProbe` (et `startupProbe` au démarrage) | Le conteneur est redémarré (Restart) par le kubelet. |
| `/actuator/health/readiness` | `readinessProbe` | Le Pod passe à `0/1 Ready` et est exclu des Endpoints du Service Kubernetes (il ne reçoit plus de requêtes). |

*Rôle de `server.shutdown: graceful`* : Lors d'un rolling update, cela permet au serveur Tomcat d'attendre la fin du traitement des requêtes HTTP en cours avant d'arrêter le processus, évitant ainsi d'interrompre brutalement des transactions client pendant la mise à jour des Pods.


## Partie 2
Sortie de `curl -s localhost:8080/api/movies | jq '.[].title'`
```
"Pod Fiction"
"Le Seigneur des Pods"
"Docker Wars"
"Rollback to the Future"
```

Sortie de `curl -s localhost:8080/api/movies/whoami`
```
{"hostname":"laptop-lws","environment":"local"}
```

Sortie de `curl -s -X POST localhost:8082/api/tickets -H 'Content-Type: application/json' -d '{"movieId":2,"seats":3}' | jq`
```
{
  "id": 1,
  "movieId": 2,
  "movieTitle": "Le Seigneur des Pods",
  "seats": 3,
  "total": 36.00,
  "createdAt": "2026-10-08T09:16:08.965472806Z"
}
```

Sortie de `curl -s localhost:8082/actuator/health/readiness | jq` :
```json
{
  "status": "UP",
  "components": {
    "movie": {
      "status": "UP"
    },
    "readinessState": {
      "status": "UP"
    }
  }
}
```

### 2.2 — Couper movie-service
Sortie de `curl -s localhost:8082/actuator/health/readiness | jq` (après arrêt de `movie-service`) :
```json
{
  "status": "DOWN",
  "components": {
    "movie": {
      "status": "DOWN",
      "details": {
        "error": "I/O error on GET request for \"http://localhost:8080/actuator/health/liveness\": null"
      }
    },
    "readinessState": {
      "status": "UP"
    }
  }
}
```

Sortie de `curl -s localhost:8082/actuator/health/liveness | jq .status` :
```
"UP"
```

Sortie de `curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8082/api/tickets -H 'Content-Type: application/json' -d '{"movieId":2,"seats":3}'` :
```
503
```

### 2.3 — Questions
**Q2.1**
On lance `ticket-service` avec `SERVER_PORT=8082` pour ne pas modifier le fichier `application.yaml` qui contient la configuration cible standard (port 8080 en conteneur / Kubernetes). Cela permet d'adapter le port temporairement pour l'environnement local où deux processus ne peuvent pas écouter sur le même port de la machine hôte.
Le mécanisme Spring Boot qui rend cela possible est l'**externalisation de la configuration** (*Externalized Configuration*) combinée au **relaxed binding** : les variables d'environnement ont une priorité plus élevée que les fichiers de configuration YAML et Spring fait automatiquement correspondre `SERVER_PORT` à `server.port`.

**Q2.2**
C'est le comportement attendu car :
- La **liveness** vérifie la santé intrinsèque du processus JVM de `ticket-service`. Le processus fonctionne normalement, il n'est pas bloqué (pas de deadlock ni de fuite mémoire critique), donc un redémarrage serait inutile voire néfaste.
- La **readiness** vérifie la capacité du service à accomplir son travail métier. Comme `movie-service` est indisponible, `ticket-service` ne peut plus valider ni créer de réservations. Il se déclare donc `DOWN` pour indiquer aux routeurs / à Kubernetes de ne pas lui acheminer de trafic client tant que sa dépendance n'est pas rétablie.



## Partie 3
**Q3.1**
On copie `pom.xml` en premier pour profiter du cache Docker. Les dépendances changent rarement, donc l'étape `dependency:go-offline` reste en cache. Si on modifie seulement une ligne de code Java dans `src/`, Docker réutilise ce cache et ne re-télécharge pas toutes les librairies, ce qui fait gagner plusieurs minutes par build.

**Q3.2**
`-Xmx512m` impose une valeur fixe en dur. Si on change la limite mémoire du conteneur (par exemple à 256Mo ou 1Go), la JVM ne s'adapte pas et risque de se faire tuer par l'OS (`OOMKilled`). Avec `-XX:MaxRAMPercentage=75`, la JVM s'adapte automatiquement à la mémoire allouée au conteneur (cgroups) en prenant 75% pour le tas (heap) et en laissant 25% pour le reste (threads, métadonnées, OS).

**Q3.3**
Dans Kubernetes, les Pods démarrent de façon indépendante. Si `ticket` démarre avant `movie`, son conteneur tourne mais sa `readinessProbe` échoue (car elle n'arrive pas à joindre `movie`). Le Pod `ticket` reste simplement en `0/1 NotReady` et le Service ne lui envoie aucun trafic client tant que `movie` n'est pas opérationnel. Dès que `movie` est prêt, la probe de `ticket` passe au vert et le trafic arrive, sans redémarrer le conteneur.
## Partie 4
### 4.4 — Sorties des commandes
`kubectl get pods` :
```
NAME                      READY   STATUS    RESTARTS   AGE
movie-59684459f4-6rwd7    1/1     Running   0          50s
movie-59684459f4-glxd7    1/1     Running   0          50s
ticket-66d95c98b6-8fkcl   1/1     Running   0          50s
ticket-66d95c98b6-jqp46   1/1     Running   0          50s
```

`kubectl get endpoints movie ticket` :
```
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME     ENDPOINTS                           AGE
movie    10.244.0.36:8080,10.244.0.38:8080   70s
ticket   10.244.0.37:8080,10.244.0.39:8080   70s
```

Réservation créée (via port-forward) :
```json
{
  "id": 1,
  "movieId": 2,
  "movieTitle": "Le Seigneur des Pods",
  "seats": 2,
  "total": 24.00,
  "createdAt": "2026-10-08T10:43:57.098362903Z"
}
```

### 4.5 — Questions
**Q4.1**
`kubectl apply -f k8s/` lit et applique les fichiers dans l'ordre alphabétique. Les préfixes `00-`, `10-`, `20-` permettent de maîtriser l'ordre de création des ressources : on crée d'abord le Namespace (`00-`), ensuite les ConfigMaps (`10-`), puis les Deployments et Services (`20-`, `30-`) qui en ont besoin.

**Q4.2**
C'est la `startupProbe` qui est en train de tourner. Ce n'est pas une anomalie : Spring Boot a besoin de 15 à 30 secondes pour initialiser son contexte et démarrer Tomcat. Tant que la startupProbe n'a pas validé le démarrage, le Pod reste en `0/1` et la livenessProbe est mise en pause pour éviter de redémarrer le conteneur trop tôt.

**Q4.3**
Les Pods passeraient en erreur `ErrImagePull` / `ImagePullBackOff`. Avec `Always`, Kubernetes tente systématiquement de télécharger l'image depuis un registre distant (Docker Hub). Comme nos images ont été créées localement et chargées dans Minikube sans être poussées sur un registre distant, le téléchargement échouerait.
## Partie 5
### 5.3 — Tests de l'Ingress
Boucle whoami (load-balancing) :
```
movie-59684459f4-glxd7
movie-59684459f4-6rwd7
movie-59684459f4-glxd7
movie-59684459f4-glxd7
movie-59684459f4-6rwd7
movie-59684459f4-glxd7
```

Code HTTP pour `/actuator/health` :
```
404
```

### 5.4 — Questions
**Q5.1**
Les 2 Pods `movie` distincts répondent en alternance. C'est l'objet **Service** de Kubernetes (via kube-proxy et ses règles de routage iptables/IPVS) qui répartit la charge entre les différents Pods enregistrés dans ses Endpoints.

**Q5.2**
La requête renverrait une erreur **404 Not Found**. Avec `pathType: Exact`, l'Ingress ne transmet au Service que les requêtes dont l'URL est strictement `/api/movies`. Les sous-chemins comme `/api/movies/1` ne matcheraient plus la règle.

**Q5.3**
On obtient un code **404 Not Found**. C'est **tout à fait souhaitable** pour la sécurité : les endpoints Actuator sont purement techniques et internes (santé, métriques, éventuellement variables d'environnement). Ils n'ont pas vocation à être exposés publiquement à l'extérieur du cluster.
## Partie 6
### 6.1 — Le service movie disparaît
#### Prédictions avant test :
- (a) `READY` et `RESTARTS` des Pods `ticket` après 30 s : `0/1` et `0` restart.
- (b) Contenu de `kubectl get endpoints ticket` : aucun endpoint (`<none>`).
- (c) Code HTTP de `GET http://cinema.local/api/tickets` : `503 Service Temporarily Unavailable` (généré par l'Ingress).
- (d) Statut de la liveness de `ticket` : `UP`.

#### Observations :
`kubectl get pods` :
```
NAME                      READY   STATUS    RESTARTS   AGE
ticket-66d95c98b6-8fkcl   0/1     Running   0          25m
ticket-66d95c98b6-jqp46   0/1     Running   0          25m
```

`kubectl get endpoints ticket` :
```
NAME     ENDPOINTS   AGE
ticket               25m
```

`curl -si http://cinema.local/api/tickets | head -1` :
```
HTTP/1.1 503 Service Temporarily Unavailable
```

#### Explication Q6.1 :
**En 4 étapes :**
1. Le scale à 0 supprime tous les Pods `movie` et vide les endpoints du Service `movie`.
2. La `readinessProbe` de chaque Pod `ticket` (exécutée toutes les 5 s via `MovieHealthIndicator`) tente de joindre `movie-service` et échoue.
3. Après 3 échecs consécutifs, le kubelet marque les Pods `ticket` comme non prêts (`0/1 NotReady`), et le contrôleur Kubernetes retire leurs adresses IP des Endpoints du Service `ticket`.
4. Lorsque l'Ingress Nginx reçoit une requête pour `/api/tickets`, il constate que le Service `ticket` n'a aucun backend sain disponible et renvoie une erreur HTTP `503`.

### 6.2 — Mission dépannage
| # | Statut observé | Commande de diagnostic | Cause exacte | Correction apportée |
|---|----------------|------------------------|--------------|---------------------|
| 1 | `ImagePullBackOff` / `ErrImagePull` | `kubectl describe pod -l app=ticket-debug` (section Events) | `imagePullPolicy: Always` force le téléchargement depuis Docker Hub où l'image n'est pas publiée | Remplacé par `imagePullPolicy: IfNotPresent` |
| 2 | `CreateContainerConfigError` | `kubectl describe pod -l app=ticket-debug` (section Events) | La ConfigMap référencée `ticket-configmap` n'existe pas dans le cluster | Corrigé par le nom réel `ticket-config` |
| 3 | `0/1 Running` (Readiness probe failed) | `kubectl describe pod -l app=ticket-debug` (section Events) | La `readinessProbe` teste le port 8081 alors que le service écoute sur 8080 | Remplacé `port: 8081` par `port: 8080` (ou `port: http`) |

### 6.3 — Changer la configuration sans rebuild
**Q6.3**
- *Pourquoi la modification n'a-t-elle pas été prise en compte immédiatement ?*
  Dans Kubernetes, les variables d'environnement injectées depuis une ConfigMap ne sont transmises au conteneur qu'à son démarrage. Modifier la ConfigMap ne modifie pas l'environnement des conteneurs déjà actifs en mémoire.
- *Qu'est-ce qui l'a rendue effective ?*
  La commande `kubectl rollout restart deploy/movie` a déclenché le remplacement progressif des Pods (rolling update). Les nouveaux Pods démarrés ont ainsi lu la nouvelle version de la ConfigMap au lancement.
## Partie 7
**Q7.1**
1. **Résolution DNS** : CoreDNS (le DNS interne du cluster) résout le nom court `movie` vers la ClusterIP virtuelle du Service `movie`.
2. **Routage et Load-balancing** : Les règles iptables/IPVS gérées par `kube-proxy` interceptent le trafic vers cette ClusterIP et sélectionnent aléatoirement l'un des Pods `movie` sains enregistrés dans les Endpoints.
3. **Réception** : La requête est acheminée jusqu'au conteneur du Pod `movie` désigné, qui la traite sur son port 8080.

**Q7.2**
- *Variation du nombre* : Les tickets sont stockés dans une liste en mémoire vive propre à chaque instance Java de Pod `ticket`. Avec 2 réplicas et le load-balancing, chaque requête interroge un Pod différent qui ne possède qu'une partie des réservations.
- *Suppression des Pods* : Toutes les réservations sont perdues car les conteneurs sont sans persistance et éphémères.
- *Solution architecturale* : Rendre le microservice réellement stateless en déportant la persistance des données dans une base de données partagée (ex: PostgreSQL) adossée à du stockage persistant (PersistentVolume).

**Q7.3**
- *Constat* : Dès la suppression, un nouveau Pod `movie` est instantanément créé pour le remplacer.
- *Perte sans Deployment* : Un Pod « nu » n'a aucun contrôleur de gestion d'état. S'il est supprimé ou s'il plante, il disparaît définitivement. Le `Deployment` garantit l'auto-guérison (self-healing), le maintien du nombre de réplicas et les mises à jour sans interruption de service.
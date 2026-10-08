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
…
## Partie 4
…
## Partie 5
…
## Partie 6
(tableau de dépannage, prédictions, explications)
## Partie 7
…
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
```
(collez ici les sorties de commandes demandées)
```
**Q2.1** …

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
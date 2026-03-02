# Etape 2 : 
# Notes d'Architecture — Lab 85

## Diagramme de l'architecture actuelle

![Diagramme d'architecture](./architecture.png)

## Analyse de l'architecture

### Où se passe l'isolation ?
L'isolation se produit au niveau des containers à l'intérieur du Pod.
Chaque container (quote-app et postgres) tourne dans son propre espace
de processus isolé, avec son propre système de fichiers et son interface
réseau. Le Pod est lui-même isolé des autres Pods via le réseau Kubernetes.

### Qu'est-ce qui redémarre automatiquement ?
Kubernetes redémarre automatiquement les containers qui plantent via le
kubelet. Le contrôleur ReplicaSet s'assure que le nombre de Pods souhaité
tourne en permanence. Si un Pod meurt, le ReplicaSet en recrée un nouveau.

### Qu'est-ce que Kubernetes ne gère PAS ?
- La machine physique ou virtuelle qui héberge le nœud
- L'infrastructure réseau externe au cluster
- Les données réelles dans le PersistentVolume (pas de backup automatique)
- La configuration DNS en dehors du cluster
- Le système d'exploitation du nœud

# Etape 3 : 

## Comparaison Containers vs Machines Virtuelles

| Critère               | Container                              | Machine Virtuelle (VM)                  |
|-----------------------|----------------------------------------|-----------------------------------------|
| Partage du kernel     | Partage le kernel de l'hôte            | Kernel propre à chaque VM               |
| Temps de démarrage    | Très rapide (secondes)                 | Lent (minutes)                          |
| Overhead ressources   | Léger (pas d'OS complet)               | Lourd (OS complet par VM)               |
| Isolation sécurité    | Isolation partielle (kernel partagé)   | Isolation forte (hyperviseur)           |
| Complexité opération  | Simple à orchestrer (Kubernetes)       | Plus complexe à gérer                   |

## Quand préférer une VM à un container ?
- Quand on a besoin d'une isolation forte (ex: clients différents sur le même serveur)
- Quand l'application nécessite un OS différent (ex: Windows sur hôte Linux)
- Pour des workloads legacy qui ne sont pas containerisables
- Dans des contextes de sécurité stricts (banques, défense)

## Quand combiner les deux ?
- Les nœuds Kubernetes tournent souvent sur des VMs dans le cloud
- On utilise des VMs pour isoler les clusters entre eux
- Les containers tournent à l'intérieur des VMs pour la flexibilité
- Ex: AWS EC2 (VM) qui héberge des pods Kubernetes (containers)

# Etape 4 : 

## Scaling horizontal

### Commande utilisée
`kubectl scale deployment quote-app --replicas=3 -n quote-lab`

### Qu'est-ce qui change quand on scale ?
- Le nombre de Pods passe de 1 à 3
- Le trafic est réparti entre les 3 pods via le Service (load balancing)
- La capacité de traitement des requêtes augmente
- La résilience augmente : si un pod tombe, les 2 autres continuent

### Qu'est-ce qui ne change pas ?
- Le Service reste le même (même IP, même port)
- La base de données reste unique (partagée par les 3 pods)
- La configuration de l'application ne change pas
- L'URL d'accès reste identique pour l'utilisateur

# Etape 5 : 

## Simulation de panne

### Qui a recréé le pod ?
Le contrôleur **ReplicaSet** de Kubernetes a recréé le pod automatiquement.
Il surveille en permanence que le nombre de pods correspond au nombre
souhaité (replicas: 3). Dès qu'un pod disparaît, il en recrée un nouveau.

### Pourquoi ?
Car le Deployment définit un état désiré (3 replicas). Le ReplicaSet
est responsable de maintenir cet état en permanence. C'est le principe
de la "réconciliation" dans Kubernetes.

### Que se passerait-il si le nœud lui-même tombait ?
Si le nœud (la machine) tombe, Kubernetes détecterait le nœud comme
"NotReady" après un délai (~5 minutes). Les pods seraient alors
replanifiés sur un autre nœud disponible. Dans notre cas avec un seul
nœud, l'application serait indisponible jusqu'au retour du nœud.
C'est pourquoi une architecture de production nécessite plusieurs nœuds.



# Etape 6 : 

## Resource Limits

### Requests vs Limits

**Requests** : c'est la quantité de ressources **garantie** au container.
Kubernetes utilise cette valeur pour décider sur quel nœud placer le pod.
Si un nœud n'a pas assez de ressources disponibles, le pod ne sera pas
planifié dessus.

**Limits** : c'est le **maximum** que le container peut consommer.
Si le container dépasse la limite CPU, il est ralenti (throttling).
S'il dépasse la limite mémoire, il est tué (OOMKilled) et redémarré.

### Pourquoi c'est important dans un système multi-tenant ?
Dans un cluster partagé entre plusieurs équipes ou applications :
- Sans limits, un container peut consommer toutes les ressources du nœud
  et faire tomber les autres applications
- Les requests garantissent que chaque app a les ressources minimales
  dont elle a besoin
- Les limits protègent le cluster contre les fuites mémoire ou les
  boucles infinies qui consommeraient tout le CPU

  # Etape 7 : 

  ## Readiness et Liveness Probes

### Différence entre Readiness et Liveness

**Liveness probe** : vérifie si le container est **vivant**.
Si elle échoue, Kubernetes tue et redémarre le container.
Utile pour détecter un container bloqué/en deadlock.

**Readiness probe** : vérifie si le container est **prêt à recevoir du trafic**.
Si elle échoue, Kubernetes retire le pod du Service (plus de trafic envoyé)
mais ne le redémarre PAS. Utile pendant le démarrage ou si l'app est
temporairement surchargée.

### Pourquoi c'est important en production ?
- Sans liveness probe : un container bloqué reste en place sans être redémarré
- Sans readiness probe : du trafic est envoyé à un pod pas encore prêt,
  causant des erreurs pour les utilisateurs
- Ensemble, elles garantissent que seuls les pods sains et prêts
  reçoivent du trafic

  # Etape 8 : 

  ## Kubernetes et la virtualisation

### Qu'est-ce qui tourne sous notre cluster k3s ?
k3s tourne directement sur une machine virtuelle Ubuntu (VirtualBox/VMware).
Le cluster Kubernetes ne sait pas qu'il est dans une VM, il voit juste
un système Linux avec des ressources disponibles.

### Kubernetes remplace-t-il la virtualisation ?
Non. Kubernetes et la virtualisation sont complémentaires :
- Les VMs fournissent l'isolation au niveau matériel (hyperviseur)
- Kubernetes orchestre les containers à l'intérieur de ces VMs
- Dans le cloud, les nœuds Kubernetes sont des VMs (ex: EC2 sur AWS)

### Comment cette stack se présente dans différents contextes ?

**Cloud (ex: AWS, Azure, GCP) :**
- Hyperviseur → VMs (EC2/instances) → k8s nodes → Pods → Containers
- Le cloud provider gère les VMs, l'équipe DevOps gère Kubernetes

**Système

# Etape 9 : 

## Kubernetes et la virtualisation

### Qu'est-ce qui tourne sous notre cluster k3s ?
k3s tourne directement sur une machine virtuelle Ubuntu (VirtualBox/VMware).
Le cluster Kubernetes ne sait pas qu'il est dans une VM, il voit juste
un système Linux avec des ressources disponibles.

### Kubernetes remplace-t-il la virtualisation ?
Non. Kubernetes et la virtualisation sont complémentaires :
- Les VMs fournissent l'isolation au niveau matériel (hyperviseur)
- Kubernetes orchestre les containers à l'intérieur de ces VMs
- Dans le cloud, les nœuds Kubernetes sont des VMs (ex: EC2 sur AWS)

### Comment cette stack se présente dans différents contextes ?

**Cloud (ex: AWS, Azure, GCP) :**
- Hyperviseur → VMs (EC2/instances) → k8s nodes → Pods → Containers
- Le cloud provider gère les VMs, l'équipe DevOps gère Kubernetes

**Système embarqué automobile :**
- Hardware dédié → OS temps réel → containers légers (k3s/k0s)
- Contraintes fortes : faible RAM, pas de cloud, haute disponibilité
- Kubernetes version allégée pour gérer les mises à jour OTA

**Institution financière :**
- Datacenter privé → VMs sur VMware/Hyper-V → Kubernetes on-premise
- Réglementation stricte : données ne quittent pas le datacenter
- Isolation réseau forte, audit de chaque accès

# Etape 10 : 

## Controlled Failure — Analyse

### Failure introduite
Image invalide : `quote-app:version-inexistante`

### Ce qui a échoué en premier
Le container `quote-app` n'a pas pu démarrer car l'image n'existe pas.
Kubernetes a tenté de la télécharger depuis Docker Hub et a reçu une
erreur d'autorisation (image introuvable).

### Signal le plus rapide
`kubectl get events -n quote-lab` a montré immédiatement :
`Error: ImagePullBackOff` et `Failed to pull image`

### Comportement de Kubernetes pendant la panne
Kubernetes a gardé l'ancien pod en état Running pendant que le nouveau
pod échouait. L'application est restée accessible pour les utilisateurs
grâce au rolling update qui ne détruit pas l'ancien pod tant que le
nouveau n'est pas prêt.

### Ce qu'on vérifierait en production
1. `kubectl get events -n <namespace>` → premier réflexe
2. `kubectl describe pod <pod>` → détails de l'erreur
3. Vérifier que le tag d'image existe dans le registry
4. Vérifier les droits d'accès au registry

# Etape 11 : 

## Configuration via Secrets Kubernetes

### Pourquoi c'est mieux que la configuration en clair ?
- Les mots de passe ne sont pas visibles dans les fichiers YAML
  commitées sur GitHub
- Les secrets sont encodés en base64 dans etcd (la base de Kubernetes)
- On peut donner accès au deployment.yaml sans exposer les credentials
- Les secrets peuvent être mis à jour sans modifier le manifest
- Dans un vrai cluster, les accès aux secrets sont contrôlés par RBAC

### Les Secrets sont-ils chiffrés par défaut ?
Non, pas par défaut. Les secrets Kubernetes sont encodés en base64
mais pas chiffrés. Ils sont stockés en clair dans etcd.
Pour un vrai chiffrement, il faut activer "Encryption at Rest" dans
la configuration du cluster, ou utiliser des solutions externes comme
HashiCorp Vault ou AWS Secrets Manager.

# Etape 12 : 

## Rollout contrôlé et Rollback

### Ce qui a changé pendant le rollout v1 → v2
- Kubernetes a créé un nouveau pod avec l'image v2
- Il a attendu que le nouveau pod soit Ready avant de supprimer l'ancien
- Le trafic a continué sans interruption pendant la transition

### Ce qui n'a pas changé
- Le Service (même IP, même port)
- La base de données et ses données
- Le namespace et la configuration

### Comment Kubernetes décide quand créer/supprimer les pods
Il attend que la readiness probe du nouveau pod soit OK avant de
supprimer l'ancien. Cela garantit zéro downtime pendant le rollout.

### Rollout cassé — ce qui a échoué en premier
L'image `quote-app:version-cassee` n'existait pas → `ImagePullBackOff`
Le signal le plus rapide : `kubectl get pods` montrant `ImagePullBackOff`
En production on vérifierait : le nom du tag dans le registry

### Ce que le rollback a changé
- Kubernetes est revenu à la révision précédente (v2)
- Le pod cassé a été supprimé
- Un nouveau pod sain a été recréé

### Ce que le rollback n'a pas changé
- Les données en base de données
- Le Service et sa configuration
- L'historique des révisions (toujours visible avec rollout history)

## Rollout Strategy (Option A)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```
- **maxSurge: 1** → autorise 1 pod supplémentaire pendant le rollout
- **maxUnavailable: 0** → aucun pod ne peut être indisponible pendant
  le rollout, garantissant zéro downtime
- On choisit 0 pour maxUnavailable quand la disponibilité est critique




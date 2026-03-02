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
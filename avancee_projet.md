# JOUR 1

## 1. État de l'art des outils RBAC Kubernetes

### 1.1 RBAC Tool

**Installation**
```bash
git clone https://github.com/alcideio/rbac-tool.git
```

**Fonctionnalités**
- Visualisation des politiques RBAC sous forme de graphe (`viz`)
- Analyse des permissions trop larges ou risquées (`analysis`)
- Recherche des autorisations (`who-can`)
- Génération automatique de rôles

**Points forts**
- Outil très complet avec de nombreuses fonctionnalités
- Génération de graphe visuel en HTML ou PNG

**Limites**
- Écrit en Go (pas en Python)
- Interface CLI uniquement, peu accessible
- Graphe basique (format dot/graphviz)

**Exemples d'utilisation**
```bash
# Générer un graphe au format dot
./bin/rbac-tool viz --outformat dot

# Vérifier qui peut récupérer les pods
./bin/rbac-tool who-can get pods
```

**Génération d'image PNG**
```bash
brew install graphviz
# Puis convertir le fichier .dot en PNG
dot -Tpng rbac.dot > rbac.png
open rbac.png
```

---

### 1.2 RBAC View

**Installation**
```bash
git clone https://github.com/jasonrichardsmith/rbac-view.git
```

**Fonctionnalités**
- Visualisation interactive des permissions RBAC dans le navigateur
- Deux modes : HTML (interface web) ou JSON

**Points forts**
- Interface web claire et agréable

**Limites**
- Projet abandonné (plus maintenu activement)
- Installation complexe (Go + npm)
- Pas de génération d'image, uniquement web live
- Non compatible avec Mac Apple Silicon (ARM)

**Tentative d'installation avec kubectl krew**
```bash
brew install krew
kubectl krew install rbac-view
# → Erreur : non compatible avec Mac Apple Silicon
```

---

### 1.3 Krane

**Fonctionnalités**
- Dashboard web complet avec vue graphe et arborescence
- Détection des permissions dangereuses
- Alertes Slack
- Peut tourner directement dans un cluster Kubernetes
- Base de données graphe (RedisGraph) pour requêter efficacement

**Points forts**
- Outil le plus complet des trois

**Limites**
- Installation très complexe (Docker, RedisGraph, Ruby...)
- Écrit en Ruby (pas en Python)
- Trop lourd pour une simple visualisation RBAC
- Difficile à faire tourner sur Mac Apple Silicon

**Tentative d'utilisation**
```bash
cd krane
docker-compose up -d
docker-compose ps

# Entrer dans le conteneur
docker-compose exec -e KUBECONFIG=/root/.kube/config krane bash

# Lancer un rapport
krane report -k minikube
# → Non fonctionnel (problèmes de certificats sur Mac Apple Silicon)
```

---

### 1.4 Récapitulatif des trois outils

| Outil | Langage | Visualisation | Analyse sécu | Facilité |
|-------|---------|---------------|--------------|----------|
| RBAC Tool | Go | Graphe basique | ✅ | Moyenne |
| RBAC View | Go | Interface web | ❌ | Difficile (abandonné) |
| Krane | Ruby | Dashboard complet | ✅✅ | Très complexe |

**Conclusion** : Aucun de ces outils n'est léger, écrit en Python et intégré à un outil d'architecture Kubernetes existant.

---

## 2. Analyse interne : KubeDiagrams

**Dépôt** : https://github.com/philippemerle/KubeDiagrams

KubeDiagrams est un outil Python qui génère des diagrammes d'architecture Kubernetes à partir de fichiers YAML. Il supporte déjà les objets RBAC grâce aux icônes de la bibliothèque `diagrams` de mingrammer.

### 2.1 Ce qui existe déjà dans KubeDiagrams

**Icônes disponibles**
```python
diagrams.k8s.rbac.CRole   # ClusterRole
diagrams.k8s.rbac.CRB     # ClusterRoleBinding
diagrams.k8s.rbac.Role    # Role
diagrams.k8s.rbac.RB      # RoleBinding
diagrams.k8s.rbac.SA      # ServiceAccount
diagrams.k8s.rbac.User    # User
diagrams.k8s.rbac.Group   # Group
```

**Fonctions existantes dans `kube-diagrams`**
```python
add_subjects()
# → crée les flèches entre les ServiceAccounts/Users/Groups et les RoleBindings

add_role()
# → crée la flèche entre un RoleBinding et son Role

add_rules_resource_names()
# → crée des flèches vers des ressources nommées explicitement dans les rules
#   (uniquement quand resourceNames est défini)
```

**Ce que ça donne visuellement**
```
[ServiceAccount] ──→ [RoleBinding] ──→ [Role]
```

### 2.2 Limites identifiées

1. **Verbes non affichés** : les règles (get, list, create, delete...) ne sont pas visibles sur les flèches. On ne sait pas ce qu'un Role peut faire concrètement.

2. **Ressources cibles absentes** : `add_rules_resource_names()` gère uniquement les ressources nommées explicitement (`resourceNames`), mais pas le cas général où un Role donne accès à tous les pods ou tous les secrets.

**Ce qui manque — le cas général non géré**
```yaml
rules:
- resources: ["pods", "secrets"]  # ← cas pas géré
  verbs: ["get", "list"]
```

### 2.3 Ce que le projet doit apporter

Créer une fonction `add_rules()` dans `kube-diagrams` qui :
- Lit toutes les `rules` d'un Role ou ClusterRole
- Crée des flèches vers les ressources cibles (Pods, Secrets, ConfigMaps...)
- Affiche les verbes sur les flèches (get, list, create...)

**Rendu attendu**
```
[ServiceAccount] ──→ [RoleBinding] ──→ [Role] ──get, list──→ [pods]
                                              ──*──────────→ [secrets]
                                              ──create────→ [configmaps]
```

---

# JOUR 2

## 3. Rappels sur le RBAC Kubernetes

### 3.1 Les verbes possibles

| Verbe | Ce que ça fait |
|---|---|
| `get` | Lire un objet précis |
| `list` | Lister tous les objets |
| `watch` | Observer les changements en temps réel |
| `create` | Créer un objet |
| `update` | Modifier un objet |
| `patch` | Modifier partiellement |
| `delete` | Supprimer un objet |
| `deletecollection` | Supprimer plusieurs objets |
| `*` | Tout faire |

### 3.2 Les objets RBAC 

**Role**
```yaml
kind: Role                        # c'est un badge de permissions
name: mon-role                    # le nom du badge
rules:                            # la liste des règles du badge
  - resources: ["pods"]           # règle 1 : sur les pods
    verbs: ["get", "list"]        # → on peut juste lire, pas modifier ni supprimer

  - resources: ["secrets"]        # règle 2 : sur les secrets (mots de passe)
    verbs: ["*"]                  # → on peut tout faire (dangereux !)

  - resources: ["configmaps"]     # règle 3 : sur les configs
    verbs: ["create", "update"]   # → on peut créer et modifier, mais pas supprimer
```

**ClusterRole**
```yaml
kind: ClusterRole                 # pareil qu'un Role mais s'applique à tout le cluster
name: mon-cluster-role
rules:
  - resources: ["nodes"]          # règle 1 : sur les serveurs physiques
    verbs: ["get", "list"]        # → lecture seule (on ne veut pas qu'on supprime des serveurs !)

  - resources: ["pods"]           # règle 2 : sur les applications
    verbs: ["*"]                  # → droits complets sur tous les pods du cluster
```

**RoleBinding**
```yaml
kind: RoleBinding                 # c'est le lien entre des utilisateurs et un badge
name: mon-binding
subjects:                         # qui reçoit le badge (peut être plusieurs personnes)
  - kind: ServiceAccount          # type : compte applicatif
    name: asma                    # asma reçoit le badge
  - kind: User                    # type : utilisateur humain
    name: aya                 # aya reçoit aussi le badge
roleRef:                          # quel badge on leur donne
  kind: Role                      # c'est un Role (pas un ClusterRole)
  name: mon-role                  # le badge s'appelle mon-role
```

**ClusterRoleBinding**
```yaml
kind: ClusterRoleBinding          # pareil mais à l'échelle du cluster entier
subjects:
  - kind: Group                   # type : groupe d'utilisateurs
    name: admins                  # tous les admins reçoivent le badge
roleRef:
  kind: ClusterRole               # c'est un ClusterRole cette fois
  name: mon-cluster-role          # le badge cluster s'appelle mon-cluster-role
```

### 3.3 Récap visuel complet

```
[User: asma]  [User: aya]
       ↘           ↙
    [RoleBinding: mon-binding]
               ↓
          [Role: mon-role]
          ↙          ↓            ↘
  [pods]         [secrets]    [configmaps]
  get, list          *         create, update
```

---

## 4. Notes complémentaires

### Pourquoi les PVC apparaissent sans être déclarés ?

(exemple dans issues/issue#2) pq 6 objets alors que 4 créés ? 
Dans un StatefulSet, le champ `volumeClaimTemplates` dit à Kubernetes de créer automatiquement un PVC pour chaque replica. KubeDiagrams lit ce champ et crée automatiquement le nœud PVC dans le diagramme via `add_volume_claim_templates()`. C'est une bonne inspiration pour `add_rules()` : lire les `rules` et créer automatiquement les nœuds ressources cibles.

### Le fichier semiotics.yaml

C'est le fichier de démonstration de KubeDiagrams qui montre un exemple de chaque type de ressource Kubernetes supportée. Le RBAC y est déjà présent mais les flèches entre les Roles et les ressources cibles sont absentes : ce que le projet doit ajouter.

---

## 5. À faire

1. Tester KubeDiagrams sur `semiotics.yaml` pour voir le rendu actuel
2. Comprendre en détail comment `add_rules_resource_names()` fonctionne
3. Implémenter `add_rules()` dans `kube-diagrams`
4. Ajouter l'appel à `add_rules()` dans `kube-diagrams.yaml` pour Role et ClusterRole
5. Tester sur des fichiers YAML RBAC réels récupérés sur GitHub


**Liens**
- https://github.com/philippemerle/KubeDiagrams
- https://kubediagrams.lille.inria.fr
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/
# Suivi de projet — Visualisation RBAC dans KubeDiagrams
Aya Boudjou & Asma Benyahia

---

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

# Convertir le fichier .dot en PNG
brew install graphviz
dot -Tpng rbac.dot > rbac.png
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

**Tentative d'installation**
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
- Installation très complexe (Docker, RedisGraph, Ruby…)
- Écrit en Ruby (pas en Python)
- Trop lourd pour une simple visualisation RBAC
- Difficile à faire tourner sur Mac Apple Silicon

**Tentative d'utilisation**
```bash
cd krane
docker-compose up -d
docker-compose ps
docker-compose exec -e KUBECONFIG=/root/.kube/config krane bash
krane report -k minikube
# → Non fonctionnel (problèmes de certificats sur Mac Apple Silicon)
```

---

### 1.4 Récapitulatif

| Outil     | Langage | Visualisation     | Analyse sécu | Facilité d'installation |
|-----------|---------|-------------------|--------------|------------------------|
| RBAC Tool | Go      | Graphe basique    | Oui          | Moyenne                |
| RBAC View | Go      | Interface web     | Non          | Difficile (abandonné)  |
| Krane     | Ruby    | Dashboard complet | Oui          | Très complexe          |

**Conclusion** : Aucun de ces outils n'est léger, écrit en Python, ni intégré à un outil d'architecture Kubernetes existant.

---

## 2. Analyse de KubeDiagrams

**Dépôt** : <https://github.com/philippemerle/KubeDiagrams>

KubeDiagrams est un outil Python qui génère des diagrammes d'architecture Kubernetes à partir de fichiers YAML. Il supporte déjà les objets RBAC grâce aux icônes de la bibliothèque `diagrams` de mingrammer.

### 2.1 Ce qui existe déjà

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

**Fonctions existantes dans `kube-diagrams`** (membres de la classe `EdgesContext`)

| Fonction | Rôle |
|---|---|
| `add_subjects()` | Crée les flèches entre ServiceAccounts/Users/Groups et les RoleBindings |
| `add_role()` | Crée la flèche entre un RoleBinding et son Role |
| `add_rules_resource_names()` | Crée des flèches vers des ressources nommées explicitement (`resourceNames`) — les verbes ne sont **pas** affichés |

**Rendu actuel (avant `add_rules()`)**
```
[ServiceAccount] ──→ [RoleBinding] ──→ [Role]
```

### 2.2 Limites identifiées

1. **Verbes non affichés** : les règles (get, list, create, delete…) ne sont pas visibles sur les flèches.
2. **Ressources cibles absentes** : `add_rules_resource_names()` gère uniquement les ressources nommées explicitement, pas le cas général.

**Cas non géré**
```yaml
rules:
  - resources: ["pods", "secrets"]
    verbs: ["get", "list"]
```

### 2.3 Objectif : implémenter `add_rules()`

Créer une fonction `add_rules()` dans `kube-diagrams` qui :
- lit toutes les `rules` d'un Role ou ClusterRole
- crée des flèches vers les ressources cibles
- affiche les verbes

**Rendu visé**
```
[ServiceAccount] ──→ [RoleBinding] ──→ [Role] ──get, list──→ [pods]
                                              ──*──────────→ [secrets]
                                              ──create────→ [configmaps]
```

---

## 3. Rappel RBAC Kubernetes

### 3.1 Les verbes possibles

| Verbe              | Description                            |
|--------------------|----------------------------------------|
| `get`              | Lire un objet précis                   |
| `list`             | Lister tous les objets                 |
| `watch`            | Observer les changements en temps réel |
| `create`           | Créer un objet                         |
| `update`           | Modifier un objet                      |
| `patch`            | Modifier partiellement                 |
| `delete`           | Supprimer un objet                     |
| `deletecollection` | Supprimer plusieurs objets             |
| `*`                | Tous les droits                        |

### 3.2 Les objets RBAC

**Role**
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create", "update"]
```

**ClusterRole**
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]
    resources: ["pods"]
    verbs: ["*"]
```

**RoleBinding**
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mon-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: asma
    namespace: default
  - kind: User
    name: aya
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRoleBinding**
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mon-cluster-binding
subjects:
  - kind: Group
    name: admins
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-reader
  apiGroup: rbac.authorization.k8s.io
```

### 3.3 Récap visuel

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

### 3.4 Principes RBAC (source : institute.sfeir)

RBAC fonctionne en deux étapes :
1. Créer un Role qui liste les permissions
2. Créer un RoleBinding qui attribue ce Role à un utilisateur

Principe fondamental : **accorder toujours le minimum de privilèges nécessaires**.

- **Attribution de rôle** : un utilisateur doit se voir attribuer un ou plusieurs rôles actifs pour pouvoir exercer des permissions.
- **Autorisation de rôle** : l'utilisateur doit être autorisé à assumer le ou les rôles qui lui ont été attribués.
- **Autorisation de permissions** : les permissions ne sont accordées qu'aux utilisateurs autorisés par l'attribution de leurs rôles.

---

## 4. Notes complémentaires

### Le fichier `semiotics.yaml`

Fichier de démonstration de KubeDiagrams montrant un exemple de chaque type de ressource Kubernetes supportée. Le RBAC y est déjà présent, mais les flèches entre les Roles et les ressources cibles sont absentes → à ajouter (voir issue #2).

### Fichiers de référence dans `rbac-tool`

| Fichier | Intérêt |
|---|---|
| `cmd/visualize_cmd.go` | Modèle pour définir des options CLI (`--show-rules`, `--exclude-namespaces`) |
| `pkg/rbac/subject_permissions.go` | Modélisation des permissions en mémoire — inspiration pour structurer les données extraites des YAML en Python |
| `pkg/visualize/rbacviz.go` | Logique de génération du graphe : itération sur les RoleBindings, création de sous-graphes par namespace, affichage des règles |

---

## 5. Conception de `add_rules()`

### Distinction `add_rules_resource_names()` vs `add_rules()`

`add_rules_resource_names()` — cas spécifique avec `resourceNames` :
- Crée une flèche vers la ressource nommée exactement (doit exister dans le YAML)
- Les verbes **ne sont pas** affichés

`add_rules()` — cas général sans `resourceNames` :
- Crée une flèche vers toutes les ressources du type concerné trouvées dans le YAML
- Les verbes **sont affichés** dans le nœud intermédiaire

### Nœud intermédiaire « permissions »

Pour éviter l'illisibilité des flèches avec labels, un nœud rect intermédiaire est inséré entre le Role et la ressource cible, affichant les verbes sous forme d'initiales colorées. Placé à l'intérieur du cluster namespace.

```
[Role] → [G L] → [Pod]
```

### Affichage des verbes — initiales colorées

Graphviz ne supportant pas le HTML/CSS complet, les verbes sont affichés sous forme d'initiales colorées via `generate_label()`. Code couleur inspiré de `rbac-view` :

| Initiale | Verbe              | Couleur  |
|----------|--------------------|----------|
| G        | get                | 🟡 Jaune  |
| L        | list               | 🔵 Bleu   |
| W        | watch              | 🟤 Marron |
| C        | create             | 🟢 Vert   |
| U        | update             | 🩷 Rose   |
| D        | delete             | 🔴 Rouge  |
| P        | patch              | 🩶 Gris   |
| DC       | deletecollection   | ⚫ Noir   |
| ★        | *                  | 🟠 Orange |

### Nœud générique `All <Kind>`

Quand une règle cible un type de ressource absent du YAML, un nœud générique `All <Kind>` est créé (ex: `All Pods`, `All Secrets`).

```
[Role: pod-reader] ──G L──→ [All Pods]
```

### Cas particuliers traités

| Cas | Traitement |
|---|---|
| `resources: ["*"]` | Ignoré |
| `resources: ["pods/log"]` (sous-ressource) | Ignoré |
| `verbs: ["*"]` | Conservé, affiché ★ |
| Plusieurs resources dans une règle | Une flèche par ressource |
| `resourceNames` combiné avec `resources` | Délégué à `add_rules_resource_names()` |
| Ressource absente du YAML | Nœud générique `All <Kind>` |
| `apiGroups: [""]` | Converti en `"v1"` |
| `nonResourceURLs` | Ignoré |

### Bugs rencontrés et résolus

**Nœud permission flottant** : créé avant de vérifier si la ressource cible existait. Résolu en déplaçant la vérification en amont.

**Icônes vides** : noms contenant `:` (ex: `cert-manager-webhook:dynamic-serving`), invalide dans un identifiant Graphviz. Résolu en remplaçant les caractères invalides.

**Flèche manquante pour une ressource partagée** (1 ressource, 2 rôles) : résolu.

### Clé de déduplication

```python
permission_key  # tuple unique identifiant une permission
                # évite les doublons Workload → Permission → Ressource
```

Regroupement des permissions identiques : quand plusieurs règles d'un même Role ciblent la même ressource, un seul nœud est créé avec l'union de tous les verbes.

### Options en ligne de commande

- **`--show-verbs`** : activer/désactiver l'affichage des verbes
- **`--show-rbac-legend`** : afficher une légende des initiales (G, L, W, C…)
- **`--simplify-rbac`** : masquer ServiceAccount, RoleBinding, Role, ClusterRole et ClusterRoleBinding pour n'afficher que Workload → Permission → Ressources

### Mode simplifié (`--simplify-rbac`)

`process_edges()` a été modifié pour continuer à exécuter les edges des Role/ClusterRole même quand ils sont masqués, afin que `add_rules()` soit appelée. Le diagramme affiche directement : **Workload → Permission → Ressources**.

Pour remonter la chaîne :
1. Parcourir les RoleBindings pointant vers le Role/ClusterRole
2. Récupérer les ServiceAccounts dans les subjects
3. Identifier les Workloads utilisant ce ServiceAccount
4. Créer la flèche Workload → nœud permission

`created_workload_permission_edges` évite les doublons.

### Pistes envisagées et abandonnées

> ❌ **Regrouper les `resourceNames` en un seul nœud** : le nœud créé n'était pas attaché au bon cluster Graphviz et n'apparaissait pas sur le diagramme. La gestion des clusters dans KubeDiagrams ne permet pas facilement ce type de regroupement dynamique.

> ❌ **Cacher les nœuds RBAC via `show: false` dans `kube-diagrams.yaml`** : si le Role est masqué, ses edges ne sont plus exécutées → plus de nœuds permission créés. Contourner cela aurait nécessité de remonter toute la chaîne depuis les Workloads : refonte trop lourde.

> ❌ **Regroupement des ressources avec seuil (THRESHOLD)** : quand un Role agissait sur beaucoup de ressources d'un même type, elles étaient regroupées sous un nœud `Pods (12)` au lieu d'être affichées individuellement. Abandonné le 26/05 : ce format masque l'information réelle et produit des nœuds génériques redondants avec `All <Kind>`. On conserve uniquement `All <Kind>` pour les ressources absentes du YAML.

> ❌ **Fusion des nœuds permission entre plusieurs Roles** (01/06) : l'idée était de fusionner en un seul nœud permission les verbes de plusieurs Roles différents quand ils partagent le même Workload source et la même ressource cible. Abandonné car un Workload est souvent lié à plusieurs ClusterRoles pour des raisons métier distinctes — fusionner leurs verbes ferait perdre l'information sur quel Role donne quel droit. De plus, `add_rules()` est appelée depuis chaque Role individuellement et ne voit que ses propres règles, ce qui rendrait la fusion techniquement complexe à implémenter proprement.

---

## 6. Exemple de fichiers YAML pour tester `add_rules()`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod-precis
  namespace: default
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: asma
  namespace: default
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    resourceNames: ["mon-pod-precis"]
    verbs: ["get"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: asma-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: asma
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 7. Commandes de test

```bash
python3 bin/kube-diagrams test.yaml -o test_output.png
python3 bin/kube-diagrams examples/kube-prometheus-stack/kube-prometheus-stack-corrected.yaml -o test_output2.png
python3 bin/kube-diagrams examples/argo/argo-cd-manifests-install-corrected.yaml -o test_output3.png
python3 bin/kube-diagrams examples/helm-charts/cert-manager.yaml -o test_output4.png
python3 bin/kube-diagrams examples/custom-object-items/config/custom-object-items.yaml -o test_output5.png
python3 bin/kube-diagrams examples/opentelemetry-demo/downloads/opentelemetry-demo.yaml -o test_output6.png
```

---

## 8. À faire

- [x] Tester KubeDiagrams sur `semiotics.yaml`
- [x] Comprendre `add_rules_resource_names()`
- [x] Implémenter `add_rules()` dans `kube-diagrams`
- [x] Ajouter l'appel à `add_rules()` dans `kube-diagrams.yaml` pour Role et ClusterRole
- [x] Affichage des verbes sous forme d'initiales colorées (`generate_label()`)
- [x] Ajustement des couleurs du nœud permissions
- [x] Nœud intermédiaire « permissions » avec titre, placé dans le cluster namespace
- [x] Nœud générique `All <Kind>` quand la ressource est absente du YAML
- [x] Flèche correcte pour une ressource partagée entre plusieurs rôles
- [x] Déduplication des arêtes (`permission_key`)
- [x] Regroupement des verbes d'un même Role vers la même ressource cible
- [x] Option `--show-rbac-legend`
- [x] Option `--show-verbs`
- [x] Option `--simplify-rbac`
- [x] Tester sur des fichiers YAML RBAC réels récupérés sur GitHub
- [x] Vérifier les fichiers YAML avec les diagrammes générés
- [x] Écrire le rapport (FR + EN)
- [ ] Préparer la soutenance

---

## Liens

- <https://github.com/philippemerle/KubeDiagrams>
- <https://kubediagrams.lille.inria.fr>
- <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
# kube-prometheus-stack (Prometheus, Alertmanager, Grafana)

Ce composant installe les CRDs Prometheus Operator, le stack complet de monitoring (Prometheus Operator, Prometheus, Alertmanager, Grafana, kube-state-metrics, node-exporter) via le chart `kube-prometheus-stack`, et expose Grafana derrière la `Gateway` partagée (`platform-gateway`) livrée par le composant Gateway API/NGF.

## Architecture

Cette plateforme repose sur trois rôles distincts :

- **VM Mirror** — héberge le registre Docker (images) et le dépôt Helm (charts `.tgz`). C'est une source, rien ne s'y exécute.
- **VM PIC** — lance les playbooks Ansible (`ansible-playbook`), pilote le cluster via `kubectl`/`helm`. Contient les projets `bundle/` + `deploy-<composant>/`.
- **Cluster K8s** — exécute les workloads, doit pouvoir joindre le Mirror pour tirer images et charts.

**Point important** : la grande majorité des erreurs `ImagePullBackOff` rencontrées sur ce composant ne viennent pas du chart lui-même, mais d'un override d'image manquant ou incomplet. `kube-prometheus-stack` est un chart à 9 images réparties sur 6 sous-composants différents (Operator, Prometheus, Alertmanager, Grafana + son sidecar, node-exporter, kube-state-metrics), et certaines images (notamment `prometheus-config-reloader`) ne sont **jamais visibles** dans un simple `helm template | grep image:` — elles sont injectées au runtime par l'Operator lui-même. Avant de creuser un pod en échec, toujours comparer la liste des images réellement utilisées (`kubectl get pods -n monitoring -o jsonpath='{...}.image}'`) au `mapping.txt` du bundle.

Grafana est exposé via la `Gateway` partagée `platform-gateway` (composant Gateway API/NGF, documenté séparément). Ce composant ne crée que le `HTTPRoute` (namespace `monitoring`, `parentRefs` vers `platform-gateway`/`gateway-system`) — le `Certificate` TLS de Grafana est géré ailleurs, centralisé dans `gateway-system` avec ceux de Harbor et ArgoCD.

---

## 1️⃣ Déploiement

Depuis la VM PIC :

```bash
cd deploy-prometheus
ansible-playbook playbook-crds.yml     # installe les 10 CRDs Prometheus Operator
ansible-playbook playbook.yml          # crée le namespace, déploie le chart Helm kube-prometheus-stack
ansible-playbook playbook-route.yml    # crée le HTTPRoute Grafana sur platform-gateway
```

**Ordre strict à respecter** — chaque playbook dépend du précédent :

```
playbook-crds.yml
        ↓ (les CRDs Prometheus/Alertmanager/ServiceMonitor... doivent exister
           avant que l'Operator ne démarre, sinon il crash au boot)
playbook.yml
        ↓ (le Service kube-prometheus-stack-grafana doit exister
           avant de créer le HTTPRoute qui le référence)
playbook-route.yml
```

> **Prérequis externe** : `platform-gateway` (namespace `gateway-system`) et le `Certificate grafana-tls` doivent déjà exister avant de lancer `playbook-route.yml` — tous deux sont créés par le composant Gateway API/NGF, pas par celui-ci.

---

## 2️⃣ Vérifications

**CRDs installées** (10 au total) :

```bash
kubectl get crds | grep monitoring.coreos.com
```

Attendu : `alertmanagerconfigs`, `alertmanagers`, `podmonitors`, `probes`, `prometheusagents`, `prometheuses`, `prometheusrules`, `scrapeconfigs`, `servicemonitors`, `thanosrulers`.

**Pods du stack** :

```bash
kubectl get pods -n monitoring
```

Attendu, tous `Running` : `kube-prometheus-stack-operator-*` (1/1), `prometheus-kube-prometheus-stack-prometheus-0` (2/2 — conteneur `prometheus` + sidecar `config-reloader`), `alertmanager-kube-prometheus-stack-alertmanager-0` (2/2, même schéma), `kube-prometheus-stack-grafana-*` (3/3 — `grafana` + 2 sidecars `k8s-sidecar` pour dashboards/datasources), `kube-prometheus-stack-kube-state-metrics-*` (1/1), un `prometheus-node-exporter` par nœud du cluster (1/1 chacun).

**Stockage persistant** (pas de fallback silencieux en `emptyDir`) :

```bash
kubectl get pvc -n monitoring
```

Attendu : une PVC `Bound` pour Prometheus, taille et StorageClass conformes à `group_vars/all.yml`.

**Images réellement utilisées** (toutes doivent pointer vers le registre Mirror, aucune vers `quay.io`/`docker.io`/`ghcr.io`/`registry.k8s.io`) :

```bash
kubectl get pods -n monitoring -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}  {.name}: {.image}{"\n"}{end}{end}'
```

**HTTPRoute Grafana** :

```bash
kubectl describe httproute grafana-route -n monitoring
```

Attendu : `Accepted: True`, `ResolvedRefs: True` sur la condition de parent.

**Rétention et config effective de Prometheus** — les manifestes Kubernetes (`args:` du pod) ne reflètent pas toujours la config réelle sur ce chart (elle est portée par un fichier généré dynamiquement, pas par les arguments CLI) : la source de vérité est l'API runtime de Prometheus lui-même :

```bash
kubectl port-forward -n monitoring prometheus-kube-prometheus-stack-prometheus-0 9090:9090 &
curl -s http://localhost:9090/api/v1/status/flags | grep retention
kill %1
```

---

## 3️⃣ Test fonctionnel

Test end-to-end sur le hostname Grafana, via `platform-gateway` :

```bash
curl -v --resolve grafana.dev.local:443:<IP_PLATFORM_GATEWAY> https://grafana.dev.local/login -k -I
```

Attendu : `HTTP/2 200`, `server: nginx`. Le `-v` permet d'inspecter la chaîne de certificats retournée : elle doit correspondre au `Certificate grafana` créé dans `gateway-system`, signé par `internal-ca-issuer`.

Récupération des identifiants Grafana (password auto-généré par le chart au premier déploiement) :

```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath="{.data.admin-user}" | base64 -d && echo
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d && echo
```

---

## Contenu des playbooks

**`playbook-crds.yml`**

Rôle unique : applique les 10 CRDs Prometheus Operator en *server-side apply* (`apply: true` + `server_side_apply`), gérées hors du cycle de vie Helm. Les CRDs sont lues depuis le sous-chart `crds/` vendored (`charts/kube-prometheus-stack/charts/crds/crds/`), pas d'un manifeste GitHub externe comme pour Gateway API. Une tâche d'attente boucle sur la condition `Established` pour chacune des 10 CRDs avant de continuer.

> **Piège upstream connu** : la CRD `AlertmanagerConfig` contient un bug de génération YAML — un `enum:` de matcher (`matchType`) porte une valeur `=` non quotée, que le parseur YAML interprète comme un tag spécial (`tag:yaml.org,2002:value`) et rejette. Le fichier `crds/crd-alertmanagerconfigs.yaml` du bundle a été corrigé (3 occurrences) via `sed -i "s/^\(\s*\)- =\s*$/\1- '='/"` avant packaging — ce correctif doit être reconduit si le chart est mis à jour vers une nouvelle version du sous-chart CRDs.

**`playbook.yml`**

- *Create monitoring namespace* : crée le namespace `monitoring`
- *Add internal Helm repository* : ajoute le dépôt Helm pointant vers la VM Mirror
- *Render values* : génère `values-kube-prometheus-stack.yaml` à partir du template — overrides d'images pour les 9 composants (Operator, config-reloader, webhook patch/deployment, Alertmanager, Prometheus, node-exporter, Grafana, sidecar, kube-state-metrics), stockage (`storageSpec.volumeClaimTemplate`), réplicas, rétention (`retention` + `retentionSize`), et `crds.enabled: false` pour ne jamais laisser Helm gérer les CRDs
- *Deploy via Helm* : installe/upgrade le release `kube-prometheus-stack`
- *Verify pods Running* : liste et affiche la phase de chaque pod du namespace

**`playbook-route.yml`**

- *Create Grafana HTTPRoute* : crée le `HTTPRoute grafana-route` dans `monitoring`, `parentRefs` vers `platform-gateway`/`gateway-system`, backend `kube-prometheus-stack-grafana:80`. Ne crée ni `Certificate` ni `Gateway` — ces deux objets sont gérés par le composant Gateway API/NGF.

---

## Variables clés (`group_vars/all.yml`)

```yaml
# --- Registre et dépôt Helm internes ---
image_registry: "192.168.1.20:5000/prometheus"

# --- Stockage Prometheus (TSDB) ---
prometheus_storage_class: "ceph-rbd"
prometheus_pvc_size: "50Gi"

# --- HA ---
prometheus_replicas: 1
alertmanager_replicas: 1

# --- Exposition Grafana (via platform-gateway) ---
grafana_namespace: "monitoring"
grafana_hostname: "grafana.dev.local"

# --- Helm release ---
prometheus_helm_repo_name: "internal-helm-repo"
prometheus_helm_repo_url: "http://192.168.1.20:8080"
prometheus_release_name: "kube-prometheus-stack"
prometheus_chart_name: "kube-prometheus-stack"
prometheus_chart_version: "88.1.5"

# --- Tags des images (registre commun: image_registry) ---
prometheus_operator_tag: "v0.93.0"
prometheus_config_reloader_tag: "v0.93.0"
kube_webhook_certgen_tag: "1.8.5"
alertmanager_tag: "v0.33.1"
prometheus_tag: "v3.13.2-distroless"
node_exporter_tag: "v1.12.1"
grafana_tag: "13.1.2"
k8s_sidecar_tag: "2.10.1"
kube_state_metrics_tag: "v2.19.1"

# --- Rétention TSDB ---
prometheus_retention: "10d"
prometheus_retention_size: "40GiB"

# --- Gateway partagée (référence au composant Gateway API/NGF) ---
gateway_system_namespace: "gateway-system"
platform_gateway_name: "platform-gateway"
```

> **Note sur les images distroless** — `node-exporter` et `prometheus` utilisent la variante `distroless` (image allégée, sans shell ni outils système, inutilisable avec `kubectl exec -it ... sh`). Le sous-chart `prometheus-node-exporter` construit le tag final en ajoutant automatiquement le suffixe `-distroless` si `image.distroless: true` est positionné — le tag dans `group_vars/all.yml` ne doit **jamais** inclure ce suffixe manuellement (`node_exporter_tag: "v1.12.1"`, pas `"v1.12.1-distroless"`), sous peine de doublon (`v1.12.1-distroless-distroless`) et d'échec de pull.

---

## Pratique : ajouter un dashboard Grafana personnalisé

Le chart embarque un sidecar `k8s-sidecar` sur le pod Grafana, qui surveille dynamiquement les `ConfigMap` du cluster portant un label spécifique et les charge comme dashboards, sans redémarrage de Grafana.

1. **Créer le `ConfigMap`** contenant le JSON du dashboard, dans le namespace `monitoring`, avec le label attendu par le sidecar :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dashboard-<nom>
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  <nom>.json: |
    { ... contenu JSON exporté depuis Grafana ou grafana.com/dashboards ... }
```

2. **Appliquer** : `kubectl apply -f dashboard-<nom>.yaml`

3. **Vérifier la prise en compte** par le sidecar (pas besoin d'attendre, le montage est quasi immédiat) :

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard --tail=20
```

4. Le dashboard apparaît automatiquement dans l'UI Grafana, sans action supplémentaire.

Même mécanisme pour une datasource custom, avec le label `grafana_datasource: "1"` à la place.

# Airflow talos-02 — secrets Doppler et bootstrap cluster vierge

## Prérequis plateforme (repo Talos)

Avant `workloads-root` / Airflow :

1. Cluster Talos bootstrappé (Cilium, ingress-nginx, cert-manager, external-dns, Longhorn).
2. Doppler Operator actif : `make CLUSTER=talos-02 apply-doppler-token`
3. AppProject `workloads` autorise `https://airflow.apache.org` (déjà dans Talos `gitops/projects/workloads.yaml`).
4. `APPS_GIT_REPO` / `APPS_GIT_REVISION` dans `cluster.env` + `make gitops-render`.

## Variables Doppler (obligatoires avant le 1er sync `airflow`)

Projet **`k8s-talos-pve1`**, config **`prd`**.

| Variable Doppler | Clé Secret K8s | Génération |
|------------------|----------------|------------|
| `AIRFLOW_FERNET_KEY` | `fernet-key` | `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"` |
| `AIRFLOW_API_SECRET_KEY` | `api-secret-key` | `openssl rand -hex 32` |
| `AIRFLOW_JWT_SECRET` | `jwt-secret` | `openssl rand -hex 64` |
| `AIRFLOW_POSTGRES_PASSWORD` | `postgres-password` | `openssl rand -base64 24` |
| `AIRFLOW_METADATA_CONNECTION` | `connection` | voir ci-dessous |
| `AIRFLOW_ADMIN_PASSWORD` | `admin-password` | mot de passe UI (user `admin`) |

**`AIRFLOW_METADATA_CONNECTION`** — référence Doppler recommandée :

```text
postgresql+psycopg2://airflow:${AIRFLOW_POSTGRES_PASSWORD}@airflow-postgresql.airflow.svc:5432/airflow?sslmode=disable
```

Sans cette clé, le chart ne peut pas joindre PostgreSQL (plus de `postgres:postgres` généré par défaut).

## Ordre ArgoCD (cluster vierge)

```text
sync-wave -1  airflow-doppler  →  Namespace airflow + DopplerSecret
sync-wave  0  airflow          →  chart Helm 1.21 (Airflow 3.2.1)
```

Vérifier avant / pendant le premier sync `airflow` :

```bash
export KUBECONFIG=clusters/talos-02/kubeconfig   # repo Talos

kubectl -n airflow get secret airflow-credentials -o json \
  | jq -r '.data | keys[]' | grep -v DOPPLER
# Attendu : admin-password api-secret-key connection fernet-key jwt-secret postgres-password
```

## Connexion UI

| Champ | Valeur |
|-------|--------|
| URL | https://airflow-test.alexetnico.com |
| Utilisateur | `admin` |
| Mot de passe | `AIRFLOW_ADMIN_PASSWORD` (Doppler) |

```bash
kubectl -n airflow get secret airflow-credentials \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Les jobs Helm sont ordonnés pour ArgoCD : **PreSync** = ServiceAccount puis migration DB, déploiement principal, **PostSync** = ServiceAccount puis utilisateur `admin`. Les ServiceAccounts des jobs doivent porter le **même hook** que le Job (sinon `serviceaccount … not found`). Le job admin est idempotent (`create` ou `reset-password`).

**Important** : ne pas définir `webserver.defaultUser` dans `values.yaml` — le chart y lie `createUserJob.enabled` ; sans `enabled: true` explicite, le job admin ne part pas.

## Vérifications post-déploiement

```bash
kubectl -n argocd get application airflow-doppler airflow
kubectl -n airflow get pods
kubectl -n airflow get jobs | rg -i 'migrate|create-user'
curl -sI https://airflow-test.alexetnico.com | head -1
```

## Dépannage

| Symptôme | Cause probable | Action |
|----------|----------------|--------|
| ArgoCD bloqué sur Job migration | `connection` manquante ou mauvais mot de passe PG | Compléter Doppler, vérifier le secret |
| `password authentication failed for user postgres` | Ancien secret `airflow-metadata` | Vérifier `data.metadataSecretName: airflow-credentials` dans values |
| UI : identifiants invalides | `createUserJob` pas passé ou Doppler vide | Vérifier job `airflow-create-user` ; recréer user (voir ci-dessous) |
| Triggerer / dag-processor bloqués | PVC `airflow-dags` en RWO | Supprimer le PVC et resync (values : `ReadWriteMany`) |

Recréer l’admin manuellement :

```bash
PASS=$(kubectl -n airflow get secret airflow-credentials -o jsonpath='{.data.admin-password}' | base64 -d)
kubectl -n airflow exec deploy/airflow-scheduler -c scheduler -- \
  env AIRFLOW_ADMIN_PASSWORD="$PASS" bash -c \
  'airflow users reset-password -u admin -p "$AIRFLOW_ADMIN_PASSWORD"'
```

Après rotation Doppler :

```bash
kubectl -n airflow rollout restart deployment -l component=scheduler
kubectl -n airflow rollout restart deployment -l component=api-server
```

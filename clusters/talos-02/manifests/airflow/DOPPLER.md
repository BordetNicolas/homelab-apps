# Secrets Doppler pour Airflow (talos-02)

Projet : **`k8s-talos-pve1`** — config : **`prd`**

Créer ces variables dans [Doppler](https://dashboard.doppler.com) avant le premier sync ArgoCD de `airflow`.

| Variable Doppler | Clé Secret K8s | Génération suggérée |
|------------------|----------------|---------------------|
| `AIRFLOW_FERNET_KEY` | `fernet-key` | `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"` |
| `AIRFLOW_WEBSERVER_SECRET_KEY` | `webserver-secret-key` | `openssl rand -hex 32` |
| `AIRFLOW_POSTGRES_PASSWORD` | `postgres-password` | `openssl rand -base64 24` |
| `AIRFLOW_ADMIN_PASSWORD` | `admin-password` | mot de passe fort (UI Airflow, user `admin`) |

Les valeurs sont stockées en clair dans Doppler ; l’opérateur les place dans `Secret/airflow/airflow-credentials`.

Après ajout ou rotation : l’opérateur Doppler met à jour le Secret ; redémarrer scheduler/webserver si besoin :

```bash
kubectl -n airflow rollout restart deployment -l component=scheduler
kubectl -n airflow rollout restart deployment -l component=webserver
```

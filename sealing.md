# Sealing the secrets with kubeseal

## Get the latest release version of kubeseal
```bash
KUBESEAL_VERSION=$(curl --silent "https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
```

## Download the binary  
```bash
wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION#v}-linux-amd64.tar.gz"
```

## Extract the binary
```bash
tar -xvzf kubeseal-${KUBESEAL_VERSION#v}-linux-amd64.tar.gz kubeseal
```

## Move to a directory in your PATH
```bash
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

## Create a encrypted temporary secrets file (don't push it to the repository)
```bash
user=app_user
password=app_password
name=app_db

kubectl create secret generic database-credentials \
  --from-literal=database-user=$user \
  --from-literal=database-password=$password \
  --from-literal=database-name=$name \
  --from-literal=database-url=postgresql://$user:$password@database:5432/$name \
  --dry-run=client -o yaml > database-secret.yaml
```

## Seal the secrets
```bash
kubeseal --format yaml \
  --controller-namespace default \
  --controller-name sealed-secrets \
  < database-secret.yaml > templates/database-sealed-secret.yaml
```

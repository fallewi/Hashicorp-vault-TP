
# Dynamic Secrets with HashiCorp Vault and AWS

Welcome to the **Dynamic Secrets with HashiCorp Vault and AWS** project! This repository provides a detailed guide and commands to enable dynamic secrets generation in AWS using HashiCorp Vault, ensuring secure, time-limited access for your AWS resources.

## 📘 About This Tutorial

In this session, you'll learn:

1. **How to Enable the AWS Secret Engine** in HashiCorp Vault.
2. **Configure AWS Root Access** for generating dynamic secrets.
3. **Set Up Roles** in AWS to dynamically create access and secret keys.
4. **Generate Dynamic Secrets** using HashiCorp Vault.
5. **Revoke Dynamic Secrets** before their expiration, enhancing your security control.

Each section of the tutorial builds upon the previous, providing a concrete example of securely managing temporary AWS credentials.

## 📋 Prerequisites

To follow this guide, ensure you have:

- **AWS Access Key and Secret Key** for the root user account.
- **HashiCorp Vault** installed and configured on your system.
- Access to the terminal.

## 🚀 Getting Started

Follow these steps to generate dynamic AWS secrets with HashiCorp Vault:

### Step 1: Enable the AWS Secret Engine Path

To begin, enable the AWS secret engine in Vault:

```bash
vault secrets enable -path=aws aws
```

Verify that the secret engine has been enabled by listing the secrets:

```bash
vault secrets list
```

### Step 2: Configure AWS Root Access

Now, configure the root access in Vault with your AWS root credentials. Replace `YOUR_ACCESS_KEY`, `YOUR_SECRET_KEY`, and `YOUR_REGION` with your actual details.

```bash
vault write aws/config/root access_key=YOUR_ACCESS_KEY secret_key=YOUR_SECRET_KEY region=YOUR_REGION
```

### Step 3: Set Up a Role for Dynamic Secrets

Define a role, `my-ec2-role`, to allow specific actions on EC2 instances. This example grants all actions, but customize as per your security requirements.

```bash
vault write aws/roles/my-ec2-role credential_type=iam_user policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*"
    }
  ]
}
EOF
```

### Step 4: Generate the Dynamic Secret

Once the role is set, generate dynamic secrets for `my-ec2-role` using:

```bash
vault read aws/creds/my-ec2-role
```

This command will output a new AWS access key and secret key, along with their lease duration.

### Step 5: Revoke the Dynamic Secret

If you need to revoke these credentials before they expire, use the `vault lease revoke` command with the lease ID generated in Step 4:

```bash
vault lease revoke LEASE_ID
```

This instantly invalidates the dynamically generated credentials.

## 🛠 Commands Overview

| Command                                    | Description                                    |
|--------------------------------------------|------------------------------------------------|
| `vault secrets enable -path=aws aws`       | Enables AWS secret engine in Vault             |
| `vault write aws/config/root ...`          | Sets up AWS root configuration in Vault        |
| `vault write aws/roles/my-ec2-role ...`    | Creates a role in Vault with defined policies  |
| `vault read aws/creds/my-ec2-role`         | Generates dynamic AWS credentials              |
| `vault lease revoke LEASE_ID`              | Revokes the generated credentials by lease ID  |

## 📹 Video Tutorial

For a detailed walkthrough, check out the video tutorial in this repository or visit our YouTube channel [here](https://www.youtube.com/watch?v=zgC78QRPBfM&list=PL_OdF9Z6GmVZV62x9JI_pk31e2YdHyKuf&pp=iAQB) for the full step-by-step guide.

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a pull request.

## 📄 License

This project is licensed under the MIT License.

---
TP Vault Database Secrets Engine + MySQL + Docker (Rotation Dynamique)
Objectif

Déployer une base MySQL dans Docker

Activer le Database Secret Engine dans Vault

Créer des credentials dynamiques pour l’application

Gérer la rotation automatique des secrets

1️⃣ Prérequis

Docker & Docker Compose

Vault installé (binaire ou Docker)

MySQL client (mysql) optionnel pour tests

2️⃣ Structure du projet
vault-mysql-dynamic/
├─ docker-compose.yml
├─ setup_vault_mysql_dynamic.sh
├─ app/
│   ├─ Dockerfile
│   ├─ requirements.txt
│   └─ main.py

3️⃣ docker-compose.yml
version: "3.9"

services:
  vault:
    image: hashicorp/vault:1.16.2
    container_name: vault
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: root
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    ports:
      - "8200:8200"
    cap_add:
      - IPC_LOCK

  mysql:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: appdb
    ports:
      - "3306:3306"

  app:
    build: ./app
    depends_on:
      - vault
      - mysql
    environment:
      VAULT_ADDR: "http://host.docker.internal:8200"
      VAULT_ROLE_ID: "${VAULT_ROLE_ID}"
      VAULT_SECRET_ID: "${VAULT_SECRET_ID}"

4️⃣ setup_vault_mysql_dynamic.sh
#!/usr/bin/env bash
set -e

# Variables
export VAULT_ADDR=http://127.0.0.1:8200
VAULT_ROOT_TOKEN="root"
DB_ROLE_NAME="app-role"

# Lancer les conteneurs
docker compose up -d

echo "⏳ Attente 10s pour que Vault et MySQL soient prêts..."
sleep 10

# Login Vault
export VAULT_TOKEN=$VAULT_ROOT_TOKEN
vault login $VAULT_TOKEN >/dev/null

# Activer Database Secret Engine
vault secrets enable database || true

# Config MySQL
vault write database/config/mysql \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(mysql:3306)/" \
    allowed_roles="${DB_ROLE_NAME}" \
    username="root" \
    password="rootpass"

# Créer rôle pour rotation automatique
vault write database/roles/${DB_ROLE_NAME} \
    db_name=mysql \
    creation_statements="
      CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';
      GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO '{{name}}'@'%';
    " \
    default_ttl="1m" \
    max_ttl="5m"

# Activer AppRole
vault auth enable approle || true

# Créer AppRole
vault write auth/approle/role/python-app token_policies=default token_ttl=1h token_max_ttl=4h

# Récupérer ROLE_ID et SECRET_ID
ROLE_ID=$(vault read -field=role_id auth/approle/role/python-app/role-id)
SECRET_ID=$(vault write -field=secret_id -f auth/approle/role/python-app/secret-id)

export VAULT_ROLE_ID=$ROLE_ID
export VAULT_SECRET_ID=$SECRET_ID

echo ""
echo "✅ Vault et MySQL prêts"
echo "ROLE_ID=$VAULT_ROLE_ID"
echo "SECRET_ID=$VAULT_SECRET_ID"

# Lancer l'application Python
docker compose up --build app

5️⃣ Application Python
app/Dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

CMD ["python", "main.py"]

app/requirements.txt
hvac==2.3.0
mysql-connector-python==9.0.0

app/main.py
import os
import hvac
import mysql.connector

vault_addr = os.environ.get("VAULT_ADDR")
role_id = os.environ.get("VAULT_ROLE_ID")
secret_id = os.environ.get("VAULT_SECRET_ID")

client = hvac.Client(url=vault_addr)

# Authentification AppRole
client.auth.approle.login(role_id=role_id, secret_id=secret_id)

# Générer des credentials dynamiques MySQL
creds = client.secrets.database.generate_credentials(name="app-role")
username = creds['data']['username']
password = creds['data']['password']

print("✅ Credentials dynamiques générés :")
print(f"User: {username}, Password: {password}")

# Tester connexion MySQL
conn = mysql.connector.connect(
    host="mysql",
    user=username,
    password=password,
    database="appdb"
)
print("✅ Connexion MySQL réussie !")
conn.close()

6️⃣ Exécution du TP

Rendre le script exécutable :

chmod +x setup_vault_mysql_dynamic.sh


Lancer tout le TP :

./setup_vault_mysql_dynamic.sh


Vérifier que la rotation fonctionne :

vault read database/creds/app-role
# Attendre 1 min et relire
vault read database/creds/app-role


Chaque lecture crée un nouvel utilisateur temporaire MySQL

L’ancien utilisateur est révoqué automatiquement

7️⃣ Résumé du flux
Vault Database Secret Engine
        │
        ├─> Génération dynamique d'un user MySQL
        │
        ├─> TTL court (1 à 5 min)
        │
        ├─> Révocation automatique à l'expiration
        │
        └─> Application récupère credentials via AppRole


✅ Ce setup est idempotent, sécurisé et prêt pour test ou apprentissage.

By following these steps, you’ll learn to create and manage secure, temporary AWS credentials using HashiCorp Vault. Stay tuned for more tutorials on cloud security and dynamic secrets! 

+++
title = 'Microsoft SQL Server on GCP'
date = 2026-01-23T08:44:09-05:00
showtoc = true
draft = true
+++
## Microsoft SQL Server with Google Cloud SQL

  > :bulb: Disclaimer: This article reflects solely my opinion.

Generally speaking, if a vendor like Microsoft offers products or services,
including but not limited to to Windows Server and SQL Server, the integration
and ease of use with the same vendor's cloud, i.e., Azure, will be the best in terms of flexibility and
seamlessness.
But sometimes, the flexibility of installing another vendor's product in the Google Cloud Platform (GCP)
can be a good option. For example, when your application that is already running
on Google needs to connect to Microsoft SQL Server database close by.

Is running MS SQL in Google a good option? Let's find out.

Google offers the SQL Server editions: Express, Web, Standard, and Enterprise.
Each for the 2017, 2019, and 2022 releases of SQL Server.
For this exploration, we examine two scenarios:

- the replication conundrum, and
- the account management conundrum.

The above scenarios are worth further investigation as implementation details
matter when it comes to productionizing SQL Server in GCP.
How replication and disaster failover actually works matters in Production.
So does the account management and integration options, if any, with your
central Identity Platform (IdP).

## The Replication Conundrum

Cloud SQL for SQL Server offers replication between instances, which can be
considered a standard feature for Enterprise usage.
Replication enables well-established architecture patterns. For example,
running the primary SQL Server in a your primary GCP region (e.g., `northamerica-northeast2` - Toronto),
and a SQL Server read replica in a different GCP region e.g., `northamerica-northeast1` - Montréal. 
How exactly though is replication offered and what are the implications and limitations?
We will explore this with the following resources:

- Primary SQL Server
- Read replica SQL Server (different region)

{{< rawhtml >}}
<!--
mermaid.js
---
displayMode: compact
config:
  theme: neutral
  look: default
  themeVariables:
    fontSize: 20px
markdownAutoWrap: true
---
graph LR;
%%   subgraph "fa:fa-vault HashiCorp Vault"
%%     GCE(fa:fa-vault Vault)
%%   end
  subgraph "fa:fa-city Toronto"
    SQLPRI(fa:fa-database SQL Primary)
  end
  subgraph "fa:fa-tree-city" Montreal
     SQLREP(fa:fa-database SQL Replica)
  end
SQLPRI-\->|replication|SQLREP
-->
{{< /rawhtml >}}

[![Architecture of MS SQL Server replication in GCP](/images/mssql-on-gcp-primary-to-replica.png)](/images/mssql-on-gcp-primary-to-replica.png)

### Our Environment

:warning: It is for demonstration purposes only.

We need two Cloud SQL instances representing: (1) the primary MS SQL Server, and (2) the read replica.

We streamline the instance configuration by providing an instance startup script.
It installs packages, disables AppArmor, and enables IP forwarding at the OS-level.

### Primary SQL Server

An overview of the primary SQL Server.

- Name : `ms-sql-server-primary`
- SQL Edition: `SQL Server 2022 Enterprise`
- Cloud SQL Edition: `Enterprise Plus`
- Region: `northamerica-northeast2` Toronto

[![Primary SQL Server creation part one](/images/mssql-on-gcp-create-primary-1.png)](/images/mssql-on-gcp-create-primary-1.png)

[![Primary SQL Server creation part two](/images/mssql-on-gcp-create-primary-2.png)](/images/mssql-on-gcp-create-primary-2.png)

### Read replica SQL Server

- Name : `ms-sql-server-primary-replica`
- SQL Edition: `SQL Server 2022 Enterprise`
- Cloud SQL Edition: `Enterprise Plus`
- Region: `northamerica-northeast1` Montréal

  > :bulb: "Multiple zones (Highly available)" is grayed out and shows on mouse hover "Not supported for read replicas".

This is due to the fact that Cloud SQL uses [SQL Server Read Scale Availability Groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/read-scale-availability-groups?view=sql-server-linux-2017#read-scale-availability-groups-without-cluster).

[![Read replica SQL Server creation part one](/images/mssql-on-gcp-create-replica-1.png)](/images/mssql-on-gcp-create-replica-1.png)

[![Read replica SQL Server creation part one](/images/mssql-on-gcp-create-replica-2.png)](/images/mssql-on-gcp-create-replica-2.png)

Alternatively, you can set up Advanced Disaster Recovery that simplifies the recovery
and also the fallback if a region is affected.
Advanced DR requires the Cloud SQL Enterprise Edition and a [cascadable replica](https://docs.cloud.google.com/sql/docs/sqlserver/intro-to-cloud-sql-disaster-recovery#cascadable-replicas).
However, the [cascadable replica](https://docs.cloud.google.com/sql/docs/sqlserver/intro-to-cloud-sql-disaster-recovery#cascadable-replicas) is also zonal, even though
the Google documentation suggests differently (as of January 23, 2026).

Let's summarize the replication implications.

### Summary of Replication Conundrum

A Disaster Recovery (DR) SQL Server that can only be created as a zonal
replica is worth careful consideration.
The same applies to the Advanced DR setup.
While you can promote a replica to become primary and after that make it
highly available, it warrants careful planning and having the right tools
for the job ready.
A lot of companies use Terraform for provisioning; however, Terraform is
not a good tool when it comes to post-deployment configuration management.

## The Account Management Conundrum

Google offers what they can, which is GCP [Microsoft Managed AD](https://docs.cloud.google.com/sql/docs/sqlserver/ad)
and [Customer-managed Active Directory (CMAD)](https://docs.cloud.google.com/sql/docs/sqlserver/cmad), which is the customer's AD, e.g., running on-premises.

I would argue that both options are challenging for a lot of client since
they have severe technical limitations and implications.
Especially, if a client has strong regulatory and security requirements,
these options are likely a no-go.

My select and opinionated limitations of each integration:

- Microsoft Managed AD
  - necessary but unwanted overhead "just to use" SQL Server (No one likes overhead.)
  - requires a one-way AD trust from on-premises (No, thank you!)

- CMAD
  - No Private Service Connect (PSC) support
  - Open TCP and UDP ports `53`, `88`, `135`, `389`, `445`, `464`, `3268`, `3269`, and `49152` through `65535` from Cloud SQL instances to all domain controllers
    - No, thank you!

In addition, [Integration with Microsoft Entra ID](https://docs.cloud.google.com/sql/docs/sqlserver/integration-with-microsoft-entra-id) is in public preview, which typically means
that it will become generally available within six months.
That doesn't mean that it will support all configurations immediately when it becomes GA, e.g.,
PSC support may take longer.

Keeping things simple, we keep building on the further above deployed SQL Server instances.
Additionally, we need a GCE instance that will host the HashiCorp Vault server.
Vault will be used later for dynamic database credentials with its [database secrets engine](https://developer.hashicorp.com/vault/docs/secrets/databases).

{{< rawhtml >}}
<!--
---
title: Architecture of Vault and SQL Server integration
displayMode: compact
config:
  theme: redux
  look: default
  themeVariables:
    fontSize: 20px
markdownAutoWrap: true
---
graph LR;
  GCE(fa:fa-vault Vault)
  SQLPRI(fa:fa-database SQL Primary)
  GCE-\->|fas:fa-user fa:fa-key account|SQLPRI
-->
{{< /rawhtml >}}

[![Architecture of Vault and SQL Server integration](/images/mssql-on-gcp-vault-integration-1.png)](/images/mssql-on-gcp-vault-integration-1.png)

### Create GCE resources

We will create a startup script for our GCE instance, the GCE instance itself, and a firewall rule.

#### GCE startup script

Create the startup script `startup-script.sh` where you will run `gcloud compute instances create`.

```shell
#!/usr/bin/env bash
set -euo pipefail

VAULT_VERSION="1.15.4"
VAULT_ZIP="vault_${VAULT_VERSION}_linux_amd64.zip"
VAULT_URL="https://releases.hashicorp.com/vault/${VAULT_VERSION}/${VAULT_ZIP}"

VAULT_ADDR="http://127.0.0.1:8200"
VAULT_BIN="/usr/local/bin/vault"
VAULT_DATA_DIR="/opt/vault/data"
VAULT_CONFIG_DIR="/etc/vault.d"
VAULT_CONFIG_FILE="$VAULT_CONFIG_DIR/vault.hcl"
VAULT_USER="vault"

echo "=== Installing dependencies ==="
apt-get update -y
apt-get install -y curl unzip iputils-ping tmux vim

echo "=== Creating vault user ==="
if ! id "$VAULT_USER" >/dev/null 2>&1; then
  useradd --system --home /etc/vault.d --shell /bin/false "$VAULT_USER"
fi

echo "=== Downloading Vault ${VAULT_VERSION} ==="
curl -fsSL "$VAULT_URL" -o /tmp/vault.zip

echo "=== Installing Vault binary ==="
unzip -o /tmp/vault.zip -d /tmp
install -m 0755 /tmp/vault "$VAULT_BIN"
setcap cap_ipc_lock=+ep "$VAULT_BIN"

echo "=== Creating directories ==="
mkdir -p "$VAULT_DATA_DIR" "$VAULT_CONFIG_DIR"
chown -R "$VAULT_USER:$VAULT_USER" /opt/vault /etc/vault.d
chmod 750 "$VAULT_DATA_DIR"


if [ -s "/etc/systemd/system/vault.service" ]; then
  echo "Vault config exists already."
else
  echo "=== Writing Vault config ==="
cat > "$VAULT_CONFIG_FILE" <<EOF
storage "file" {
  path = "$VAULT_DATA_DIR"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

ui = true
EOF
fi

chown "$VAULT_USER:$VAULT_USER" "$VAULT_CONFIG_FILE"
chmod 640 "$VAULT_CONFIG_FILE"

if [ -s "/etc/systemd/system/vault.service" ]; then
  echo "vault.service is not empty (has a size greater than zero)."
else
echo "=== Creating systemd service ==="
cat > /etc/systemd/system/vault.service <<EOF
[Unit]
Description=HashiCorp Vault
Documentation=https://www.vaultproject.io/docs
Requires=network-online.target
After=network-online.target

[Service]
User=$VAULT_USER
Group=$VAULT_USER
Environment="VAULT_LOG_LEVEL=debug"
ExecStart=$VAULT_BIN server -config=$VAULT_CONFIG_FILE
ExecReload=/bin/kill --signal HUP \$MAINPID
KillMode=process
Restart=on-failure
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
EOF
fi

echo "=== Starting Vault ==="
systemctl daemon-reexec
systemctl daemon-reload
systemctl enable vault
systemctl start vault

export VAULT_ADDR="$VAULT_ADDR"

echo "=== Waiting for Vault to start ==="
sleep 5

INIT_FILE="/root/vault-init.txt"
if [ -s "$INIT_FILE" ]; then
  echo === Vault initialized already ===
  echo "$INIT_FILE is not empty (has a size greater than zero)."
else
  echo "=== Initializing Vault ==="
  vault operator init -key-shares=1 -key-threshold=1 > $INIT_FILE
  cp $INIT_FILE /root/vault-init.bak
fi

UNSEAL_KEY=$(awk '/Unseal Key 1:/ {print $4}' $INIT_FILE)
ROOT_TOKEN=$(awk '/Initial Root Token:/ {print $4}' $INIT_FILE)

echo "Unseal Key: $UNSEAL_KEY"
echo "Root Token: $ROOT_TOKEN"

echo "=== Unsealing Vault ==="
vault operator unseal "$UNSEAL_KEY"

echo "=== Logging in ==="
vault login "$ROOT_TOKEN"

echo "=== Enabling database secrets engine ==="
vault secrets enable database

echo "=== Vault status ==="
vault status

echo "=== Done ==="
echo "Initialization details saved to $INIT_FILE"
```

#### GCE Instance Config

- Machine: `e2-small`
- Region: `northamerica-northeast2`
- OS: Ubuntu 25.10 minimal
- Disk: `10` Gbyte
- Cloud API access scopes: `cloud-platform`
```shell
gcloud compute instances create \
--machine-type=e2-small \
--image-family=ubuntu-minimal-2510-amd64 \
--image-project=ubuntu-os-cloud \
--scopes=cloud-platform \
--can-ip-forward \
--tags=vault \
--metadata-from-file=startup-script=./startup-script.sh \
--async \
vault
```

#### GCP VPC Firewall rule

The firewall rule allows TCP traffic to our port `8200` to our `vault`
GCE instance tagged with `vault`.

```shell
# replace $CLOUDSDK_CORE_PROJECT with your GCP project or set the variable
gcloud compute --project=$CLOUDSDK_CORE_PROJECT \
firewall-rules create allow-vault \
--direction=INGRESS --priority=1000 --network=default --action=ALLOW \
--rules=tcp:8200 --source-ranges=99.225.69.206/32 --target-tags=vault
```

### Vault Prerequisites

First, we grant Vault the required permissions on the MS SQL instance.
Then, we configure the Vault database secrets engine and policies.

#### Vault User in MS SQL

Connect to your MS SQL instance with Cloud SQL Studio.
Create a database called `acme`.

```sql
-- Create Login
CREATE LOGIN vault_login WITH PASSWORD = 'yourStrong123Password#';

-- Create User works in acme, fails in master
CREATE user vault_user for login vault_login;

-- Grant Permissions, fails in acme, fails in master
-- GRANT ALTER ANY LOGIN TO "vault_user";

-- works in master, but was supposed to be vault_user, not vault_login
GRANT ALTER ANY LOGIN TO "vault_login" AS CustomerDbRootRole;

-- works in acme
GRANT ALTER ANY USER TO "vault_user";

-- fails in acme, fails in master with Grantor does not have GRANT permission.
-- GRANT ALTER ANY CONNECTION TO "vault_login";
-- works in master
GRANT ALTER ANY CONNECTION TO "vault_login" AS CustomerDbRootRole;

-- works in acme
GRANT CONTROL ON SCHEMA::dbo TO "vault_user";
-- works in acme
EXEC sp_addrolemember "db_accessadmin", "vault_user";

-- master
ALTER SERVER ROLE CustomerDbRootRole ADD MEMBER [vault_login]

-- acme 
ALTER ROLE [db_accessadmin] ADD MEMBER [vault_user];
EXEC sp_addrolemember "db_accessadmin", "vault_user";
ALTER ROLE [db_securityadmin] ADD MEMBER [vault_user];
```

#### Prepare Vault

Connect to the GCE instance `vault`.

```shell
# replace $CLOUDSDK_CORE_PROJECT with your GCP project or set the variable
gcloud compute ssh \
  --tunnel-through-iap --project $CLOUDSDK_CORE_PROJECT \
  vault
```

Configure the Vault MS SQL plugin.

```shell
vault write database/config/my-mssql-database \
    plugin_name=mssql-database-plugin \
    connection_url='sqlserver://{{username}}:{{password}}@10.122.144.6:1433' \
    allowed_roles="my-role" \
    username="vault_login" \
    password="yourStrong123Password#"

Success! Data written to: database/config/my-mssql-database
```

Define a role that maps its name to a SQL statement for database credential creation.

```shell
vault write database/roles/my-role \
    db_name=my-mssql-database \
    creation_statements="CREATE LOGIN [{{name}}] WITH PASSWORD = '{{password}}';\
        USE acme CREATE USER [{{name}}] FOR LOGIN [{{name}}];\
        GRANT SELECT ON SCHEMA::dbo TO [{{name}}];" \
    revocation_statements="USE acme DROP USER IF EXISTS [{{name}}];\
        USE master DROP LOGIN [{{name}}];" \
    default_ttl="30m" \
    max_ttl="24h"

Success! Data written to: database/roles/my-role
```

Generate a dynamic database credential by reading from the new Vault role.

```shell
vault read database/creds/my-role

Key                Value
---                -----
lease_id           database/creds/my-role/9jFQz0AV1UPAKYBm7I5AzL90
lease_duration     30m
lease_renewable    true
password           7MBK6OguP24xMf4-OVJM
username           v-root-my-role-FEWE2ejlHFCalfLd9SNC-1769437222
```

The account above was created in our Cloud SQL instance.
If it isn't actively renewed, it expires after the specified time-to-live (TTL) parameter `default_ttl` and will be dropped from the database.

## Summary

The feature set of SQL Server on Google Cloud SQL has important implications.

While you can technically integrate with Active Directory, the current options
have big architecture implications. Either by running an additional Google service
called Microsoft Managed AD, which then establishes a one-way trust relationship
with your existing Active Directory, or, by opening a whole bunch of TCP and UDP
ports on all of your AD Domain Controllers from all your Google SQL Server instances.

The upcoming, in public preview, integration with Entra ID may solve some of these
implications. It remains to be seen when the Entra ID integration would also work
for Cloud SQL instances provisioned with Private Service Connect as their only
connectivity option. 

Similarly, and dependent on the above, a central account management in Google SQL
Server can become more involved in comparison to other services that can directly
integrate with Google IAM. When the integration with AD is not an option, e.g., for
security, regulatory, or architectural reasons, the alternative can be the database
secrets engine from HashiCorp Vault. This Vault integration allows for dynamic database
credential provisioning and revocation.

Vault is a powerful holistic secrets management solution that might already be
integrated with your Identity Platform and by proxy enable a centralized account
management.

A lot to think through and plan. Feel free to explore yourself and revisit the
latest feature set as things change fast.

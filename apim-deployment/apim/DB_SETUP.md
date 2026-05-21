# WSO2 API Manager — Database Setup Guide

WSO2 APIM requires two MySQL databases before the Helm deployment:

| Database | Purpose | Scripts |
|---|---|---|
| `wso2amdb` | APIM registry, API metadata, subscriptions | `db-scripts/apimgt/mysql.sql` |
| `wso2shareddb` | User management, shared governance | `db-scripts/shareddb/mysql.sql` |

A MySQL 8.x instance reachable from within the cluster is required.  
Options: AWS RDS, in-cluster pod (see below), or any accessible MySQL server.

The `values.yaml` expects the database at `mysql:3306` — this matches the in-cluster  
Kubernetes Service name used in the manifest below.

---

## Option A: In-Cluster MySQL (Dev/Demo)

### 1. Deploy MySQL

```bash
oc apply -f mysql.yaml -n jerry-juma-dev

# Wait for the pod to be Ready
oc get pods -n jerry-juma-dev -l app=mysql -w
```

### 2. Create the Databases

```bash
# Open a MySQL shell inside the pod
oc exec -it deployment/mysql -n jerry-juma-dev -- \
  mysql -u root -prootpass123
```

Inside the MySQL shell:

```sql
CREATE DATABASE wso2amdb   CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE wso2shareddb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

GRANT ALL ON wso2amdb.*     TO 'wso2'@'%';
GRANT ALL ON wso2shareddb.* TO 'wso2'@'%';
FLUSH PRIVILEGES;

EXIT;
```

### 3. Run the DB Scripts

Copy the scripts into the pod and execute them:

```bash
# APIM database
oc cp db-scripts/apimgt/mysql.sql \
  deployment/mysql:/tmp/apimgt.sql -n jerry-juma-dev

oc exec -it deployment/mysql -n jerry-juma-dev -- \
  mysql -u root -prootpass123 wso2amdb -e "source /tmp/apimgt.sql"

# Shared database
oc cp db-scripts/shareddb/mysql.sql \
  deployment/mysql:/tmp/shareddb.sql -n jerry-juma-dev

oc exec -it deployment/mysql -n jerry-juma-dev -- \
  mysql -u root -prootpass123 wso2shareddb -e "source /tmp/shareddb.sql"
```

### 4. Verify

```bash
oc exec -it deployment/mysql -n jerry-juma-dev -- \
  mysql -u wso2 -pwso2pass123 -e "SHOW DATABASES;"
```

Expected output should include both `wso2amdb` and `wso2shareddb`.

---

## Option B: External MySQL (Production)

For production use AWS RDS MySQL 8.x or an equivalent managed service.

1. Create the two databases and the `wso2` user with the same grants shown above.
2. Run the scripts from your local machine:

```bash
mysql -h <rds-endpoint> -u root -p wso2amdb    < db-scripts/apimgt/mysql.sql
mysql -h <rds-endpoint> -u root -p wso2shareddb < db-scripts/shareddb/mysql.sql
```

3. Update `values.yaml` with the RDS endpoint:

```yaml
databases:
  apim_db:
    url: "jdbc:mysql://<rds-endpoint>:3306/wso2amdb?useSSL=true&serverTimezone=UTC"
    username: "wso2"
    password: "<password>"
  shared_db:
    url: "jdbc:mysql://<rds-endpoint>:3306/wso2shareddb?useSSL=true&serverTimezone=UTC"
    username: "wso2"
    password: "<password>"
```

---

## Credentials Reference

> **Change these before any production deployment.**

| Setting | Dev Default |
|---|---|
| MySQL root password | `rootpass123` |
| WSO2 DB user | `wso2` |
| WSO2 DB password | `wso2pass123` |

These match the values already set in `values.yaml` under `wso2.apim.configurations.databases`.
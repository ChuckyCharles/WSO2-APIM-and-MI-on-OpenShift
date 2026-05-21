# WSO2 APIM and MI on OpenShift (ROSA)

This repository contains the Helm charts, Docker customizations, and supporting assets for deploying WSO2 API Manager 4.5 and WSO2 Micro Integrator 4.x on Red Hat OpenShift Service on AWS (ROSA).

---

## Repository Structure

```
WSO2-APIM-and-MI-on-OpenShift/
├── apim-deployment/               # API Manager deployment
│   ├── apim/                      # Docker customization and database setup
│   │   ├── Dockerfile             # Custom APIM image (MySQL driver + OpenShift SCC fixes)
│   │   ├── DB_SETUP.md            # Database setup guide
│   │   ├── mysql.yaml             # In-cluster MySQL manifest (dev/demo)
│   │   ├── db-scripts/            # Schema scripts for wso2amdb and wso2shareddb
│   │   ├── keystores/             # Source keystore files
│   │   └── lib/                   # MySQL JDBC driver
│   └── helm-apim/                 # Helm charts
│       ├── all-in-one/            # All-in-one deployment chart
│       ├── distributed/           # Distributed deployment charts
│       │   ├── control-plane/
│       │   ├── gateway/
│       │   ├── key-manager/
│       │   └── traffic-manager/
│       └── docs/                  # Deployment pattern reference configs
│
└── mi-deployment/                 # Micro Integrator deployment
    ├── agency-intg/               # Integration project source
    │   ├── Dockerfile             # Custom MI image (baked .car + OpenShift SCC fixes)
    │   ├── carbonapps/            # Compiled .car files
    │   ├── conf/                  # deployment.toml, log4j2, file.properties
    │   └── libs/                  # Custom JARs
    └── helm-mi/
        └── mi/                    # Helm chart for WSO2 MI
            ├── carbonapps/        # .car files (dev/test)
            ├── confs/             # deployment.toml, log4j2.properties
            ├── security/          # Keystore files (not committed)
            ├── templates/         # Kubernetes manifest templates
            ├── mi-pvcs.yaml       # Manual PVC definition (EFS)
            ├── mi-init-pod.yaml   # Init pod for EFS volume seeding
            └── values.yaml        # Main configuration (not committed)
```

---

## Architecture

Both products run as containerized workloads on ROSA, exposed via OpenShift Routes (APIM) and NGINX Ingress (MI). The APIM gateway proxies traffic to MI APIs registered through the APIM Publisher.

```
Internet
   │
   ▼
OpenShift Routes / NGINX Ingress
   │                    │
   ▼                    ▼
WSO2 APIM          WSO2 MI Pod
(Publisher,        (CarbonApps
 DevPortal,         via EFS PVC
 Gateway)           or baked image)
   │                    │
   └────────────────────┘
     APIM Gateway proxies
     to MI backend service
          │
          ▼
     MySQL (wso2amdb
     + wso2shareddb)
```

---

## Deployments

### API Manager

APIM runs in an all-in-one topology (Publisher, DevPortal, Gateway, and Key Manager in a single pod). It requires two pre-populated MySQL databases and a Kubernetes secret containing the JKS keystores.

See **[`apim-deployment/apim/DB_SETUP.md`](apim-deployment/apim/DB_SETUP.md)** for database setup and **[`apim-deployment/helm-apim/README.md`](apim-deployment/helm-apim/README.md)** for the full deployment guide including:

- Building and pushing the custom Docker image
- Setting up MySQL databases
- Creating the keystore secret
- Installing the Helm chart
- Exposing a backend MI API through the Publisher

### Micro Integrator

MI runs as a stateless pod with CarbonApps (`.car` files) either baked into the Docker image (dev) or mounted from an AWS EFS volume (production). It integrates with APIM as a backend gateway environment.

See **[`mi-deployment/helm-mi/README.md`](mi-deployment/helm-mi/README.md)** for the full deployment guide including:

- EFS provisioning and PVC setup
- Keystore secret creation
- Helm chart configuration and deployment
- CarbonApp deployment strategies (baked image vs EFS volume mount)

---

## Prerequisites

| Requirement | Notes |
|---|---|
| ROSA cluster | `oc` CLI authenticated |
| Helm 3.x | `helm version` to verify |
| Docker / Podman | For building custom images |
| MySQL 8.x | In-cluster pod or AWS RDS |
| AWS EFS | Required for MI production CarbonApp mounts |
| EFS CSI driver | `efs-sc` StorageClass must be present |
| Container registry | Docker Hub or private (ACR, ECR, etc.) |

---

## Quick Reference

| Task | Command |
|---|---|
| Deploy APIM | `helm upgrade --install apim apim-deployment/helm-apim/all-in-one/ -f values.yaml -n <ns>` |
| Deploy MI | `helm upgrade --install mi mi-deployment/helm-mi/mi/ -f values.yaml -n <ns>` |
| Watch pods | `oc get pods -n <ns> -w` |
| APIM Publisher | `https://<management-hostname>/publisher` |
| APIM DevPortal | `https://<management-hostname>/devportal` |
| MI health check | `curl -k https://<mi-hostname>/healthz` |

---

## Related Documentation

- [WSO2 API Manager Docs](https://apim.docs.wso2.com/en/latest/)
- [WSO2 Micro Integrator Docs](https://mi.docs.wso2.com/en/latest/)
- [ROSA Documentation](https://docs.openshift.com/rosa/welcome/index.html)
- [AWS EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver)
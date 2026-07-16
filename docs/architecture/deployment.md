# OpenShift Deployment

## Deployment Architecture on Red Hat OpenShift

All platform components run as containerised workloads inside the OpenShift cluster. No external SaaS dependencies — full on-premises deployment.

---

## Namespace Strategy

```mermaid
graph TD
    subgraph OCP["Red Hat OpenShift Cluster"]
        subgraph INFRA_NS["Namespace: data-catalog-infra"]
            PG["PostgreSQL\nStatefulSet x3\n+ PVC"]
            CASSANDRA["DataStax Cassandra\nStatefulSet x3\n+ PVC"]
            ES["Elasticsearch\nStatefulSet x3\n+ PVC"]
        end

        subgraph CATALOG_NS["Namespace: data-catalog-core"]
            IKC["IBM Knowledge Catalog\nCP4D Operator\nDeployment x3"]
            WXD["watsonx.data\nIBM Operator\nDeployment x2"]
            ASTRA_DB["DataStax AstraDB\nStatefulSet x3"]
            ASTRA_STR["DataStax Astra Streaming\nStatefulSet x3"]
        end

        subgraph INGEST_NS["Namespace: data-catalog-ingest"]
            NIFI["Apache NiFi Cluster\nStatefulSet x10\n+ ZooKeeper"]
            AIRFLOW["Astronomer Airflow\nDeployment\nScheduler + Workers"]
            DS["IBM DataStage\nCP4D Operator\nDeployment x2"]
        end

        subgraph AI_NS["Namespace: data-catalog-ai"]
            WX_AI["IBM watsonx.ai\nInference Service\nDeployment x2"]
            LANGFLOW["DataStax Langflow\nDeployment x2"]
        end

        subgraph TEAM_NS["Per-Team Namespaces"]
            NS_A["genomics-team-a\nRBAC isolated"]
            NS_B["research-team-b\nRBAC isolated"]
            NS_C["shared-generalist\nRead-only"]
        end

        subgraph NET_NS["Namespace: data-catalog-network"]
            ROUTE["OpenShift Routes\nTLS Termination\nIngress"]
            SVC["ClusterIP Services\nInternal mesh"]
        end
    end
```

---

## Component Pod Topology

| Component | Workload Type | Replicas | Storage | Notes |
|---|---|---|---|---|
| **IBM Knowledge Catalog** | CP4D Deployment | 3 | PVC via CP4D operator | Managed by CP4D operator |
| **watsonx.data** | IBM Operator Deployment | 2 | PVC (Scale-backed) | Managed by watsonx.data operator |
| **Apache NiFi** | StatefulSet | 10 | PVC per node | ZooKeeper for cluster state |
| **IBM DataStage** | CP4D Deployment | 2 | PVC via CP4D operator | Managed by CP4D operator |
| **watsonx.ai** | KServe InferenceService | 2 | Model store PVC | GPU nodes if available |
| **DataStax Langflow** | Deployment | 2 | Stateless + AstraDB backend | Horizontal scale |
| **DataStax AstraDB** | StatefulSet | 3 | PVC (Scale-backed) | Cassandra-based vector store |
| **DataStax Astra Streaming** | StatefulSet | 3 | PVC (Scale-backed) | Pulsar-based streaming |
| **Astronomer Airflow** | Deployment | 1 scheduler + N workers | PVC for DAG store | Astronomer Software operator |
| **PostgreSQL** | StatefulSet | 3 | PVC (Scale-backed) | Primary + 2 replicas |
| **Elasticsearch** | StatefulSet | 3 | PVC (Scale-backed) | IKC search backend |

---

## Storage Class Design

IBM Storage Scale serves as the backing PersistentVolume provider for all stateful workloads:

```mermaid
flowchart LR
    subgraph SC["OpenShift Storage Classes"]
        SC_FAST["StorageClass: scale-fast\nGPFS SSD tier\nfor DBs and indexes"]
        SC_STD["StorageClass: scale-standard\nGPFS HDD tier\nfor app data"]
        SC_TAPE["StorageClass: scale-hsm\nGPFS HSM tier\nfor archive PVCs"]
    end

    subgraph PVC["PersistentVolumeClaims"]
        PVC_PG["PostgreSQL PVC\nscale-fast"]
        PVC_ES["Elasticsearch PVC\nscale-fast"]
        PVC_ASTRA["AstraDB PVC\nscale-fast"]
        PVC_NIFI["NiFi PVC\nscale-standard"]
        PVC_DS["DataStage PVC\nscale-standard"]
    end

    SS["IBM Storage Scale\nGPFS Cluster"] --> SC_FAST & SC_STD & SC_TAPE
    SC_FAST --> PVC_PG & PVC_ES & PVC_ASTRA
    SC_STD --> PVC_NIFI & PVC_DS
```

---

## Networking & Ingress

| Route | Target Service | TLS | Auth |
|---|---|---|---|
| `catalog.internal.org` | IBM Knowledge Catalog UI | ✅ TLS terminate | IBM IAM / LDAP |
| `api.catalog.internal.org` | IKC REST API | ✅ TLS terminate | OAuth2 / API Key |
| `nifi.internal.org` | NiFi UI | ✅ TLS terminate | OIDC |
| `airflow.internal.org` | Airflow UI | ✅ TLS terminate | OIDC |
| `langflow.internal.org` | Langflow NL UI | ✅ TLS terminate | IBM IAM |

---

## Security Architecture

```mermaid
flowchart TD
    subgraph SEC["Security Controls"]
        IAM["IBM IAM\nCentral identity provider\nSSO via OIDC/LDAP"]
        VAULT["HashiCorp Vault\nor OpenShift Secrets\nCredential management"]
        CERTMGR["cert-manager\nAutomatic TLS certificates\nLet's Encrypt / Internal CA"]
        RBAC["OpenShift RBAC\nNamespace-level isolation\nRole bindings per team"]
        NET_POL["NetworkPolicies\nDeny-all default\nExplicit allow rules"]
    end

    IAM --> IKC_AUTH["IKC Authentication"]
    IAM --> AIRFLOW_AUTH["Airflow Authentication"]
    IAM --> NIFI_AUTH["NiFi Authentication"]
    VAULT --> NIFI_CREDS["NiFi — Storage Scale credentials"]
    VAULT --> DS_CREDS["DataStage — DB credentials"]
    CERTMGR --> ROUTES["All OpenShift Routes"]
    RBAC --> NS_ISOLATION["Team namespace isolation"]
    NET_POL --> SVC_MESH["Inter-service communication control"]
```

---

## Operators Used

| Operator | Source | Manages |
|---|---|---|
| **IBM Cloud Pak for Data Operator** | IBM Operator Catalog | IKC, DataStage, watsonx.ai |
| **IBM watsonx.data Operator** | IBM Operator Catalog | watsonx.data, Hive Metastore |
| **Astronomer Operator** | Astronomer Software | Airflow deployment |
| **cert-manager Operator** | OperatorHub | TLS certificate lifecycle |
| **Crunchy Postgres Operator** | OperatorHub | PostgreSQL StatefulSet |
| **ECK Operator** | Elastic | Elasticsearch StatefulSet |

---

## Resource Estimates

| Namespace | CPU Request | Memory Request | Storage |
|---|---|---|---|
| `data-catalog-infra` | 48 cores | 192 GB | 20 TB PVC |
| `data-catalog-core` | 64 cores | 256 GB | 10 TB PVC |
| `data-catalog-ingest` | 80 cores | 320 GB | 5 TB PVC |
| `data-catalog-ai` | 32 cores + GPU | 128 GB | 2 TB PVC |
| **Total estimate** | **~224 cores** | **~896 GB** | **~37 TB PVC** |

!!! info "Sizing Note"
    These are initial estimates for 20PB at steady state. The NiFi cluster can scale down after the initial crawl is complete. GPU nodes are optional — watsonx.ai can run on CPU with reduced throughput.

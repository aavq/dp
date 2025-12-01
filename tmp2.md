
Да, похоже, GitHub-овский Mermaid очень капризный. Давай сделаем максимально простой и совместимый вариант: без скобок, двоеточий и HTML.

### 1) Flowchart: single-cluster + multi-cluster

```mermaid
graph LR
    client["Client"]
    lb["External L4 Load Balancer"]

    subgraph clusterA["Cluster A"]
        direction LR
        igw["Istio IngressGateway (ns sm-gateways)"]
        svcA["Service A pod with sidecar"]
        svcB["Service B pod with sidecar"]
        egw["Istio EgressGateway (ns sm-gateways)"]
        ewA["Istio EastWestGateway A"]
    end

    subgraph clusterB["Cluster B"]
        direction LR
        svcB2["Service in Cluster B with sidecar"]
        ewB["Istio EastWestGateway B"]
    end

    ext["External destination SaaS DB API"]

    %% single cluster
    client -->|1| lb
    lb -->|2| igw
    igw -->|3 VirtualService + DestinationRule| svcA
    svcA -->|4 mTLS| svcB
    svcA -->|5 egress policy| egw
    egw --> ext

    %% multi cluster east-west
    svcA -->|6| ewA
    ewA --> ewB
    ewB --> svcB2
```

---

### 2) Sequence diagram по шагам 1–6

```mermaid
sequenceDiagram
    participant C as Client
    participant LB as External L4 LB
    participant IGW as istio-ingressgateway
    participant SA as Service A sidecar
    participant SB as Service B sidecar
    participant EGW as istio-egressgateway
    participant EW_A as eastwestgateway A
    participant EW_B as eastwestgateway B
    participant SB_B as Service in Cluster B

    C->>LB: 1) Send request
    LB->>IGW: 2) Forward to ingress Service
    IGW->>SA: 3) Route via VirtualService and DestinationRule
    SA->>SB: 4) In-mesh call with mTLS
    SA->>EGW: 5) Outbound via egressgateway
    SA->>EW_A: 6) Cross-cluster call
    EW_A->>EW_B: Forward to remote cluster
    EW_B->>SB_B: Deliver to service in Cluster B
```

Если всё ещё будут ошибки, попробуй сначала только `graph LR` кусок без `subgraph` и комментариев — но этот вариант обычно нормально рендерится на GitHub.

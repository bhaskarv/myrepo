**Cloud NAT** (Network Address Translation) is the third piece of the puzzle. While PGA and PSC are for talking to **Google**, Cloud NAT is for talking to the **rest of the internet** (like GitHub, Linux package repositories, or 3rd-party APIs) without giving your VMs a public IP.

  ### Why use Cloud NAT?
  
  1. **Security:** You get internet access without exposing your VM to random scans or "brute force" attacks from the web.
  2. **Software Updates:** Most private workloads still need to download patches or libraries from public mirrors. Cloud NAT makes this possible.
  3. **Static IP Whitelisting:** If you are connecting to a client's database that requires you to "whitelist" your IP, you can assign a **Fixed Static IP** to your Cloud NAT gateway. All your private VMs will then appear to the outside world as coming from that one specific IP.
  
  ---
  
  ### Comparison: When to use what?
  
  | Feature | Private Google Access (PGA) | Private Service Connect (PSC) | Cloud NAT |
  | --- | --- | --- | --- |
  | **Primary Goal** | Access Google APIs. | Access Google APIs via a private IP. | Access the **Public Internet**. |
  | **Destination** | `storage.googleapis.com`, etc. | `10.x.x.x` (Your internal IP). | `github.com`, `nytimes.com`, etc. |
  | **Internet Path** | Traffic stays on Google Network. | Traffic stays on Google Network. | Traffic goes to the Public Web. |
  | **Cost** | **Free** (No gateway charge). | Monthly charge per endpoint. | Hourly charge + per GB processing. |

  ## PCA Comparison Snapshot
  
  | Capability                   | **PSC** | **PGA**    |
  | ---------------------------- | ------- | -----------|
  | Access Google APIs privately | ✅       | ✅       |
  | Expose custom services       | ✅       | ❌       |
  | Cross-VPC / cross-project    | ✅       | ❌       |
  | Service-level isolation      | ✅       | ❌       |
  | Route sharing                | ❌       | ❌       |
  | Network connectivity         | ❌       | ❌       |
  | Outbound internet access     | ❌       | ❌       |

---

## One-Line Exam Memory Hook

> **PGA lets you *call* Google APIs.
> PSC lets you *publish* and *consume* private services.**
  
  ### The "All-in-One" Scenario
  
  Most secure architectures use a combination:
  
  * **PGA** is turned on for the subnet (it's free and easy).
  * **PSC** is used for critical database connections (like Cloud SQL).
  * **Cloud NAT** is set up so the VMs can download security patches once a week.

> VMs within same VPC can communicate with each other using private IP regardless of them being in different subnets. 
---
* From a PCA exam perspective: Private Google Access (PGA) allows VMs without external IPs to reach Google APIs and services (like Cloud Storage or BigQuery) over Google’s network, but does not provide general internet access. Private Service Connect (PSC) provides private, VPC-internal access to Google-managed services or published services using internal IPs, enabling producer–consumer isolation without VPC peering or public endpoints, and is the preferred modern pattern for private service access. Cloud NAT enables outbound internet access for private resources (VMs, GKE nodes) without external IPs, supports automatic scaling and high availability, but does not allow inbound connections. In short: PGA = private access to Google APIs, PSC = private access to services via internal IPs, Cloud NAT = outbound internet access only.

## Statefulness [MIGs & GKE Pods]
* Stateful MIGs support auto-healing to recover failed VMs without data loss.
* Autoscaling is disabled when the MIG has stateful configuration (stateful disks, per-instance configs)
  - If a workload requires autoscaling, it must be stateless.
* GKE StatefulSets: StatefulSets support automatic Pod recovery while preserving persistent storage.
* With stateufulsets, GKE auto scaling is very limited though HPA can target a statefulset, new replicas get new empty volumes. Data replicaion is application responsibility. "Autoscaling StatefulSets is possible, but only appropriate when the application handles replication."
* From a PCA exam perspective: auto-healing is supported for both stateful and stateless workloads, but autoscaling requires statelessness. Stateful Managed Instance Groups support auto-healing via health checks that recreate failed VMs and reattach the same persistent disks, but autoscaling is not supported because instances are not interchangeable. Stateless MIGs support both auto-healing and autoscaling and are the correct choice for horizontally scalable VM workloads. In GKE, StatefulSets support auto-healing at the Pod level with PersistentVolumeClaims preserved, and autoscaling is technically allowed via HPA, but new replicas receive new empty volumes, so data replication and consistency are the application’s responsibility. For the exam, remember: GCP handles infrastructure recovery, but state management and replication are always the application’s job.

---
## Network Design
| Feature | Scope | Purpose |
| --- | --- | --- |
| **VPC** | Global | Provides a single, private network container worldwide. |
| **Subnet** | Regional | Groups resources geographically and defines IP ranges. |
| **Shared VPC** | Organizational | Connects multiple projects to a centralized network. |

> A Shared VPC is strictly an intra-organization feature.

* Google Cloud networking is built on a Global VPC, where a single private network spans all regions using Google’s private backbone, while subnets remain regional to support zonal isolation and high availability. For large enterprises, Shared VPC centralizes network control in a host project and allows multiple service projects to deploy resources, but it is limited to a single organization; cross-organization connectivity requires VPC Peering, which enables private IP communication but forbids overlapping CIDR ranges, often driving designs toward Private Service Connect or NAT-based solutions. To extend connectivity to on-premises environments, Google Cloud offers Cloud VPN and Cloud Interconnect: Cloud VPN provides encrypted, internet-based tunnels ideal for rapid setup or small workloads, while Cloud Interconnect delivers private, high-bandwidth, low-latency connectivity for enterprise-scale traffic. Dedicated Interconnect is chosen when an organization requires direct physical connections at 10 or 100 Gbps, maximum throughput, and predictable performance from a colocation facility, whereas Partner Interconnect is used when direct colocation is not feasible and flexible bandwidth options are needed via a service provider. For mission-critical hybrid architectures, HA VPN is commonly layered on top of Cloud Interconnect to add encryption and redundancy, combining Interconnect’s performance with VPN’s security to meet strict availability and compliance requirements.
---
### PCA Scenario Mapping (Quick Recall)

* **“Enterprise data center, massive throughput, lowest latency”** → Dedicated Interconnect
* **“No colocation, need private connectivity with flexible speeds”** → Partner Interconnect
* **“Fast setup, encrypted, low-to-medium traffic”** → Cloud VPN
* **“Regulatory or security requirement + Interconnect”** → HA VPN over Interconnect
* **“99.99% hybrid availability requirement”** → Dual Interconnects + HA VPN

---

### One-line memory hook

> **VPN = speed of setup, Interconnect = speed of data, HA VPN + Interconnect = best of both.**


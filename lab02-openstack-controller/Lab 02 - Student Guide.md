# Lab 02 - Under the IaaS Hood: Control Plane Microservices and Podman Auditing

* **Program:** MBA in Cloud Computing & Infrastructure
* **Environment / Platform:** Red Hat Academy (CL110 / Red Hat OpenStack Platform)
* **Technical Stack:** Red Hat Enterprise Linux 8, Podman Engine, Systemd, TripleO Controller
* **Estimated Duration:** 30 to 35 minutes

---

## 🎯 Lab Objective

Demystify the internal architecture of a modern enterprise IaaS control plane by examining how infrastructure management services (Compute, SDN Networking, and Identity APIs) are containerized to ensure high availability, fast updates, and failure isolation.

**Enterprise Business Scenario (SRE Cloud Audit):**  
Prior to approving a newly deployed private cloud cluster for business workloads, the SRE and Platform Engineering team must perform a health and resilience audit on the master controller node (`controller0`). Your mission is to gain administrative access to the control plane, inspect the containerized microservices managed by Podman, and verify API log streams for Nova and Neutron.

**Key Skills Acquired:**
1. Establish secure, key-based SSH administrative connections to Overcloud nodes.
2. Inspect the ecosystem of containerized infrastructure daemons managed by **Podman**.
3. Filter and diagnose the lifecycle of core services (Nova, Neutron, Keystone, Cinder).
4. Stream and troubleshoot real-time API events and database interactions.
5. Understand host-level supervision and self-healing mechanisms orchestrated via **Systemd** service units.
6. Execute standard lab environment teardown in Red Hat Academy.

---

## 📋 Prerequisites & Credentials

* Active terminal session on the `workstation` VM (CL110 environment).
* Pre-configured SSH key for administrative user `heat-admin`.
* Superuser privileges (`sudo`) on the target `controller0` node.

---

## 🚀 Step-by-Step Guided Walkthrough

### Step 1: Administrative Access to the Controller Node

From the `workstation` terminal, establish a secure SSH session to the Overcloud master control node (`controller0`):

```bash
ssh heat-admin@controller0
```

Elevate your shell session to `root` to inspect the host-level Podman runtime:

```bash
sudo -i
```

> **Enterprise Architecture Context:**  
> In Red Hat OpenStack Platform (RHOSP), the Undercloud Director utilizes automated Ansible playbooks to provision and inject SSH public keys for the `heat-admin` service account across all infrastructure nodes, completely removing shared plaintext passwords.

---

### Step 2: System-Wide Container Runtime Auditing

Unlike legacy OpenStack deployments where services executed as native OS processes directly on the host, modern RHOSP encapsulates each daemon into dedicated OCI containers managed by **Podman**.

List all containerized services running on `controller0`:

```bash
podman ps --format "table {{.Names}} {{.Status}} {{.Image}}"
```

Notice the modular composition: relational databases (MariaDB/Galera), message brokers (RabbitMQ), and API endpoints all run in segregated namespaces.

---

### Step 3: Filter Core Infrastructure Daemons

Filter the output using regular expressions to isolate the primary compute, networking, and identity subsystems:

```bash
podman ps --format "table {{.Names}}  {{.Status}}" | grep -E "nova|neutron|keystone"
```

> **Expected Output:**
> ```text
> nova_api_cron          Up 29 minutes ago
> nova_metadata          Up 29 minutes ago
> nova_api               Up 29 minutes ago
> nova_vnc_proxy         Up 29 minutes ago
> nova_scheduler         Up 29 minutes ago
> nova_conductor         Up 28 minutes ago
> neutron_api            Up 29 minutes ago
> keystone               Up 29 minutes ago
> ```

---

### Step 4: Real-Time API Log Streaming and Telemetry

Whenever a compute scheduling request or virtual network binding fails, your primary troubleshooting telemetry source is the API container log stream.

Inspect the last 30 log events from the Nova compute API:

```bash
podman logs --tail 30 nova_api
```

Next, verify network routing and binding events on the Neutron daemon:

```bash
podman logs --tail 30 neutron_api
```

> **Key Architectural Takeaway:**  
> Process isolation prevents cascading failures: a high-load event or memory leak on the Horizon web frontend will not compromise the KVM hypervisors or Open vSwitch packet forwarding pipelines.

---

### Step 5: Systemd Resilience and Supervision

Containers in RHOSP do not run unmonitored; they are encapsulated within native **Systemd** service units (`tripleo_*`) to ensure ordered dependencies and automated restarts upon host failure:

```bash
systemctl status tripleo_nova_api.service --no-pager
```

> **Expected Output:**
> ```text
> ● tripleo_nova_api.service - nova_api container
>    Loaded: loaded (/etc/systemd/system/tripleo_nova_api.service; enabled; vendor preset: disabled)
>    Active: active (running) since ...
>  Main PID: 5613 (conmon)
>    CGroup: /system.slice/tripleo_nova_api.service
>            └─5613 /usr/bin/conmon ...
> ```

> **The Role of `conmon` (Container Monitor):**  
> Notice that the primary process monitored by Systemd is `/usr/bin/conmon`. Podman uses a daemonless architecture. Each container is supervised by a dedicated `conmon` monitor, allowing Systemd to track exit codes, stream standard I/O, and trigger automated self-healing restarts without relying on a monolithic container daemon.

---

### Step 6: Session Teardown and Workstation Return

Exit the privileged `root` shell, disconnect from `controller0`, and return to your `workstation` prompt:

```bash
exit
exit
```

Verify that you have safely returned to the workstation prompt:

```bash
whoami; hostname
```

> **Expected Output:**
> ```text
> student
> workstation.lab.example.com
> ```

---

## 🧪 Validation & Acceptance Criteria

This laboratory is considered complete when:
* The student successfully establishes an administrative SSH session to node `controller0`.
* The `podman ps` inspection confirms core daemons (`nova_api`, `neutron_api`, and `keystone`) are in the `Up` state.
* Real-time log telemetry (`podman logs`) verifies that Nova and Neutron APIs are actively handling requests with zero runtime exceptions.
* The `systemctl status` command validates host-level Systemd supervision and the role of the container monitor (`conmon`).
* Remote administrative shell sessions are cleanly terminated (`exit`), ensuring an orderly return to the workstation.

---

## 🧹 Resource Cleanup

Because Lab 02 performs a read-only architectural audit on the master controller node, no persistent cloud resources were created. Cleanly exiting all SSH sessions (`exit`) concludes the lab teardown.

---

## 💡 Advanced Bonus Challenges

1. **Volume and Mount Configuration Auditing:**  
   Execute `podman inspect nova_api | grep -A 10 Mounts` on `controller0` to analyze how host configuration trees (`/var/lib/config-data/...`) are mounted in read-only mode inside containers.
2. **Compute Node Inspection:**  
   From the workstation, SSH into `compute0` (`ssh heat-admin@compute0`) and inspect hypervisor-specific daemons (e.g., `nova_compute` and `neutron_ovs_agent`).

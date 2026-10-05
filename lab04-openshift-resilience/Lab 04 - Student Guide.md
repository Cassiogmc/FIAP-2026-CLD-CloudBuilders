# Lab 04 - Enterprise Resilience, Public Exposure, and Elastic Scaling on Red Hat OpenShift

* **Program:** MBA in Cloud Strategy & Architecture
* **Environment / Platform:** Red Hat Academy (DO180 v4.14 / Red Hat OpenShift Container Platform 4.14)
* **Technical Stack:** OpenShift Routes (HAProxy Ingress L7), Kubernetes Services (ClusterIP), Self-Healing Controller, Kubelet
* **Estimated Duration:** 35 to 40 minutes

---

## 🎯 Lab Objectives

Empower the student to implement enterprise resilience and exposure patterns on Red Hat OpenShift private cloud, mastering Layer 7 routing with *Routes*, validating cluster *Self-Healing* behavior under simulated failure conditions, and operating horizontal elastic scaling of application replicas.

**Enterprise Business Scenario (FinCorp Resilience & Traffic):**  
Following the successful initial deployment in Lab 03, the *FinCorp* Enterprise Architecture Review Board requires completing the application delivery pipeline: the catalog frontend service (`my-web-app`) must be published with a public FQDN reachable by external clients, demonstrate immunity against unexpected process crashes via Kubernetes autonomous *Self-Healing*, and be prepared to handle seasonal peak traffic through elastic scaling to 3 load-balanced replicas.

**Skills Acquired:**
1. Inspect the *Service* object and understand internal Layer 4 load balancing (*ClusterIP*).
2. Create and manage enterprise *OpenShift Routes* using `oc expose`, enabling public ingress through the native HAProxy router.
3. Validate enterprise route DNS resolution and HTTP connectivity via web browser.
4. Execute chaos engineering tests (*Pod Failure*), auditing the Kubernetes continuous reconciliation loop in real time.
5. Horizontally scale application capacity on demand using `oc scale`.
6. Audit traffic distribution across multiple Pods and visually confirm replica scaling in the *Topology View*.

---

## 📋 Prerequisites & Materials

* Completion of **Lab 03** with the `my-web-app` application running in project `lab-open-shift-yourname`.
* Access to the DO180 graphical `workstation` on Red Hat Academy.
* Authenticated terminal session as user `developer`:
  ```bash
  oc project lab-open-shift-yourname
  ```
* Active administrative session in Firefox Web Console (`admin` / `redhatocp`).

---

## 🚀 Step-by-Step Guided Walkthrough

### Step 1: Inspecting the Internal Service

Before exposing the application to external networks, audit the internal *Service* object created in the previous lab:

```bash
oc get svc my-web-app
```

> **Expected Output:**
> ```text
> NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
> my-web-app   ClusterIP   172.30.128.45    <none>        8080/TCP   10m
> ```

Inspect internal routing details and the Pod endpoint:

```bash
oc describe svc my-web-app
```

> **Expected Output (Relevant Snippet):**
> ```text
> Selector:          app=my-web-app
> Type:              ClusterIP
> IP:                172.30.128.45
> Port:              8080-tcp  8080/TCP
> TargetPort:        8080/TCP
> Endpoints:         10.128.2.35:8080
> ```

*Architectural Insight:* The *Service* holds an immutable virtual IP (`ClusterIP`). It uses the label selector `app=my-web-app` to dynamically forward incoming traffic to the private Pod IP (`10.128.2.35`). This address is only reachable by other services within the cluster's software-defined network (SDN).

---

### Step 2: External Exposure via OpenShift Route

To make the microservice accessible to external enterprise clients, create a **Route** based on the service:

```bash
oc expose svc/my-web-app
```

> **Expected Output:**
> ```text
> route.route.openshift.io/my-web-app exposed
> ```

Inspect the public corporate URL automatically assigned by the OpenShift Router (HAProxy):

```bash
oc get routes
```

> **Expected Output:**
> ```text
> NAME         HOST/PORT                                                         PATH   SERVICES     PORT       TERMINATION   WILDCARD
> my-web-app   my-web-app-lab-open-shift-yourname.apps.ocp4.example.com                 my-web-app   8080-tcp                 None
> ```

*How the OpenShift Router Works:* OpenShift runs a set of HAProxy instances on the infrastructure nodes (*Ingress Controller*). When a `Route` is created, the cluster updates HAProxy routing tables in microseconds, binding the generated FQDN domain to the internal service.

---

### Step 3: Validating External Access via Browser & CLI

1. In the **Firefox** browser on `workstation`, open a new tab.
2. Paste the public URL obtained in the previous step (e.g., `http://my-web-app-lab-open-shift-yourname.apps.ocp4.example.com`).
3. Verify that the default Nginx welcome page renders successfully (*"Welcome to nginx!"*).
4. In the terminal, execute an HTTP request via CLI to verify the HTTP response status code:
   ```bash
   curl -I http://$(oc get route my-web-app -o jsonpath='{.spec.host}')
   ```
   > Verify the `HTTP/1.1 200 OK` header in the response.

---

### Step 4: The Chaos Test (Failure Simulation & Self-Healing)

Kubernetes operates under the principle of **Continuous Reconciliation**: it constantly compares the *Current State* against the *Desired State*. Let us provoke a deliberate failure to audit the **Self-Healing** mechanism:

1. Identify the name of the currently running Pod:
   ```bash
   oc get pods
   ```
   *(Take note of the name, for example: `my-web-app-5d8f6b7c4d-x9q2p`)*

2. Force the immediate deletion of the Pod, simulating an unhandled system crash:
   ```bash
   oc delete pod -l app=my-web-app
   ```

3. Immediately after the command executes, list the pods in rapid succession:
   ```bash
   oc get pods
   ```

> **Expected Output:**
> ```text
> NAME                          READY   STATUS              RESTARTS   AGE
> my-web-app-5d8f6b7c4d-k4v7z   1/1     Running             0          4s
> ```

*Critical Analysis:* Notice the new Pod name suffix (`-k4v7z`) and the `AGE` field (a few seconds old). The previous Pod was terminated, but the *Deployment/ReplicaSet* controller immediately identified the discrepancy (`Current: 0 != Desired: 1`) and instructed the Kubelet to instantiate a new healthy replica. Refreshing the web browser confirms the application remained available without downtime.

---

### Step 5: Elastic Scaling of Replicas

The finance squad forecasts a significant spike in transaction volume during monthly financial closing. Scale the application to **3 replicas**:

```bash
oc scale deployment/my-web-app --replicas=3
```

> **Expected Output:**
> ```text
> deployment.apps/my-web-app scaled
> ```

Monitor the provisioning of new instances in real time:

```bash
oc get pods -o wide
```

> **Expected Output:**
> ```text
> NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE
> my-web-app-5d8f6b7c4d-k4v7z   1/1     Running   0          3m    10.128.2.36   compute1.ocp4.example.com
> my-web-app-5d8f6b7c4d-m8w2l   1/1     Running   0          12s   10.131.0.40   compute2.ocp4.example.com
> my-web-app-5d8f6b7c4d-t9r5p   1/1     Running   0          12s   10.129.2.18   compute1.ocp4.example.com
> ```

Audit the *Service* endpoints table to confirm all 3 replicas are registered behind the same load balancer:

```bash
oc get endpoints my-web-app
```

> **Expected Output:**
> ```text
> NAME         ENDPOINTS                                               AGE
> my-web-app   10.128.2.36:8080,10.129.2.18:8080,10.131.0.40:8080   15m
> ```

---

### Step 6: Visual Traffic & Topology Audit in Web Console

1. Return to the **Firefox** browser displaying the OpenShift Web Console.
2. Confirm you are in the **Developer** perspective and on the **Topology** view.
3. Observe the visual evolution of the application node:
   * The circular ring now displays the number **3** in its center, indicating **3 active Pods**.
   * The ring is segmented into 3 proportional cyan/blue sections.
   * To the upper right of the application circle, an arrow icon links to the enterprise **Route**.
4. Click the route shortcut icon (the link on the top right of the card) and confirm the application opens directly in a new browser tab.

---

## 🧪 Validation & Acceptance Criteria

This lab is considered successfully completed when:
* The command `oc get routes` lists the public route in active status with the corporate URL reachable in the browser.
* The Pod deletion test confirms autonomous recreation of the Pod by the *Self-Healing* reconciliation controller.
* The command `oc get endpoints my-web-app` lists exactly 3 registered container IP addresses.
* The Web Console interface on the **Topology** view displays the ring with **3 healthy Pods** and an active route shortcut link.

---

## 🧹 Cleanup & Resource Deallocation

Upon finishing verification, clean up all created resources in your project to release cluster memory and compute resources:

```bash
oc delete all --all
```

> **Expected Output:**
> ```text
> pod "my-web-app-5d8f6b7c4d-k4v7z" deleted
> pod "my-web-app-5d8f6b7c4d-m8w2l" deleted
> pod "my-web-app-5d8f6b7c4d-t9r5p" deleted
> service "my-web-app" deleted
> deployment.apps "my-web-app" deleted
> route.route.openshift.io "my-web-app" deleted
> ```

---

## 💡 Complementary Challenges (For Advanced Students)

1. **Load Balancing Audit with HTTP Loop:**  
   In the terminal, run a loop dispatching 15 rapid requests against the corporate route:
   ```bash
   URL=$(oc get route my-web-app -o jsonpath='{.spec.host}')
   for i in {1..15}; do curl -s -I http://$URL | grep HTTP; done
   ```
   Inspect aggregated logs across all pods using `oc logs deployment/my-web-app --tail=5 --all-containers` and prove that incoming traffic was distributed across different nodes and replica instances.

2. **Graceful Draining and Scale-Down:**  
   Scale down the replica count from 3 back to 1 (`oc scale deployment/my-web-app --replicas=1`). Observe with `oc get pods -w` how OpenShift issues a `SIGTERM` signal to excess pods, awaits graceful connection draining, and deregisters their IPs from the Service *Endpoints* table without causing 502/503 bad gateway errors on active connections.

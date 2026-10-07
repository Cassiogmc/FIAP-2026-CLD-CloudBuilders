# Lab 05 - Resilient Operations, Secrets, and Persistence in Red Hat OpenShift: Health Probes and PVC Storage

* **Program:** MBA in MultiCloud Strategy & Architecture
* **Environment / Platform:** Red Hat Academy (DO180 v4.14 / Red Hat OpenShift Container Platform 4.14)
* **Technical Stack:** OpenShift CLI (`oc`), OpenShift Web Console (Health Checks / Storage), Kubernetes Secrets, Liveness & Readiness Probes, PersistentVolumeClaims (PVC)
* **Estimated Duration:** 35 to 40 minutes

---

## 🎯 Lab Objectives

Equip the engineer to operate microservices in production on Red Hat OpenShift utilizing advanced resilience, credential governance, and persistent storage patterns—decoupling environment parameters from container images, securing credentials with Kubernetes Secrets, implementing autonomous health monitoring and self-healing (*Health Probes*), and attaching enterprise persistent volumes (*PVCs*).

**Enterprise Scenario (FinCorp Resilience & Platform Operations):**  
The operations team at *FinCorp* is hardening the core catalog and telemetry microservice (`health-app`) for enterprise production rollout. To meet strict regulatory and Central Bank compliance mandates, applications must never store plaintext credentials inside container images or require code recompilation to change operational parameters. Furthermore, the platform must autonomously detect internal thread deadlocks (triggering automated pod restarts), isolate unready pods from ingress routing, and guarantee data durability through persistent enterprise storage.

**Acquired Competencies:**
1. Decouple configuration parameters from container images using environment variables (the *12-Factor App* methodology).
2. Create and manage Kubernetes *Secrets* in OpenShift, encrypting confidential credentials in Base64 and injecting them seamlessly into *Deployments*.
3. Audit confidential runtime environment variable injection inside live containers via `oc rsh`.
4. Configure and tune *Liveness Probes*, enabling Kubelet supervisory processes to detect deadlocks and trigger automatic container recovery.
5. Configure *Readiness Probes*, ensuring the HAProxy Ingress router only directs traffic to fully initialized replicas.
6. Provision and attach a 1Gi *PersistentVolumeClaim (PVC)*, safeguarding state against pod lifecycle ephemerality.
7. Validate application health and storage bindings using the OpenShift Web Console.

---

## 📋 Prerequisites & Materials

* Access to the `workstation` VM in the Red Hat Academy DO180 lab environment.
* OpenShift cluster initialized via the wait script on the `utility` node:
  ```bash
  ssh lab@utility
  ./wait.sh
  exit
  ```
* OpenShift credentials:
  * **CLI Access:** Username `developer` / Password `developer`
  * **Web Console Access (IdM):** Username `admin` / Password `redhatocp`
* Cluster API Endpoint:
  * `https://api.ocp4.example.com:6443`
* Web Console URL:
  * `https://console-openshift-console.apps.ocp4.example.com`

---

## 🚀 Guided Step-by-Step Walkthrough

### Step 1: Authentication and Workspace Setup

On the management station (`workstation`), authenticate with the OpenShift cluster API to establish your local session configuration (`~/.kube/config`):

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
```

> **Expected Output:**
> ```text
> Login successful.
> 
> You have access to the following projects and can switch between them with 'oc project <projectname>':
> 
> Using project "default".
> ```

Next, verify that your dedicated project is active and cleared of previous resources (replace `seunome` with your unique identifier):

```bash
oc project lab-open-shift-seunome || oc new-project lab-open-shift-seunome
oc delete all --all
```

> **Expected Output:**
> ```text
> Now using project "lab-open-shift-seunome" on server "https://api.ocp4.example.com:6443".
> pod "my-web-app-..." deleted
> service "my-web-app" deleted
> deployment.apps "my-web-app" deleted
> route.route.openshift.io "my-web-app" deleted
> ```

---

### Step 2: Deploy Base Application and Inject Environment Variables (12-Factor)

In cloud-native architectures, operational parameters must remain decoupled from container images. Deploy the base Node.js application and inject an enterprise environment variable:

```bash
oc new-app https://github.com/sclorg/nodejs-ex.git --name=health-app
oc set env deployment/health-app APP_MSG="MBA MultiCloud FIAP - Resilient Enterprise Operations"
```

> **Expected Output:**
> ```text
> --> Creating resources ...
>     deployment.apps "health-app" created
>     service "health-app" created
> --> Success
> deployment.apps/health-app updated
> ```

The `oc set env` command updates the `Deployment` specification. The Kubernetes deployment controller detects this mutation and automatically orchestrates a zero-downtime *rolling update* with the new variable applied.

---

### Step 3: Enterprise Secret Management and Secure Injection

Sensitive data such as API tokens and database passwords require protection using native Kubernetes `Secret` objects.

1. Create a generic `Secret` storing a simulated database password:
   ```bash
   oc create secret generic db-pass --from-literal=password=P@ssw0rd123
   ```

   > **Expected Output:**
   > ```text
   > secret/db-pass created
   > ```

2. Inject the secret's keys as environment variables directly into the application Deployment:
   ```bash
   oc set env deployment/health-app --from=secret/db-pass
   ```

   > **Expected Output:**
   > ```text
   > deployment.apps/health-app updated
   > ```

3. Monitor the rollout status and verify that the secret was injected into the running container via `oc rsh`:
   ```bash
   oc rollout status deployment/health-app
   oc rsh deployment/health-app env | grep -i password
   ```

   > **Expected Output:**
   > ```text
   > deployment "health-app" successfully rolled out
   > password=P@ssw0rd123
   > ```

*Architectural Foundation:* The credential is kept decoupled from the image and securely injected by Kubelet upon pod scheduling on the worker node, guaranteeing identical container images across dev, test, and production environments.

---

### Step 4: Intelligent Self-Healing via Health Probes (Liveness & Readiness)

To enable OpenShift to actively monitor container health and control ingress routing, configure two distinct HTTP probes on port 8080:

1. **Liveness Probe (Supervisory Probe):**  
   Checks whether the container process is alive and healthy. If this probe repeatedly fails, Kubelet terminates and recreates the container to resolve internal lockups.
   ```bash
   oc set probe deployment/health-app --liveness --get-url=http://:8080/ --initial-delay-seconds=30
   ```

   > **Expected Output:**
   > ```text
   > deployment.apps/health-app updated
   > ```

2. **Readiness Probe (Traffic Routing Probe):**  
   Checks whether the container has finished initialization and is prepared to process external network traffic. If this probe fails, the pod is dynamically excluded from the *Service* endpoints and HAProxy pool without being killed.
   ```bash
   oc set probe deployment/health-app --readiness --get-url=http://:8080/ --initial-delay-seconds=5
   ```

   > **Expected Output:**
   > ```text
   > deployment.apps/health-app updated
   > ```

3. Monitor the deployment stabilization with both probes enabled:
   ```bash
   oc rollout status deployment/health-app
   oc get pods -l deployment=health-app
   ```

---

### Step 5: Attaching Enterprise Persistent Storage (PVC)

By default, pod root filesystems are ephemeral: if a container crashes or is rescheduled, all local modifications are lost. To guarantee durable persistence, request dedicated enterprise block storage through a *PersistentVolumeClaim (PVC)*.

1. Allocate and mount a 1Gi persistent volume at `/var/lib/storage`:
   ```bash
   oc set volume deployment/health-app --add --name=my-storage -t pvc --claim-size=1Gi --mount-path=/var/lib/storage
   ```

   > **Expected Output:**
   > ```text
   > persistentvolumeclaim/health-app-claim created
   > deployment.apps/health-app volume updated
   > ```

2. Inspect the persistent volume claim status:
   ```bash
   oc get pvc
   ```

   > **Expected Output:**
   > ```text
   > NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   > health-app-claim   Bound    pvc-7a2e5d8f-4b1c-4392-a9b1-5e8c1b9f0d3a   1Gi        RWO            gp2            25s
   > ```

   *Storage Architecture:* The `Bound` status confirms that the OpenShift CSI storage subsystem fulfilled the declarative claim, dynamically binding an underlying physical `PersistentVolume (PV)`.

3. Await rollout completion and audit the filesystem mount inside the container:
   ```bash
   oc rollout status deployment/health-app
   oc rsh deployment/health-app df -h /var/lib/storage
   ```

   > **Expected Output:**
   > ```text
   > Filesystem      Size  Used Avail Use% Mounted on
   > /dev/xvda1      1.0G   33M  991M   4% /var/lib/storage
   > ```

---

### Step 6: Visual Validation in OpenShift Web Console

1. In the Firefox browser on `workstation`, open the **OpenShift Web Console**:  
   `https://console-openshift-console.apps.ocp4.example.com`
2. Log in using **Red Hat Identity Management**:
   * **Username:** `admin`
   * **Password:** `redhatocp`
3. Switch to the **Administrator** perspective (top-left menu).
4. Verify your active project is `lab-open-shift-seunome`.
5. Navigate to **Workloads -> Deployments** and click on `health-app`.
6. Audit the following tabs:
   * **Health Checks Tab:** Verify that both *Liveness* (30s initial delay) and *Readiness* (5s initial delay) probes display active green status badges.
   * **Volumes Tab:** Confirm that `my-storage` is bound to `health-app-claim` and mounted at `/var/lib/storage`.
   * **YAML Tab:** Locate the `livenessProbe`, `readinessProbe`, and `volumeMounts` sections in the declarative resource document.

---

## 🧪 Validation & Acceptance Criteria

The laboratory is successfully completed when:
* `oc get pods -l deployment=health-app` confirms the pod is `Running` with `1/1` containers ready.
* `oc rsh deployment/health-app env | grep -i password` confirms the secret key was injected into the running process.
* `oc get pvc` shows the storage claim in the `Bound` state with `1Gi` capacity.
* The **Health Checks** panel in the Web Console confirms both Liveness and Readiness probes are operating as expected.
* **Classroom Quick Win:** Share a screenshot of the **Health Checks** tab or CLI status proving the `1/1 Running` pod and `Bound` PVC in the classroom chat.

---

## 🧹 Cleanup & Next Steps

To prepare the cluster environment for the S2I software factory in **Lab 06**, tear down the resources created in this module:

```bash
oc delete all -l app=health-app
oc delete secret db-pass
oc delete pvc health-app-claim
```

> **Expected Output:**
> ```text
> deployment.apps "health-app" deleted
> service "health-app" deleted
> secret "db-pass" deleted
> persistentvolumeclaim "health-app-claim" deleted
> ```

---

## 💡 Complementary Challenges (For Advanced Students)

1. **Simulating Liveness Failures and Kubelet Auditing:**  
   Intentionally configure an invalid health endpoint (`/non-existent-probe`) using:
   ```bash
   oc set probe deployment/health-app --liveness --get-url=http://:8080/non-existent-probe
   ```
   Monitor pod behavior with `oc get pods -w` and inspect Kubelet events with `oc describe pod -l deployment=health-app`. Observe repeated `Unhealthy` probe warnings, container termination, and an increasing `RESTARTS` count.

2. **Secret CLI Auditing and Base64 Decoding:**  
   Extract the raw declarative YAML of the secret:
   ```bash
   oc get secret db-pass -o yaml
   ```
   Locate the `data.password` field and decode the Base64 payload in the terminal:
   ```bash
   oc get secret db-pass -o jsonpath='{.data.password}' | base64 -d; echo
   ```
   Explain why storing Secrets in Base64 requires enabling etcd encryption at rest in OpenShift to meet enterprise compliance standards.

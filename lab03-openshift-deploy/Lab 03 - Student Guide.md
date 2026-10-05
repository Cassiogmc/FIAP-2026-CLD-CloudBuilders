# Lab 03 - Cloud-Native Lift-off on Red Hat OpenShift: Declarative Deployment and Visual Governance

* **Program:** MBA in Cloud Strategy & Architecture
* **Environment / Platform:** Red Hat Academy (DO180 v4.14 / Red Hat OpenShift Container Platform 4.14)
* **Technical Stack:** OpenShift CLI (`oc`), OpenShift Web Console (Topology View), CRI-O, Kubelet
* **Estimated Duration:** 35 to 40 minutes

---

## 🎯 Lab Objective

Empower practitioners to operate the enterprise Red Hat OpenShift platform, mastering the multi-tenant governance model based on namespaces, and executing declarative deployments of microservices using hardened, non-root container images.

**Enterprise Business Scenario (FinCorp Platform Engineering):**  
The platform engineering team at *FinCorp* is migrating legacy frontend services from OpenStack virtual machines to Red Hat OpenShift PaaS. Your primary mission is to provision an isolated tenant environment (*namespace*) for the financial squad and deploy the core web catalog microservice (`my-web-app`). You must enforce compliance with enterprise security constraints (*Security Context Constraints*) and visually validate workload health within the OpenShift Web Console.

**Acquired Skills:**
1. Authenticate to the OpenShift cluster via both the Command Line Interface (`oc login`) and the enterprise Web Console.
2. Provision and manage isolated multi-tenant workspaces (*namespaces*) with integrated access governance.
3. Rapidly deploy applications from enterprise container registries using `oc new-app`.
4. Understand the security enforcement of the `restricted-v2` profile and unprivileged port binding.
5. Audit Pod lifecycle, health status, and scheduling events via terminal commands.
6. Navigate the *Developer* perspective of the Web Console, mastering the visual **Topology View**.

---

## 📋 Prerequisites & Materials

* Access to the `workstation` management console for the DO180 course on Red Hat Academy.
* OpenShift cluster initialized via the monitoring script on the `utility` node:
  ```bash
  ssh lab@utility
  ./wait.sh
  exit
  ```
* OpenShift access credentials:
  * **CLI Access:** Username `developer` / Password `developer`
  * **Web Console Access (IdM):** Username `admin` / Password `redhatocp`
* Cluster API Endpoint:
  * `https://api.ocp4.example.com:6443`

---

## 🚀 Step-by-Step Guided Procedure

### Step 1: Cluster Authentication via CLI

Open a terminal session on the `workstation` and authenticate against the OpenShift cluster API:

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

Retrieve the official URL of the enterprise Web Console:

```bash
oc whoami --show-console
```

> **Expected Output:**
> ```text
> https://console-openshift-console.apps.ocp4.example.com
> ```

---

### Step 2: Access the OpenShift Web Console

1. In the **Firefox** web browser on the `workstation`, open the URL retrieved in the previous step:  
   `https://console-openshift-console.apps.ocp4.example.com`
2. On the identity provider selection screen, click **Red Hat Identity Management**.
3. Authenticate with administrative platform credentials:
   * **Username:** `admin`
   * **Password:** `redhatocp`
4. Notice the top masthead containing the project selector and the left navigation menu.

---

### Step 3: Provision an Isolated Tenant Workspace (Namespace)

In the terminal of the `workstation`, create a dedicated project for this laboratory (replace `yourname` with your first name or personal identifier):

```bash
oc new-project lab-open-shift-yourname
```

> **Expected Output:**
> ```text
> Now using project "lab-open-shift-yourname" on server "https://api.ocp4.example.com:6443".
> 
> You can add applications to this project with the 'new-app' command. For example, try:
> 
>     oc new-app rails-postgresql-example
> 
> to build a new example application in Ruby.
> ```

*Architectural Insight:* An OpenShift `Project` is an enterprise extension of the standard Kubernetes `Namespace`, adding governance annotations, role-based access control (RBAC), and virtual network isolation enforced by the OVN-Kubernetes CNI plugin.

---

### Step 4: Declarative Microservice Deployment

Instantiate the application using an enterprise container image. We will deploy Bitnami's Nginx image, which is purpose-built to run unprivileged (*non-root* on port 8080):

```bash
oc new-app --name=my-web-app bitnami/nginx:latest
```

> **Expected Output:**
> ```text
> --> Found container image ... (bitnami/nginx:latest)
>     * An image stream tag will be created as "my-web-app:latest" that will track this image
> 
> --> Creating resources ...
>     deployment.apps "my-web-app" created
>     service "my-web-app" created
> --> Success
>     Application is not exposed. You can expose services to the outside world with the 'oc expose' command.
>     Run 'oc status' to view your app.
> ```

OpenShift automatically declared three core Kubernetes resources:
* **Deployment:** Maintains the desired state and rolling update strategy of the workload.
* **ImageStream / Container:** Tracks upstream container image lifecycles.
* **Service:** Provisions a resilient internal virtual IP (*ClusterIP*) for inter-pod routing.

---

### Step 5: Audit Pod Lifecycle via CLI

Confirm container instantiation and wait until the status transitions to `Running`:

```bash
oc get pods -o wide
```

> **Expected Output:**
> ```text
> NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE
> my-web-app-5d8f6b7c4d-x9q2p   1/1     Running   0          35s   10.128.2.35   compute1.ocp4.example.com
> ```

Inspect detailed scheduling and runtime events:

```bash
oc describe pod -l app=my-web-app
```

Verify in the event logs that the Kubelet scheduled the pod to a worker node, pulled the image via CRI-O, and successfully started the container.

---

### Step 6: Visual Validation in Web Console (Topology View)

1. Return to the **Firefox** browser where the Web Console is open.
2. In the top-left perspective switcher dropdown, switch from **Administrator** to **Developer**.
3. In the top project dropdown, select your active project: `lab-open-shift-yourname`.
4. In the left navigation menu, click **Topology**.
5. Observe the interactive visual topology:
   * The central application circular node representing `my-web-app`.
   * The continuous cyan/blue donut ring showing **1 Pod** active and healthy.
   * The service badge icon (`S`).
6. Click the application circle to expand the right-hand inspection drawer.
7. On the **Details** tab, review real-time CPU and memory metrics. On the **Resources** tab, verify the managed *Deployment*, *Pod*, and *Service*.

---

## 🧪 Validation & Acceptance Criteria

This laboratory is successfully validated when:
* The command `oc get pods` outputs the application pod in `Running` state with `1/1` containers ready.
* The command `oc get svc my-web-app` validates an allocated *ClusterIP* listening on port 8080.
* The Web Console **Topology** view displays the application donut ring in a healthy, active state (solid blue/cyan).

---

## 🧹 Cleanup & Next Steps

> [!NOTE]
> **Important:** **Do not delete the resources created in this lab!**  
> The `my-web-app` deployment and the `lab-open-shift-yourname` namespace are required for **Lab 04**, where we will configure external enterprise *Routes*, execute chaos engineering tests (*Self-Healing*), and perform horizontal auto-scaling.

---

## 💡 Complementary Challenges (For Advanced Engineers)

1. **Declarative YAML Manifest Inspection:**  
   Extract the full declarative specification generated by OpenShift:
   ```bash
   oc get deployment my-web-app -o yaml
   ```
   Inspect `spec.template.spec.containers`, noting container ports, image pull policies, and the label selector `app=my-web-app`.

2. **Interactive In-Container Terminal (`oc rsh`):**  
   Establish an interactive remote shell inside the running container:
   ```bash
   oc rsh deployment/my-web-app
   ```
   Inspect the active non-root user via `whoami` (it should return an arbitrary numeric UID allocated by the SCC, such as `1000680000`), confirm the Nginx listen port (`cat /opt/bitnami/nginx/conf/nginx.conf | grep listen`), and exit the session with `exit`.

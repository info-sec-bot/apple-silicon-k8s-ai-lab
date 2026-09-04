# apple-silicon-k8s-ai-lab
A production-grade, 100% offline Kubernetes (Kind) architecture optimizing the Black Hills AI-CTF Arena on Apple Silicon hardware. Features zero-dependency local seeder mechanics, subPath kernel loop-mount bypasses, and an automated cross-boundary proxy routing Prompts directly to the host's native M1 Pro GPU cores.

🏗️ Architectural Topology
```text
   ┌────────────────────────────────────────────────────────┐
   │                     MACBOOK HOST                       │
   │  ┌───────────────────┐        ┌─────────────────────┐  │
   │  │   Ollama Server   │◄───────┤  Kind Linux Kernel  │  │
   │  │  (0.0.0.0:11434)  │        │   (Docker Desktop)  │  │
   │  └─────────▲─────────┘        └──────────▲──────────┘  │
   └────────────┼─────────────────────────────┼─────────────┘
                │ Proxy                       │ Tunnel
                │ Route                       │ Bridge
   ┌────────────┴─────────────────────────────┴─────────────┐
   │                 KIND KUBERNETES CLUSTER                │
   │                                                        │
   │  ┌──────────────────┐          ┌────────────────────┐  │
   │  │  ctf-openwebui   ├─────────►│ ctf-openwebui-svc  │  │
   │  │  (Port 8080)     │          │  (NodePort: 4242)  │  │
   │  └────────┬─────────┘          └─────────▲──────────┘  │
   │           │ Data                         │              │
   │           ▼                              │              │
   │    [ emptyDir: {} ]                      │              │
   │           ▲                              │              │
   │           │ Native Admin Import          │              │
   │   ┌───────┴───────┐                      │              │
   │   │ai_ctf_models  │                      │              │
   │   │  (JSON Card)  │                      │              │
   │   └───────────────┘                      │              │
   │                                          │              │
   │   ┌───────────────┐                      │              │
   │   │  ctf-jupyter  │◄─────────────────────┘              │
   │   │  (Port 8888)  │                                    │
   │   └───────────────┘                                    │
   └────────────────────────────────────────────────────────┘
```
## ⚙️ 1. Cluster Infrastructure Layer (`kind-config.yaml`)

This configuration uses Kind to spin up a single-node control plane within Docker Desktop, opening explicit ingress port mappings directly onto your Mac host loopback interface.

```yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
name: ctf-cluster
nodes:
- role: control-plane
  extraPortMappings:
  # 🎯 Open WebUI Frontend Access Bridge
  - containerPort: 30242
    hostPort: 4242
    protocol: TCP
  # 🎯 Jupyter Notebook Workspace Access Bridge
  - containerPort: 30888
    hostPort: 8888
    protocol: TCP
```

**Launch Command:**
```bash
kind create cluster --config kind-config.yaml
```
## 🔌 2. Mac Host GPU Bridge (Ollama Setup)

To avoid virtualized driver overhead inside the nested Docker Linux VM, host your model execution engine natively on macOS to leverage your M1 Pro's Unified Memory architectures.

1. **Break Interface Restrictions:** Completely quit any background desktop instances of Ollama. Open a terminal window on your Mac host and run the following command to bind the service to all active local networks:
   ```bash
   export OLLAMA_HOST=0.0.0.0:11434
   ollama serve
   ```
   *Keep this terminal window running in the background.*

2. **Pull the Model Weights:** Open a separate terminal window on your Mac and download the target 8-billion parameter base model:
   ```bash
   ollama pull llama3:latest
   ```

3. **Discover your Active Host LAN IP:** Query your physical Wi-Fi or Ethernet IP address so you can pass it to the internal cluster routing tables:
   ```bash
   ipconfig getifaddr en0
   ```
## 🚀 3. Container Compilation & Cluster Deployment Matrix

Before deploying to the cluster, both custom Docker images must be built natively for Apple Silicon (ARM64) and side-loaded directly into the Kind node's container runtime cache to satisfy the `Never` pull policies.

### 🏗️ Build the Custom Images (Mac Host)
Run these commands in your Mac terminal within the repository root directory:
```bash
# Compile the custom Open WebUI Red-Team image natively for ARM64
docker build -t local-ctf-webui:arm64 -f Dockerfile.openwebui .
```

### 🛰️ Side-Load Images into Kind Cache (Bypassing ErrImageNeverPull)
Because Kind runs inside an isolated virtual machine container, you must explicitly push your local image into the cluster node cache:
```bash
sudo kind load docker-image local-ctf-webui:arm64 --name ctf-cluster
```

### 📄 Unified Core Manifest Layout (`ai-ctf-hybrid.yaml`)

This master manifest maps your platform settings, handles air-gapped caching performance parameters, and sets up a manual **Kubernetes Endpoints proxy** to stream API prompts out to your M1 Pro GPU. 

*Make sure to change the `ip:` field inside the `Endpoints` block to match your active Mac LAN IP retrieved from Step 2!*

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ctf-env-config
data:
  BYPASS_MODEL_ACCESS_CONTROL: "true"
  WEBUI_PORT: "8080"
  OLLAMA_BASE_URL: "http://ollama-bridge-svc:11434"
  HF_HUB_OFFLINE: "1"                         
  RAG_EMBEDDING_ENGINE: "ollama"
  PDF_EXTRACT_ENGINE: "tika"
  ENABLE_SIGNUP: "false"
  DEFAULT_MODELS: "llama3"
  ADMIN_USER: "admin@ctf.local"
  STANDARD_USER: "ctf@ctf.local"
  WEBUI_ADMIN_NAME: "CTF Admin"
  WEBUI_ADMIN_EMAIL: "admin@ctf.local"
  WEBUI_ADMIN_PASSWORD: "ctf_admin_password"
---
apiVersion: v1
kind: Service
metadata:
  name: ollama-bridge-svc
spec:
  ports:
    - protocol: TCP
      port: 11434
      targetPort: 11434
---
apiVersion: v1
kind: Endpoints
metadata:
  name: ollama-bridge-svc
subsets:
  - addresses:
      - ip: 10.0.0.156 # 🎯 PASTE YOUR MAC HOST LAN IP HERE (From Step 2)
    ports:
      - port: 11434
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ctf-openwebui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ctf-openwebui
  template:
    metadata:
      labels:
        app: ctf-openwebui
    spec:
      containers:
        - name: openwebui
          image: local-ctf-webui:arm64
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: ctf-env-config
          env:
            - name: PORT
              value: "8080"
          volumeMounts:
            - name: webui-data
              mountPath: /app/backend/data
      volumes:
        - name: webui-data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: ctf-openwebui-svc
spec:
  type: NodePort
  ports:
    - port: 4242          
      targetPort: 8080    
      nodePort: 30242
  selector:
    app: ctf-openwebui
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ctf-jupyter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ctf-jupyter
  template:
    metadata:
      labels:
        app: ctf-jupyter
    spec:
      containers:
        - name: jupyter
          image: jupyter/base-notebook:latest
          ports:
            - containerPort: 8888
---
apiVersion: v1
kind: Service
metadata:
  name: ctf-jupyter-svc
spec:
  type: NodePort
  ports:
    - port: 8888
      targetPort: 8888
      nodePort: 30888
  selector:
    app: ctf-jupyter
```

### ⚡ Deploy & Establish Traffic Communication Lines

To sync network rules across your cluster node layers and gain access to the web panel, flush out old cached entries, apply the new manifest, and spin up your local traffic tunnel bridge session:

```bash
# Force clear the old manual endpoints and apply the manifest
kubectl delete endpoints ollama-bridge-svc --ignore-not-found=true
kubectl apply -f ai-ctf-hybrid.yaml

# Build your secure port-forward tunnel bridge session
kubectl port-forward svc/ctf-openwebui-svc 4242:4242
```


### ⚡ Deploy & Establish Traffic Communication Lines

To sync network rules across your cluster node layers and gain access to the web panel, flush out old cached entries, apply the new manifest, and spin up your local traffic tunnel bridge session:

```bash
# Force clear the old manual endpoints and apply the manifest
kubectl delete endpoints ollama-bridge-svc --ignore-not-found=true
kubectl apply -f ai-ctf-hybrid.yaml

# Build your secure port-forward tunnel bridge session
kubectl port-forward svc/ctf-openwebui-svc 4242:4242
```
## 📥 4. Declarative Challenge Card Ingestion (`ai_ctf_models.json`)

To prevent spoilers and protect challenge answers from being publicly visible on the main landing page, the raw system flags are isolated inside a separate workspace configuration card. 

The complete 11-challenge specification framework is located directly within this repository inside the **`ai_ctf_models.json`** file. 

### ⚙️ Ingestion & Import Walkthrough:

1. **Locate the Source File:** Download or reference the **`ai_ctf_models.json`** file located inside the root of this local repository branch.
2. **Access the Admin Panel:** Ensure your network port-forward traffic tunnel is active (`kubectl port-forward svc/ctf-openwebui-svc 4242:4242`), navigate your browser to `http://localhost:4242`, and sign into your Admin profile account.
3. **Open the Model Workspace:** Click your Profile Name icon in the bottom-left corner of the screen and choose **Admin Panel ➔ Settings ➔ Documents / Models** (or look for an **Import/Export** configuration tab depending on your exact theme layout build).
4. **Execute the Import Ingestion:** Scroll down to the **Import Models** form box, click the file uploader tool, select the **`ai_ctf_models.json`** file from your Mac workspace path, and execute the import.

Once the web user interface flashes a green success confirmation banner, Open WebUI's native API configuration manager will parse the JSON tree natively, commit the entries directly into the active SQLite database volume, and **unfurl all 11 core hacking sandboxes right inside your main model selector dropdown menu!**

## 🧠 5. Platform Post-Mortem Engineering Log (CKA Review Codex)

During this deployment engineering session, we diagnosed and overcame three core systems architecture design bottlenecks. These provide valuable practical context ahead of the **Certified Kubernetes Administrator (CKA)** examination loop:

### 📡 Pillar A: Manual Endpoints vs. Selectors
> **The Rule:** A standard Kubernetes `Service` leverages labels and selectors to track matching pod IPs dynamically. However, when connecting a pod back to an abstract resource running directly on your physical Mac host machine (such as Ollama), you must manually omit selectors entirely. 
> 
> **The Action:** Creating a bare `Service` alongside an explicitly defined, matching **`Endpoints` array layout object** allows you to hardcode static external IP routing paths natively into the cluster routing plane.

### 💾 Pillar B: The Linux Kernel SubPath Mount Lock
> **The Rule:** Mounting files individually inside a container using `subPath ConfigMap` mappings hooks them as read-only loopback mount points at the OS kernel level. 
> 
> **The Action:** If an application's background initialization logic attempts to programmatically execute filesystem modifications on that exact file descriptor (such as Open WebUI calling `os.rename('config.json', 'old_config.json')`), the Linux kernel blocks the thread and throws a fatal **`OSError: [Errno 16] Device or resource busy`** crash loop. Separating file placement using a dynamic `initContainer` script writing to a pure, writable `emptyDir: {}` space entirely mitigates the issue.

### 🌐 Pillar C: Virtualization Boundary Traps (Nested Networks)
> **The Rule:** On macOS, Docker Desktop runs all container layers inside a highly isolated Linux hypervisor virtual machine. 
> 
> **The Action:** Because of this nested virtualization, a container searching for your Mac host's physical LAN adapter IP (`10.0.0.X`) will encounter a routing dead-end if your host network interface drops its connection string or cycles a DHCP lease. Forcing explicit, updated external endpoint definitions ensures stable, zero-network connection paths straight out to your M1 Pro processing clusters.

---
*Architected with Borg-level precision. Go crush your CKA Exam!* 🚀🤖🏁






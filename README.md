# DevOps Internship Assessment - Todo App

This repository documents the complete DevOps pipeline for deploying a Node.js Todo application using Docker, GitHub Actions, Ansible, Kubernetes, and ArgoCD.

---

## Part 1: Dockerizing the App & CI Pipeline

### Repo Cloned
Original repo: [Ankit6098/Todo-List-nodejs](https://github.com/Ankit6098/Todo-List-nodejs)

### Dockerfile
Created a Dockerfile:
```dockerfile
FROM node:20-alpine
# Set the working directory
WORKDIR /usr/src/app
# Copy package.json and package-lock.json
COPY package*.json ./
# Install dependencies
RUN npm install 
# Copy the rest of the application code
COPY . .
# Expose the application port
EXPOSE 4000
# Start the application
CMD ["node", "index.js"]
```

### GitHub Actions CI


Created `.github/workflows/build-push.yaml`:

```yaml
name: build-and-push-docker 
on: 
    workflow_dispatch:
    push:
        branches:
            - main
            - 'feature'
        paths:
            - '**'
            - '!k8s/**'  
            - '!README.md'
            - '!Ansible/**'

jobs:
    build-and-push:
        runs-on: ubuntu-latest
        steps:
            - name: Checkout
              uses: actions/checkout@v4.2.2

            - name: Login to Docker
              uses: docker/login-action@v3.4.0
              with:
                username: ${{ vars.DOCKERHUB_USERNAME }}
                password: ${{ secrets.DOCKERHUB_PAT }}

            - name: Docker Build For Testing
              uses: docker/build-push-action@v6.18.0
              with:
                context: .
                push: true 
                tags: ${{ vars.DOCKERHUB_USERNAME }}/to-do:${{ github.sha }}

                
    update-gitops:
      needs: build-and-push
      runs-on: ubuntu-latest
      steps:
        - name: Checkout GitOps repo
          uses: actions/checkout@v4
          with:
            repository: Salieri20/Todo-List-nodejs
            token: ${{ secrets.GITOPS_TOKEN }}
            path: gitops
    
        - name: Update deployment image tag
          env:
            DOCKERHUB_USERNAME: ${{ vars.DOCKERHUB_USERNAME }}
          run: |
            TAG=${GITHUB_SHA}
            FILE=gitops/k8s/app-deployment.yaml
            sed -i "s|image: .*/to-do:.*|image: ${DOCKERHUB_USERNAME}/to-do:${TAG}|" $FILE
            
    
        - name: Commit and push change
          run: |
            cd gitops
            git config user.name "github-actions"
            git config user.email "github-actions@users.noreply.github.com"
            git add .
            git commit -m "Update image tag to ${GITHUB_SHA}"
            git push
```
### Why This Setup?
- **Manual + Auto Trigger**: We used both workflow_dispatch (manual) and push events (on main and feature/**) to trigger builds either manually or via commits.
- **Path Filtering**: The line - '!k8s/**' ensures that changes limited to Kubernetes YAML files do not retrigger Docker builds unnecessarily.
- **Separate GitOps Update**: Once the Docker image is built and pushed, we automatically update the app-deployment.yaml file in the same repo (under the k8s/ folder) to reflect the new image tag.
- **Security**: All sensitive credentials are handled via GitHub Secrets (DOCKERHUB_PAT, GITOPS_TOKEN).
- **Image Tagging**: The image is tagged with github.sha to ensure traceability and avoid conflicts.
- **Git Automation**: We configure git with a bot identity to commit the deployment changes without manual interaction. 

---

## Part 2: Ansible for VM Setup

- Created a Linux VM on local machine
### 🔧 What the Playbook Does
- Installs essential system tools: `curl`, `wget`, `git`, `yum-utils`, `lvm2`.
- Adds Docker CE repository and installs Docker components (`docker-ce`, `docker-ce-cli`, `containerd.io`).
- Enables and starts the Docker service.
- Adds the current user (`salieri`) to the `docker` group to allow non-root Docker usage.
- Installs `kubectl` (Kubernetes command-line tool) from the official binary.
- Installs Minikube to run a local single-node Kubernetes cluster.
- Starts Minikube with Docker as the driver.


### `Ansible/inventory`
```ini
[nodes]
node01 ansible_host=192.168.28.129 ansible_user=salieri ansible_ssh_private_key_file=~/.ssh/id_rsa 
```

### `Ansible/playbook.yml`
```yaml
- name: Prepare CentOS VM for Kubernetes and ArgoCD
  hosts: all
  become: true

  tasks:
    - name: Install required system packages
      yum:
        name:
          - curl
          - wget
          - git
          - yum-utils
          - device-mapper-persistent-data
          - lvm2
        state: present

    - name: Add Docker CE repo
      get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo

    - name: Install Docker
      yum:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
        state: present

    - name: Enable and start Docker
      systemd:
        name: docker
        enabled: true
        state: started

    - name: Add current user to docker group
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes

    - name: Install kubectl
      get_url:
        url: https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl
        dest: /usr/local/bin/kubectl
        mode: '0755'
        timeout: 120

    - name: Install Minikube
      get_url:
        url: https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
        dest: /usr/local/bin/minikube
        mode: '0755'
        timeout: 120

    - name: Start Minikube
      become: false
      shell: minikube start --driver=docker
      environment:
        PATH: "/usr/local/bin:{{ ansible_env.PATH }}"
      when: start_minikube | default(false)
```

---

## Part 3 : Kubernetes + ArgoCD GitOps + Health checks + Auto update

- Deployed Kubernetes on the VM
- Installed ArgoCD
- Used ArgoCD to watch and sync the `k8s/` folder in the repo
- Used GitHub Actions for the auto-update part

### `k8s/app-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      imagePullSecrets:
        - name: regcred
      containers:
        - name: myapp
          image: salieri20/to-do:b790f9966d954920071380f3cccc939db6da755c
          ports:
            - containerPort: 4000
          env:
            - name: mongoDbUrl
              value: mongodb://mongodb:27017/mydb
          livenessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
              
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 4000
      targetPort: 4000
      nodePort: 32001
```

### `k8s/mongodb-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-storage
              mountPath: /data/db
      volumes:
        - name: mongo-storage
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```


### ArgoCD Setup
- Exposed ArgoCD UI on port 31381
- Connected the GitHub repo with the `k8s/` folder as the manifest source
- ArgoCD continuously syncs to apply new Docker image tags when changed via GitHub Actions

---

## Folder Structure

```
├── .github/
│   └── workflows/
│       └── build-and-push-docker.yaml
├── ansible/
│   ├── playbook.yml
│   └── hosts
├── k8s/
│   ├── app-deployment.yaml
│   └── app-service.yaml
├── Dockerfile
├── .env.example
├── README.md
```

---

## 🧪 Health Checks

- Implemented in Kubernetes via `livenessProbe` and `readinessProbe`
- Health endpoint available at `/health`

```bash
curl http://<NODE-IP>:32001/health
```

Returns:
```json
{"status":"ok"}
```

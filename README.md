# app-hello-nti — Jenkins → Docker Hub → Minikube

Flask app deployed to a local Minikube cluster by a Jenkins pipeline.

## Layout

```
.
├── hello.py           # Flask app (/ and /health)
├── requirements.txt   # flask, gunicorn
├── Dockerfile         # python:3.12-slim, runs gunicorn on :5000
├── Jenkinsfile        # build → push → deploy → verify
└── K8s/
    ├── deployment.yaml # Deployment, 2 replicas, probes on /health
    └── service.yaml    # Service, NodePort 30080 → :5000
```

## Jenkins credentials

All three must exist in Jenkins under *Manage Jenkins → Credentials*:

| ID | Type | Purpose |
|---|---|---|
| `jenkins-token` | Username/Password or token | Clone the GitHub repo |
| `docker` | Username/Password | Push to Docker Hub |
| `minikube-kubeconfig` | **Secret file** | kubeconfig from `minikube kubeconfig` |

Get the kubeconfig file from the Minikube host:

```bash
minikube status
minikube kubeconfig > /tmp/minikube-kubeconfig
```

## Run it

Push the repo to `main`, create a Pipeline job pointing at it, and build.
Reach the app at `http://<minikube-ip>:30080`.

```bash
kubectl get pods
kubectl get svc app-hello-nti
```

## Local run (no cluster)

```bash
python hello.py          # dev server on :5000
docker build -t app-hello-nti:1 .
docker run -p 5000:5000 app-hello-nti:1
```
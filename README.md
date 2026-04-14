# setup-infra-docker-engine-docker-compose-k3d-kubectl-1s-1a-
1s a1 with disable treafik
----------------
#!/bin/bash

set -e

LOG_FILE="/root/setup.log"
exec > >(tee -a $LOG_FILE) 2>&1

echo "🚀 Starting Infra Setup..."

# -------------------------------
# 1. System Update
# -------------------------------
echo "📦 Updating system..."
apt-get update -y

# -------------------------------
# 2. Install Required Packages
# -------------------------------
echo "📦 Installing dependencies..."
apt-get install -y ca-certificates curl gnupg lsb-release docker.io snapd

# -------------------------------
# 3. Enable & Start Docker
# -------------------------------
echo "🐳 Starting Docker..."
systemctl enable docker
systemctl start docker

echo "⏳ Waiting for Docker..."
until docker info >/dev/null 2>&1; do
  sleep 3
done
echo "✅ Docker is ready"

# -------------------------------
# 4. Install k3d
# -------------------------------
echo "☸️ Installing k3d..."
if ! command -v k3d >/dev/null 2>&1; then
  curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
else
  echo "k3d already installed"
fi

# -------------------------------
# 5. Install kubectl
# -------------------------------
echo "📦 Installing kubectl..."
if ! command -v kubectl >/dev/null 2>&1; then
  snap install kubectl --classic
else
  echo "kubectl already installed"
fi

# -------------------------------
# 6. Create k3d Cluster
# -------------------------------
echo "🚀 Creating k3d cluster..."

if k3d cluster list | grep -q prod; then
  echo "⚠️ Cluster 'prod' already exists. Skipping..."
else
  k3d cluster create prod \
    -s 1 -a 1 \
    --k3s-arg "--disable=traefik@server:0" \
    -p "80:80@loadbalancer" \
    -p "443:443@loadbalancer"
fi

# -------------------------------
# 7. Setup kubeconfig
# -------------------------------
echo "🔧 Setting up kubeconfig..."

mkdir -p /root/.kube
k3d kubeconfig get prod > /root/.kube/config

export KUBECONFIG=/root/.kube/config

# -------------------------------
# 8. Wait for Kubernetes
# -------------------------------
echo "⏳ Waiting for Kubernetes..."

until kubectl get nodes >/dev/null 2>&1; do
  sleep 5
done

kubectl wait --for=condition=Ready nodes --all --timeout=120s

echo "✅ Kubernetes is ready"

# -------------------------------
# 9. Create Namespace
# -------------------------------
echo "📦 Creating namespace..."

kubectl create namespace prod || echo "Namespace already exists"

kubectl config set-context --current --namespace=prod

# -------------------------------
# 10. Install NGINX Ingress
# -------------------------------
echo "🌐 Installing NGINX Ingress..."

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s

echo "✅ Ingress Controller Ready"

# -------------------------------
# 11. Install cert-manager
# -------------------------------
echo "🔐 Installing cert-manager..."

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

echo "⏳ Waiting for cert-manager..."
sleep 30

# -------------------------------
# DONE
# -------------------------------
echo "🎉 Setup Completed Successfully!"
echo "📄 Logs available at: $LOG_FILE"

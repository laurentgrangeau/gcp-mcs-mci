# Multi-cluster Service & multi-cluster Ingress
## Bootstrap
### Export PROJECT_ID and REGION variables
```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION_US=us-central1
export REGION_EU=europe-west1
export REGION_AS=asia-east2
export REGION_CC=europe-west2
```

### Enable GCP APIs
```bash
gcloud services enable \
    anthos.googleapis.com \
    cloudresourcemanager.googleapis.com \
    container.googleapis.com \
    dns.googleapis.com \
    gkehub.googleapis.com \
    multiclusteringress.googleapis.com \
    multiclusterservicediscovery.googleapis.com \
    trafficdirector.googleapis.com \
    --project=$PROJECT_ID
```

### Create GKE clusters

### Get GKE credentials
```bash
gcloud container clusters get-credentials mcs-us \
    --zone=us-central1 \
    --project=$PROJECT_ID
gcloud container clusters get-credentials mcs-eu \
    --zone=europe-west1 \
    --project=$PROJECT_ID
gcloud container clusters get-credentials mcs-as \
    --zone=asia-east2 \
    --project=$PROJECT_ID
gcloud container clusters get-credentials mcs-cc \
    --zone=europe-west2 \
    --project=$PROJECT_ID
```

### Rename context for each cluster
```bash
kubectl config rename-context gke_${PROJECT_ID}_us-central1_mcs-us gke-us
kubectl config rename-context gke_${PROJECT_ID}_europe-west1_mcs-eu gke-eu
kubectl config rename-context gke_${PROJECT_ID}_asia-east2_mcs-as gke-as
kubectl config rename-context gke_${PROJECT_ID}_europe-west2_mcs-cc gke-cc
```

## Multi-cluster Service
### Enable MCS on the fleet
```bash
gcloud container fleet multi-cluster-services enable \
    --project $PROJECT_ID
```

### Register all clusters in the fleet
```bash
gcloud container fleet memberships register gke-us \
    --gke-cluster us-central1/mcs-us \
    --enable-workload-identity \
    --project=$PROJECT_ID

gcloud container fleet memberships register gke-eu \
    --gke-cluster europe-west1/mcs-eu \
    --enable-workload-identity \
    --project=$PROJECT_ID

gcloud container fleet memberships register gke-as \
    --gke-cluster asia-east2/mcs-as \
    --enable-workload-identity \
    --project=$PROJECT_ID
```

### Verify that all clusters are registered
```bash
gcloud container fleet memberships list --project=$PROJECT_ID
```

### Add IAM role to the Service Account that runs the clusters
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member "serviceAccount:${PROJECT_ID}.svc.id.goog[gke-mcs/gke-mcs-importer]" \
    --role "roles/compute.networkViewer"
```

### Verify that MCS is enabled
```bash
gcloud container fleet multi-cluster-services describe --project=$PROJECT_ID
```

### Create MCS namespace to host all MCS resources
```bash
kubectl create ns mcs --context gke-us
kubectl create ns mcs --context gke-eu
kubectl create ns mcs --context gke-as
```

### Create NGinx deployment and service in EU cluster
```bash
kubectl create deployment nginx-eu --image nginx --context gke-eu --namespace mcs
kubectl create service clusterip nginx-eu --tcp=80:80 --context gke-eu --namespace mcs
```

### Create a ServiceExport to export the EU service to all clusters of the fleet
```bash
cat <<EOF | kubectl apply --context gke-eu --namespace mcs -f -
apiVersion: net.gke.io/v1
kind: ServiceExport
metadata:
 name: nginx-eu
EOF
```

### Verify that the ServiceExport is created
```bash
kubectl get serviceexport --context gke-eu --namespace mcs
kubectl describe serviceexport/nginx-eu --context gke-eu --namespace mcs
```

### Create a deployment to cURL the service from the US
```bash
cat <<EOF | kubectl apply --context gke-us --namespace mcs -f -
apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  containers:
  - name: dnsutils
    image: curlimages/curl
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF
```

### cURL the service from the US cluster
```bash
kubectl exec -i -t curl --namespace mcs --context gke-us -- curl nginx-eu.mcs.svc.clusterset.local
```

## Multi-cluster Ingress
### Register the Config cluster in the fleet
```bash
gcloud container fleet memberships register gke-cc \
    --gke-cluster europe-west2/mcs-cc \
    --enable-workload-identity \
    --project=$PROJECT_ID
```

### Enable Multi-cluster Ingress on the fleet
```bash
gcloud container fleet ingress enable \
  --config-membership=gke-cc \
  --project=$PROJECT_ID
```

### Verify that MCI is enabled
```bash
gcloud container fleet ingress describe --project=$PROJECT_ID
```

### Create the WhereAmI deployment in each cluster of the fleet
```bash
cat <<EOF | kubectl apply --context gke-us --namespace mcs -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whereami-deployment
  labels:
    app: whereami
spec:
  selector:
    matchLabels:
      app: whereami
  template:
    metadata:
      labels:
        app: whereami
    spec:
      containers:
      - name: frontend
        image: us-docker.pkg.dev/google-samples/containers/gke/whereami:v1.2.7
        ports:
        - containerPort: 8080
EOF

cat <<EOF | kubectl apply --context gke-eu --namespace mcs -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whereami-deployment
  labels:
    app: whereami
spec:
  selector:
    matchLabels:
      app: whereami
  template:
    metadata:
      labels:
        app: whereami
    spec:
      containers:
      - name: frontend
        image: us-docker.pkg.dev/google-samples/containers/gke/whereami:v1.2.7
        ports:
        - containerPort: 8080
EOF

cat <<EOF | kubectl apply --context gke-as --namespace mcs -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whereami-deployment
  labels:
    app: whereami
spec:
  selector:
    matchLabels:
      app: whereami
  template:
    metadata:
      labels:
        app: whereami
    spec:
      containers:
      - name: frontend
        image: us-docker.pkg.dev/google-samples/containers/gke/whereami:v1.2.7
        ports:
        - containerPort: 8080
EOF
```

### Create a MultiClusterService in the Config cluster
```bash
cat <<EOF | kubectl apply --context gke-cc --namespace mcs -f -
apiVersion: networking.gke.io/v1
kind: MultiClusterService
metadata:
  name: whereami-mcs
spec:
  template:
    spec:
      selector:
        app: whereami
      ports:
      - name: web
        protocol: TCP
        port: 8080
        targetPort: 8080
EOF
```

### Create a MultiClusterIngress in the Config cluster
```bash
cat <<EOF | kubectl apply --context gke-cc --namespace mcs -f -
apiVersion: networking.gke.io/v1
kind: MultiClusterIngress
metadata:
  name: whereami-ingress
spec:
  template:
    spec:
      backend:
        serviceName: whereami-mcs
        servicePort: 8080
EOF
```

### Verify that the MultiClusterIngress is created
```bash
kubectl describe mci whereami-ingress --context gke-cc --namespace mcs
```

### cURL the MultiClusterIngress VIP
```bash
export INGRESS_VIP=$(kubectl get mci whereami-ingress --context gke-cc --namespace mcs -o json | jq -r '.status.VIP')
curl "http://${INGRESS_VIP}/ping"
```

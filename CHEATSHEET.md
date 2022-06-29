# Multi-cluster Service & multi-cluster Ingress
## Bootstrap
export PROJECT_ID=$(gcloud config get-value project)

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

kubectl config rename-context gke_${PROJECT_ID}_us-central1_mcs-us gke-us
kubectl config rename-context gke_${PROJECT_ID}_europe-west1_mcs-eu gke-eu
kubectl config rename-context gke_${PROJECT_ID}_asia-east2_mcs-as gke-as
kubectl config rename-context gke_${PROJECT_ID}_europe-west2_mcs-cc gke-cc

## Multi-cluster Service
gcloud container fleet multi-cluster-services enable \
    --project $PROJECT_ID

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

gcloud container fleet memberships list --project=$PROJECT_ID

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member "serviceAccount:${PROJECT_ID}.svc.id.goog[gke-mcs/gke-mcs-importer]" \
    --role "roles/compute.networkViewer"

gcloud container fleet multi-cluster-services describe --project=$PROJECT_ID

kubectl create ns mcs --context gke-us
kubectl create ns mcs --context gke-eu
kubectl create ns mcs --context gke-as

kubectl create deployment nginx-eu --image nginx --context gke-eu --namespace mcs

kubectl create service clusterip nginx-eu --tcp=80:80 --context gke-eu --namespace mcs

cat <<EOF | kubectl apply --context gke-eu --namespace mcs -f -
apiVersion: net.gke.io/v1
kind: ServiceExport
metadata:
 name: nginx-eu
EOF

kubectl get serviceexport --context gke-eu --namespace mcs

kubectl describe serviceexport/nginx-eu --context gke-eu --namespace mcs

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

kubectl exec -i -t curl --namespace mcs --context gke-us -- curl nginx-eu.mcs.svc.clusterset.local

## Multi-cluster Ingress
gcloud container fleet memberships register gke-cc \
    --gke-cluster europe-west2/mcs-cc \
    --enable-workload-identity \
    --project=$PROJECT_ID

gcloud container fleet ingress enable \
   --config-membership=gke-cc

gcloud container fleet ingress describe --project=$PROJECT_ID

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

kubectl describe mci whereami-ingress --context gke-cc --namespace mcs

export INGRESS_VIP=$(kubectl get mci whereami-ingress --context gke-cc --namespace mcs -o json | jq -r '.status.VIP')

curl "http://${INGRESS_VIP}/ping"
# Anthos Service Mesh w/ Internal L4 LB
以下試した時の雑なメモ：  
・ASM の istio-ingressgateway を Internal L4 LB でデプロイする  
・Authorization Policy で 送信元 IP アドレスベースの Whitelist を設定する  

オフィシャルの ASM インストール手順はこちら：  
https://cloud.google.com/service-mesh/docs/install  

## GKE クラスタのデプロイ

### 各変数の設定

```bash
export GCP_EMAIL_ADDRESS=$(gcloud auth list --filter=status:ACTIVE \
  --format="value(account)")
export PROJECT_ID=$(gcloud config list --format   "value(core.project)")
export PROJECT_NUMBER=\$(gcloud projects describe \${PROJECT_ID}   --format="value(projectNumber)")
export CLUSTER_NAME=ilb-kuchima
export CLUSTER_LOCATION=asia-southeast1-a
export CLUSTER_REGION=asia-southeast1
export WORKLOAD_POOL=\${PROJECT_ID}.svc.id.goog
export MESH_ID="proj-\${PROJECT_NUMBER}"
export CHANNEL="regular"
export GCP_EMAIL_ADDRESS=$(gcloud auth list --filter=status:ACTIVE   --format="value(account)")
export ENVIRON_PROJECT_NUMBER=$PROJECT_NUMBER
export POD_CIDR="10.56.0.0/14"
export POD_CIDR_NAME="pods"
export SVCS_CIDR="10.120.0.0/20"

gcloud config set compute/zone \${CLUSTER_LOCATION}
```

### API の有効化

```bash
gcloud services enable \
    --project=${PROJECT_ID} \
    container.googleapis.com \
    compute.googleapis.com \
    monitoring.googleapis.com \
    logging.googleapis.com \
    cloudtrace.googleapis.com \
    meshca.googleapis.com \
    meshtelemetry.googleapis.com \
    meshconfig.googleapis.com \
    iamcredentials.googleapis.com \
    gkeconnect.googleapis.com \
    gkehub.googleapis.com \
    cloudresourcemanager.googleapis.com
```

### サブネットの作成

```bash
gcloud compute networks subnets update default \
    --region ${CLUSTER_REGION} \
    --add-secondary-ranges ${POD_CIDR_NAME}=${POD_CIDR}
```

### GKE クラスタのデプロイ

```bash
gcloud beta container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION} \
    --enable-ip-alias \
    --machine-type e2-custom-4-6144 \
    --num-nodes=4 \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-stackdriver-kubernetes \
    --subnetwork=default \
    --cluster-secondary-range-name=pods \
    --services-ipv4-cidr=${SVCS_CIDR} \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing \
    --labels mesh_id=${MESH_ID} \
    --release-channel=regular
```

### Credential の取得

```bash
gcloud container clusters get-credentials ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION}
```

### プロジェクトの初期化

```bash
curl --request POST   --header "Authorization: Bearer $(gcloud auth print-access-token)"   --data ''   https://meshconfig.googleapis.com/v1alpha1/projects/${PROJECT_ID}:initialize
```

### GKE クラスタの Role Binding 設定 (cluster-admin)

```bash
kubectl create clusterrolebinding cluster-admin-binding   --clusterrole=cluster-admin   --user="$(gcloud config get-value core/account)"
```

## Anthos Service Mesh のインストール

### インストール資源のダウンロード

```bash
curl -LO https://storage.googleapis.com/gke-release/asm/istio-1.6.11-asm.1-linux-amd64.tar.gz

tar xzf istio-1.6.11-asm.1-linux-amd64.tar.gz

cd istio-1.6.11-asm.1
export PATH=$PWD/bin:$PATH
```

### ASM パッケージのダウンロードと各種設定

```bash
sudo apt-get install google-cloud-sdk-kpt -y

kpt pkg get https://github.com/GoogleCloudPlatform/anthos-service-mesh-packages.git/asm@release-1.6-asm asm
kpt cfg set asm gcloud.container.cluster ${CLUSTER_NAME}
kpt cfg set asm gcloud.core.project ${PROJECT_ID}
kpt cfg set asm gcloud.compute.location ${CLUSTER_LOCATION}
kpt cfg set asm gcloud.project.environProjectNumber ${ENVIRON_PROJECT_NUMBER}
kpt cfg set asm anthos.servicemesh.profile asm-gcp

```

### ASM インストール w/ Internal LB

```bash
istioctl install -f asm/cluster/istio-operator.yaml \
    --revision=asm-1611-1 \
    --set values.global.proxy.tracer=stackdriver \
    --set values.pilot.traceSampling=100 \
    --set values.telemetry.v2.stackdriver.enabled=true \
    --set values.telemetry.v2.stackdriver.logging=true \
    --set values.telemetry.v2.stackdriver.outboundAccessLogging=FULL \
    --set meshConfig.outboundTrafficPolicy.mode=ALLOW_ANY \
    -f istio-configuration/
```

### 対象 Namespace に対して envoy auto-injection を有効化

```bash
kubectl label namespace default istio-injection- istio.io/rev=asm-1611-1 --overwrite
```

### サンプルアプリのデプロイ

```bash
kubectl apply -f sample-app/
```

Istio の Authorization Policy は特定 IP アドレスからのみ許可するようにしておきます：  
```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ingress-policy
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: ALLOW
  rules:
  - from:
    - source:
       ipBlocks: ["10.148.0.7/32"]
```

## 疎通テスト

### Istio-ingressgateway Service の IP アドレス を確認

```bash
kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.120.5.46   10.148.0.13   15020:30612/TCP,80:32527/TCP,443:32133/TCP   138m

```

### 同一 VPC 内ホスト 1 (NG パターン)

```bash
kuchima@ilb-south-4:~$ hostname -i
10.20.0.2
kuchima@ilb-south-4:~$ curl 10.148.0.13/headers -s -o /dev/null -w "%{http_code}\n"
403
```

### 同一 VPC 内ホスト 2 (OK パターン)

```bash
kuchima@ilb-south:~$ hostname -i
10.148.0.7
kuchima@ilb-south:~$ curl 10.148.0.13/headers -s -o /dev/null -w "%{http_code}\n"
200
```

### Envoy Access Log
```bash
[2020-11-02T10:57:38.134Z] "GET /headers HTTP/1.1" 200 - "-" "-" 0 864 3 2 "10.148.0.7" "curl/7.64.0" "3eb282bc-8d3a-934a-959f-7938da6a748a" "10.148.0.13" "10.56.0.9:80" outbound|80|v1|httpbin-service.default.svc.cluster.local 10.56.0.8:44144 10.56.0.8:80 10.148.0.7:36670 - -
[2020-11-02T10:57:41.859Z] "GET /headers HTTP/1.1" 403 - "-" "-" 0 19 0 - "10.20.0.2" "curl/7.64.0" "ede32dca-1417-9f2e-949f-2cb79f34adb2" "10.148.0.13" "-" - - 10.56.0.8:80 10.20.0.2:56314 - -
```
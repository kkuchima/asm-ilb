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
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID}   --format="value(projectNumber)")
export CLUSTER_NAME=ilb-kuchima
export CLUSTER_LOCATION=asia-southeast1-a
export CLUSTER_REGION=asia-southeast1
export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
export MESH_ID="proj-${PROJECT_NUMBER}"
export CHANNEL="regular"
export GCP_EMAIL_ADDRESS=$(gcloud auth list --filter=status:ACTIVE   --format="value(account)")
export ENVIRON_PROJECT_NUMBER=$PROJECT_NUMBER
export POD_CIDR="10.56.0.0/14"
export POD_CIDR_NAME="pods"
export SVCS_CIDR="10.120.0.0/20"

gcloud config set compute/zone ${CLUSTER_LOCATION}
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
export ASM_VERSION=istio-1.6.11-asm.1
curl -LO https://storage.googleapis.com/gke-release/asm/${ASM_VERSION}-linux-amd64.tar.gz

tar xzf ${ASM_VERSION}-linux-amd64.tar.gz

cd ${ASM_VERSION}
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
git clone https://github.com/kkuchima/asm-ilb

istioctl install -f asm/cluster/istio-operator.yaml \
    --revision=asm-1611-1 \
    --set values.global.proxy.tracer=stackdriver \
    --set values.pilot.traceSampling=100 \
    --set values.telemetry.v2.stackdriver.enabled=true \
    --set values.telemetry.v2.stackdriver.logging=true \
    --set values.telemetry.v2.stackdriver.outboundAccessLogging=FULL \
    --set meshConfig.outboundTrafficPolicy.mode=ALLOW_ANY \
    -f asm-ilb/istio-configuration/ilb-gateway.yaml
```

### istio-system namespace 内の Pod が Running 状態になっていることを確認

```bash
kubectl get pods -n istio-system -w
```

### 対象 Namespace に対して envoy auto-injection を有効化

```bash
kubectl label namespace default istio-injection- istio.io/rev=asm-1611-1 --overwrite
```

### サンプルアプリのデプロイ

```bash
kubectl apply -f asm-ilb/sample-app/
```

※(Paste不要)Istio の Authorization Policy は特定 IP アドレスからのみ許可するようにしておきます  
（以下の例では後述のテスト用ホスト2のIPアドレスを指定)：  
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

### クライアントIP保持の設定
```bash
kubectl patch svc istio-ingressgateway -n istio-system -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

## 疎通テスト

### Istio-ingressgateway Service の IP アドレス を確認

```bash
kubectl get svc istio-ingressgateway -n istio-system

### 出力例:
### NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                      AGE
### istio-ingressgateway   LoadBalancer   10.120.5.102   10.148.0.18   15020:30624/TCP,80:30319/TCP,443:31027/TCP   3m36s
```

### 同一 VPC 内ホスト 1 (NG パターン)
同一VPCの同一リージョン内ホスト1に SSH し、 Istio-ingressgateway の External IP アドレス宛てに curl を発行する。
```bash
hostname -i
### 出力例:
### 10.20.0.2

curl 10.148.0.18/headers -s -o /dev/null -w "%{http_code}\n"
### 出力例:
### 403
```

### 同一 VPC 内ホスト 2 (OK パターン)
同一VPCの同一リージョン内ホスト1に SSH し、 Istio-ingressgateway の External IP アドレス宛てに curl を発行する。
```bash
hostname -i
### 出力例:
### 10.148.0.7

curl 10.148.0.18/headers -s -o /dev/null -w "%{http_code}\n"
### 出力例:
### 200
```

### Envoy Access Log の確認
```bash
kubectl logs istio-ingressgateway-66fbcbc9ff-5vd27 -n istio-system

### 中略 以下出力例:
[2020-11-02T10:57:38.134Z] "GET /headers HTTP/1.1" 200 - "-" "-" 0 864 3 2 "10.148.0.7" "curl/7.64.0" "3eb282bc-8d3a-934a-959f-7938da6a748a" "10.148.0.13" "10.56.0.9:80" outbound|80|v1|httpbin-service.default.svc.cluster.local 10.56.0.8:44144 10.56.0.8:80 10.148.0.7:36670 - -
[2020-11-02T10:57:41.859Z] "GET /headers HTTP/1.1" 403 - "-" "-" 0 19 0 - "10.20.0.2" "curl/7.64.0" "ede32dca-1417-9f2e-949f-2cb79f34adb2" "10.148.0.13" "-" - - 10.56.0.8:80 10.20.0.2:56314 - -
```
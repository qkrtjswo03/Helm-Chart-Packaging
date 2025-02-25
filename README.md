



## 1. 테스트 환경
- Kind version : kind v0.26.0 go1.23.4 darwin/arm64
- kubernetes version : v1.30.2

---
| Name | ROLE |
| --- | --- |
| platform-monitoring-package-cluster-control-plane | control-plane |
| platform-monitoring-package-cluster-worker  | Worker |
| platform-monitoring-package-cluster-worker2 | Worker |




- Component List
---
| 컴포넌트 이름 | K8s 오브젝트 | 이미지 레지스트리 | 버전 | 용도 | 컨테이너 이미지 사이즈 |
| --- | --- | --- | --- | --- | --- |
| Prometheus | StatefulSet | quay.io/prometheus/prometheus | v2.55.1 | 모니터링 메트릭 수집 및 저장 | 285MB |
| Prometheus | StatefulSet | quay.io/prometheus-operator/prometheus-config-reloader | v0.78.2 | Prometheus 설정 변경 감지 및 자동 적용 | 43.1MB |
| Grafana | Deployment | docker.io/grafana/grafana | 11.3.1 | 수집된 메트릭을 시각화하는 대시보드  | 478MB |
| Grafana | Deployment | quay.io/kiwigrid/k8s-sidecar | 1.28.0 | ConfigMap을 감시하여 Grafana 대시보드 자동 업데이트 | 85.9MB |
| Node-Exporter | DaemonSet | quay.io/prometheus/node-exporter | v1.8.2 | 노드별 시스템 메트릭 수집 (CPU, MEM, DISK 등) | 22.7MB |
| Loki | StatefulSet | docker.io/grafana/loki | 2.9.4 | 로그 수집 및 저장 | 126MB |
| Loki | StatefulSet | kiwigrid/k8s-sidecar | 1.24.3 | ConfigMap 을 감시하여 Loki 의 설정을 자동 업데이트 | 85.9MB |
| Promtail | DaemonSet | docker.io/grafana/promtail | 3.0.0 | Loki 로 로그를 전송하는 에이전트 | 193MB |
| Kube-State-Metrics | Deployment | registry.k8s.io/kube-state-metrics/kube-state-metrics | v2.14.0 | kubernetes 리소스 상태 메트릭 제공 | 50MB |



---
## 1-1. Helm Chart 디렉토리 구조

```bash
kube-prometheus-stack/                        # 최상위 Helm Chart 디렉토리
├── charts/                                   # 서브차트들이 저장되는 디렉토리
│   ├── crds/                                 # Custom Resource Definitions (CRDs)
│   ├── grafana/                              # Grafana 서브차트 (압축 해제된 상태)
│   │   └── Chart.yaml                        # Grafana Helm Chart 메타데이터
│   ├── grafana-8.9.1.tgz                     # Grafana 서브차트 (압축 파일)
│   ├── kube-state-metrics/                   # kube-state-metrics 서브차트
│   │   └── Chart.yaml
│   ├── kube-state-metrics-5.29.0.tgz         # kube-state-metrics 서브차트 (압축 파일)
│   ├── loki/                                 # Loki 서브차트 (압축 해제된 상태)
│   │   └── Chart.yaml
│   ├── loki-5.43.0.tgz                       # Loki 서브차트 (압축 파일)
│   ├── prometheus-node-exporter/             # Node Exporter 서브차트
│   │   └── Chart.yaml
│   ├── prometheus-node-exporter-4.43.1.tgz   # Node Exporter 서브차트 (압축 파일)
│   ├── prometheus-windows-exporter/          # Windows Exporter 서브차트
│   │   └── Chart.yaml
│   ├── prometheus-windows-exporter-0.8.0.tgz # Windows Exporter 서브차트 (압축 파일)
│   ├── promtail/                             # Promtail 서브차트 (압축 해제된 상태)
│   │   └── Chart.yaml
│   └── promtail-6.15.0.tgz                   # Promtail 서브차트 (압축 파일)
├── templates/                                # Kubernetes 리소스 템플릿
│   ├── _helpers.tpl                          # 템플릿 헬퍼 함수
│   └── deployment.yaml                       # 예시: 배포 매니페스트
├── values.yaml                               # 기본 values 파일 (전체 스택 설정)
├── Chart.yaml                                # kube-prometheus-stack 메타데이터
├── .helmignore                               # 제외할 파일 목록
└── README.md                                 # 프로젝트 설명

```

---

# 2. 본 작업

## **2-1. 저장소 정보 가져오기 (Prometheus)**

- Helm의 공식 Grafana 차트를 사용할 거야. 먼저, Helm 리포를 추가하고 업데이트해.

```bash
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"grafana" has been added to your repositories

$ helm repo update
```

---

- Helm 차트 커스터마이징 할 수 있도록 새로운 디렉토리를 만들고 이동 (`prometheus-community/kube-prometheus-stack`)

```bash
$ helm pull prometheus-community/kube-prometheus-stack --untar
$ cd prometheus-community/kube-prometheus-stack

$ ls -al
total 0
drwxr-xr-x  7 user  staff  224  2 18 11:11 .
drwxr-xr-x  9 user  staff  288  2 18 11:37 ..
drwxr-xr-x  7 user  staff  224  2 18 11:11 crds
drwxr-xr-x  9 user  staff  288  2 18 11:36 grafana
drwxr-xr-x  7 user  staff  224  2 18 11:29 kube-state-metrics
drwxr-xr-x  8 user  staff  256  2 18 11:31 prometheus-node-exporter
drwxr-xr-x  7 user  staff  224  2 18 11:11 prometheus-windows-exporter
```

---

```bash
$ helm show values prometheus-community/kube-prometheus-stack > values.yaml
```

---

## 2-2. Helm Chart 패키징

```bash
$ helm package . 
Successfully packaged chart and saved it to: /Users/user/work/Helm/kube-prometheus-stack/kube-prometheus-stack-69.3.1.tgz
```

---

- 패키징 된 파일로 설치

```bash
$ helm install [릴리스 이름] kube-prometheus-stack-69.3.1.tgz --namespace monitoring --create-namespace

```

---

- JSON 출력으로 모든 Pod 이미지 확인

```bash
$ kubectl get pods -o jsonpath="{.items[*].spec.containers[*].image}" | tr ' ' '\n'
quay.io/prometheus/alertmanager:v0.28.0
quay.io/prometheus-operator/prometheus-config-reloader:v0.78.2
kiwigrid/k8s-sidecar:1.24.3
docker.io/grafana/loki:2.9.4
kiwigrid/k8s-sidecar:1.24.3
docker.io/grafana/loki:2.9.4
kiwigrid/k8s-sidecar:1.24.3
docker.io/grafana/loki:2.9.4
docker.io/grafana/loki-canary:2.9.4
docker.io/grafana/loki-canary:2.9.4
docker.io/grafana/loki-canary:2.9.4
docker.io/nginxinc/nginx-unprivileged:1.24-alpine
docker.io/grafana/loki:2.9.4
docker.io/grafana/loki:2.9.4
docker.io/grafana/loki:2.9.4
docker.io/grafana/loki:2.9.4
docker.io/grafana/loki:2.9.4
docker.io/grafana/loki:2.9.4
quay.io/prometheus/prometheus:v2.55.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.78.2
docker.io/grafana/agent-operator:v0.39.1
quay.io/kiwigrid/k8s-sidecar:1.28.0
quay.io/kiwigrid/k8s-sidecar:1.28.0
docker.io/grafana/grafana:11.3.1
registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.14.0
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.67.1
grafana/agent:v0.39.1
quay.io/prometheus/node-exporter:v1.8.2
quay.io/prometheus/node-exporter:v1.8.2
quay.io/prometheus/node-exporter:v1.8.2
quay.io/prometheus/node-exporter:v1.8.2
docker.io/grafana/promtail:3.0.0
docker.io/grafana/promtail:3.0.0
docker.io/grafana/promtail:3.0.0
docker.io/grafana/promtail:3.0.0
quay.io/prometheus-operator/prometheus-operator:v0.78.2
```

---

- `kube-prometheus-stack/Chart.yaml`  수정 (Loki & Promtail 추가)
    - 보통 서브차트는 원격 Helm Chart 레지스트리(Grafana Helm Repo 등)에서 가져오지만, `file://` 프로토콜을 사용하면 **로컬 파일 시스템의 특정 경로에서 서브차트를 가져온다.**

```bash
dependencies:
- condition: crds.enabled
  name: crds
  repository: ""
  version: 0.0.0
- condition: kubeStateMetrics.enabled
  name: kube-state-metrics
  repository: file://charts/kube-state-metrics
  version: 5.29.*
- condition: nodeExporter.enabled
  name: prometheus-node-exporter
  repository: file://charts/prometheus-node-exporter
  version: 4.43.*
- condition: grafana.enabled
  name: grafana
  repository: file://charts/grafana
  version: 8.9.*
- condition: windowsMonitoring.enabled
  name: prometheus-windows-exporter
  repository: file://charts/prometheus-windows-exporter
  version: 0.8.*
- condition: loki.enabled
  name: loki
  repository: file://charts/loki
  version: 5.43.0
- condition: promtail.enabled
  name: promtail
  repository: file://charts/promtail
  version: 6.15.0
```

위 설정에서 **각 필드의 의미**는 다음과 같아:

| 필드 | 설명 |
| --- | --- |
| `name` | 서브차트 이름 (`loki`) |
| `version` | 사용할 서브차트의 버전 (`5.43.0`) |
| `repository` | 서브차트의 위치 (`file://charts/loki` → 로컬 경로) |

즉, `charts/loki` 디렉터리에 `loki` 서브차트가 있어야 `helm dependency update`를 실행할 때 해당 서브차트를 올바르게 인식할 수 있어.

---

## **2-3. Helm 의존성 업데이트**

- `loki` 와 `Promtail` 이 Helm 차트에 추가 되었기때문에 `helm dependency update`를 실행해서 Loki와 Promtail 차트 가져오기

```bash
$ cd kube-prometheus-stack

$ helm dependency update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "karmada" chart repository
...Successfully got an update from the "grafana" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 7 charts
Dependency crds did not declare a repository. Assuming it exists in the charts directory
Downloading kube-state-metrics from repo https://prometheus-community.github.io/helm-charts
Downloading prometheus-node-exporter from repo https://prometheus-community.github.io/helm-charts
Downloading grafana from repo https://grafana.github.io/helm-charts
Downloading prometheus-windows-exporter from repo https://prometheus-community.github.io/helm-charts
Downloading loki from repo https://grafana.github.io/helm-charts
Downloading promtail from repo https://grafana.github.io/helm-charts
Deleting outdated charts
```

```bash
$ helm dependency list
NAME                       	VERSION	REPOSITORY                                        	STATUS
crds                       	0.0.0  	                                                  	unpacked
kube-state-metrics         	5.29.* 	https://prometheus-community.github.io/helm-charts	ok
prometheus-node-exporter   	4.43.* 	https://prometheus-community.github.io/helm-charts	ok
grafana                    	8.9.*  	https://grafana.github.io/helm-charts             	ok
prometheus-windows-exporter	0.8.*  	https://prometheus-community.github.io/helm-charts	ok
loki                       	5.43.0 	https://grafana.github.io/helm-charts             	ok
promtail                   	6.15.0 	https://grafana.github.io/helm-charts             	ok
```

---

- **Helm으로 배포**
    - Loki와 Promtail이 포함된 `kube-prometheus-stack`을 배포

```bash
$ helm package . 
$ helm install platform-monitoring kube-prometheus-stack-69.3.1.tgz --namespace monitoring --create-namespace
```

---

- Pod 상태 확인

```bash
$ watch -d 'kubectl get pods' 
NAME                                                     READY   STATUS    RESTARTS   AGE     IP            NODE                                              NOMINATED NODE   READINES
S GATES
alertmanager-platform-kube-prometheus-stac-alertmanager-0   2/2     Running   0          3h55m   10.244.1.55   platform-monitoring-package-cluster-worker3         <none>           <none>
loki-backend-0                                           2/2     Running   0          3h55m   10.244.1.57   platform-monitoring-package-cluster-worker3         <none>           <none>
loki-backend-1                                           2/2     Running   0          3h55m   10.244.3.64   platform-monitoring-package-cluster-worker          <none>           <none>
loki-backend-2                                           2/2     Running   0          3h55m   10.244.2.62   platform-monitoring-package-cluster-worker2         <none>           <none>
loki-canary-25689                                        1/1     Running   0          3h55m   10.244.1.48   platform-monitoring-package-cluster-worker3         <none>           <none>
loki-canary-bxg75                                        1/1     Running   0          3h55m   10.244.3.56   platform-monitoring-package-cluster-worker          <none>           <none>
loki-canary-kg5lk                                        1/1     Running   0          3h55m   10.244.2.53   platform-monitoring-package-cluster-worker2         <none>           <none>
loki-gateway-6dd67c775f-q92k6                            1/1     Running   0          3h55m   10.244.2.56   platform-monitoring-package-cluster-worker2         <none>           <none>
loki-read-56979dc7-4q2qf                                 1/1     Running   0          3h55m   10.244.2.59   platform-monitoring-package-cluster-worker2         <none>           <none>
loki-read-56979dc7-4zjwf                                 1/1     Running   0          3h55m   10.244.1.50   platform-monitoring-package-cluster-worker3         <none>           <none>
loki-read-56979dc7-w5mm6                                 1/1     Running   0          3h55m   10.244.3.58   platform-monitoring-package-cluster-worker          <none>           <none>
loki-write-0                                             1/1     Running   0          3h55m   10.244.2.54   platform-monitoring-package-cluster-worker2         <none>           <none>
loki-write-1                                             1/1     Running   0          3h55m   10.244.1.53   platform-monitoring-package-cluster-worker3         <none>           <none>
loki-write-2                                             1/1     Running   0          3h55m   10.244.3.61   platform-monitoring-package-cluster-worker          <none>           <none>
prometheus-platform-kube-prometheus-stac-prometheus-0       2/2     Running   0          3h55m   10.244.2.61   platform-monitoring-package-cluster-worker2         <none>           <none>
platform-grafana-884c8686c-njfqc                            3/3     Running   0          3h55m   10.244.3.60   platform-monitoring-package-cluster-worker          <none>           <none>
platform-grafana-agent-operator-6b6787c65-mqpx2             1/1     Running   0          3h55m   10.244.3.57   platform-monitoring-package-cluster-worker          <none>           <none>
platform-kube-prometheus-stac-operator-855fdbbdb4-x2wqg     1/1     Running   0          3h55m   10.244.2.55   platform-monitoring-package-cluster-worker2         <none>           <none>
platform-kube-state-metrics-77ff7dffb6-prftv                1/1     Running   0          3h55m   10.244.1.49   platform-monitoring-package-cluster-worker3         <none>           <none>
platform-loki-logs-64z92                                    2/2     Running   0          3h55m   10.244.1.54   platform-monitoring-package-cluster-worker3         <none>           <none>
platform-loki-logs-6s6zc                                    2/2     Running   0          3h55m   10.244.3.63   platform-monitoring-package-cluster-worker          <none>           <none>
platform-loki-logs-cgzft                                    2/2     Running   0          3h55m   10.244.2.60   platform-monitoring-package-cluster-worker2         <none>           <none>
platform-prometheus-node-exporter-2sdz2                     1/1     Running   0          3h55m   172.18.0.2    platform-monitoring-package-cluster-worker2         <none>           <none>
platform-prometheus-node-exporter-fq4rx                     1/1     Running   0          3h55m   172.18.0.4    platform-monitoring-package-cluster-worker3         <none>           <none>
platform-prometheus-node-exporter-h2wb7                     1/1     Running   0          3h55m   172.18.0.3    platform-monitoring-package-cluster-control-plane   <none>           <none>
platform-prometheus-node-exporter-wv56z                     1/1     Running   0          3h55m   172.18.0.5    platform-monitoring-package-cluster-worker          <none>           <none>
platform-promtail-2qcd8                                     1/1     Running   0          3h55m   10.244.2.58   platform-monitoring-package-cluster-worker2         <none>           <none>
platform-promtail-qplnh                                     1/1     Running   0          3h55m   10.244.0.14   platform-monitoring-package-cluster-control-plane   <none>           <none>
platform-promtail-sjxff                                     1/1     Running   0          3h55m   10.244.1.51   platform-monitoring-package-cluster-worker3         <none>           <none>
platform-promtail-svvg2                                     1/1     Running   0          3h55m   10.244.3.59   platform-monitoring-package-cluster-worker          <none>           <none>
```

---

## 2-4. 커스텀 Json 파일 불러오기

- 커스텀 한 Json 파일 : test-dasboard.json
- 기본적으로 아래와 같이 대시보드 리스트가 출력 됌.

![image.png](attachment:62b144f3-d9c1-4508-86b9-c4d6dfd4f8ae:image.png)

---

- Prometheus Helm Chart 내의 서브차트 Grafana 경로 : `kube-prometheus-stack/charts/grafana/`

```bash
$ ll
total 240
-rw-r--r--   1 user  staff   932B  2 24 17:33 Chart.yaml
-rw-r--r--   1 user  staff    59K  2 24 17:33 README.md
drwxr-xr-x  11 user  staff   352B  2 24 17:33 ci
drwxr-xr-x   4 user  staff   128B  2 25 10:26 dashboards. # 커스텀 Json 파일 or 자동으로 불러오고 싶은 Json 파일을 넣어준다.
drwxr-xr-x  36 user  staff   1.1K  2 24 17:33 templates
-rw-r--r--   1 user  staff    53K  2 25 10:42 values.yaml # 실제 옵션값 등을 설정하는 파일이다.
```

---

- vi values.yaml
    - sample yaml

```yaml
# 폴더 생성 할때 적용
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
    - name: 'test-dashboard' # 프로바이더의 고유 이름 (ConfigMap 이름과 관련됨)
      orgId: 1. 
      folder: TEST  # Grafana UI에서 대시보드가 저장될 폴더명
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards/test-dashboard   # Grafana 컨테이너 내부에서 대시보드를 찾을 경로
      
# JSON을 기본적으로 불러올때 적용  
dashboards: 
  test-dashboard: # ConfigMap의 이름이 될 값 (RFC 1123 규칙을 따르므로 소문자 사용)
    TEST:  # Grafana에서 폴더명 (위 `folder: TEST`와 매칭됨)
      file: dashboards/test/test-dashboard.json  # 대시보드 JSON 파일의 경로 (Helm Chart 내 상대 경로)
```

- 실제 yaml

```yaml
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - name: 'kubernetes-basic'
        orgId: 1
        folder: '1.Kubernetes Basic Component'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/kubernetes-basic
      - name: 'platform-component'
        orgId: 1
        folder: '2.Platform Component'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/platform-component
      - name: 'pv-and-pvc'
        orgId: 1
        folder: '3.PV And PVC'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/pv-and-pvc
      - name: 'read-write-io'
        orgId: 1
        folder: '4.Read,Write IO'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/read-write-io
      - name: 'log-monitoring'
        orgId: 1
        folder: '5.Log Monitoring'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/log-monitoring

dadashboards:
  kubernetes-basic:
    cni-cilium:
      file: dashboards/1_Kubernetes_basic_component/1_CNI_cilium/_Platform Monitoring - CNI Cilium-1736922127493.json
    kubernetes-components-etcd:
      file: dashboards/1_Kubernetes_basic_component/2_Kubernetes_Components&ETCD/Platform Monitoring - Kubernetes Components _ ETCD-1736922182325.json
    node:
      file: dashboards/1_Kubernetes_basic_component/3_Node/Platform Monitoring - Node-1736922218337.json
  platform-component:
    platform-service-monitoring:
      file: dashboards/2_Platform_Component/Platform-Service-Mornitoring-0114.json
  pv-and-pvc:
    persistent-volume:
      file: dashboards/3_PV_And_PVC/Platform Persistent Volume-1736831599206.json
  read-write-io:
    disk-io:
      file: dashboards/4_Read,Write_IO/Disk I_O-1736831654928.json
  log-monitoring:
    kubernetes-pod-log:
      file: dashboards/5_LogMonitoring/Kubernetes_Pod_Log/Kubernetes Pod Log-1736929150756.json
    pod-dashboard:
      file: dashboards/5_LogMonitoring/pod-dashboard-1736831697142.json
```

---

- value.yaml 수정 후 패키징

```yaml
# 패키징 
$ helm package .
Successfully packaged chart and saved it to: /Users/user/work/Helm/Helm-Chart-package/kube-prometheus-stack/charts/grafana/grafana-8.9.1.tgz

# tgz 파일 생성 된 거 확인 
$ ls | grep grafana-8.9.1.tgz
grafana-8.9.1.tgz

# 상위 경로로 이동
$ mv grafana-8.9.1.tgz ..

$ cd ..
$ ll
total 632
drwxr-xr-x   7 user  staff   224B  2 24 17:33 crds
drwxr-xr-x  10 user  staff   320B  2 25 11:09 grafana
-rw-r--r--   1 user  staff    55K  2 25 10:42 grafana-8.9.1.tgz
drwxr-xr-x   7 user  staff   224B  2 24 17:33 kube-state-metrics
-rw-r--r--   1 user  staff    14K  2 24 17:33 kube-state-metrics-5.29.0.tgz
drwxr-xr-x  13 user  staff   416B  2 24 17:33 loki
-rw-r--r--   1 user  staff   185K  2 24 17:33 loki-5.43.0.tgz
drwxr-xr-x   8 user  staff   256B  2 24 17:33 metrics-server
-rw-r--r--   1 user  staff   6.1K  2 24 17:33 metrics-server-3.8.0.tgz
drwxr-xr-x   8 user  staff   256B  2 24 17:33 prometheus-node-exporter
-rw-r--r--   1 user  staff    20K  2 24 17:33 prometheus-node-exporter-4.43.1.tgz
drwxr-xr-x   9 user  staff   288B  2 24 17:33 promtail
-rw-r--r--   1 user  staff    17K  2 24 17:33 promtail-6.15.0.tgz

# 이렇게 하는 이유는 helm chart 는 tgz 파일을 읽어들이기 떄문
```

---

- 최상위 경로(`kube-prometheus-stack`) 의 value.yaml 이 있는곳에서 helm install

```yaml
$ helm install prometheus . 
NAME: prometheus
LAST DEPLOYED: Tue Feb 25 11:12:20 2025
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
platform-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=prometheus"

Get Grafana 'admin' user password by running:

  kubectl --namespace monitoring get secrets prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Access Grafana local instance:

  export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=prometheus" -oname)
  kubectl --namespace monitoring port-forward $POD_NAME 3000

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.

# 헬름 설치 완료 및 조회 
$ helm list
NAME      	NAMESPACE 	REVISION	UPDATED                            	STATUS  	CHART                        	APP VERSION
prometheus	monitoring	1       	2025-02-25 11:12:20.66148 +0900 KST	deployed	platform-prometheus-stack-69.3.1	v0.80.0
```

---

- 그라파나 포트포워딩

```bash
$ kubectl get svc -n monitoring | grep grafana
prometheus-grafana                        ClusterIP   10.96.194.145   <none>        80/TCP                       95s

# grafana 포트가 80 이므로 3000으로 받아서 80으로 보내도록 
$ kubectl port-forward svc/prometheus-grafana 3000:80
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
```

---

- 그라파나 접속 후 대시보드 확인
    - TEST 폴더 생성 확인 및 하위에 JSON 파일 확인 가능

![image.png](attachment:5786fa88-8d39-461b-87e0-5c4eb66ff7f6:image.png)


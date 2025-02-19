---

 ##1. 테스트 환경

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
| Promtail | DaemonSet | docker.io/grafana/promtail | 3.0.0 | Loki 로 로그를 전송하는 에이전트 |
| Kube-State-Metrics | Deployment | registry.k8s.io/kube-state-metrics/kube-state-metrics | v2.14.0 | kubernetes 리소스 상태 메트릭 제공 |



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

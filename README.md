# deploy-history-manifest

`Deploy History` 프로젝트의 Kubernetes GitOps manifest 저장소입니다.

Jenkins가 애플리케이션 이미지를 빌드해서 Nexus Docker Registry에 push한 뒤, 이 저장소의 Helm `values.yaml` 이미지 태그를 수정합니다. Argo CD는 이 저장소를 감시하다가 변경사항을 Kubernetes 클러스터에 반영합니다.

## 구조

```text
.
├── argocd/
│   ├── application.yaml
│   └── monitoring-application.yaml
├── helm/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── backend-deployment.yaml
│       ├── backend-service.yaml
│       ├── frontend-deployment.yaml
│       ├── frontend-service.yaml
│       ├── postgres-deployment.yaml
│       ├── postgres-service.yaml
│       ├── ingress.yaml
│       └── _helpers.tpl
└── monitoring/
    └── values-monitoring.yaml
```

- `argocd/`: Deploy History와 monitoring을 배포하는 Argo CD Application 정의
- `helm/`: Deploy History 애플리케이션 Helm Chart
- `monitoring/`: `kube-prometheus-stack`의 Prometheus/Grafana 설정

## 배포 흐름

```text
Jenkins
-> Nexus Docker Registry
-> deploy-history-manifest
-> Argo CD
-> Kubernetes
```

Jenkins는 `helm/values.yaml`의 `image.backendTag`, `image.frontendTag`를 빌드한 이미지 태그로 수정합니다. Argo CD의 `deploy-history` Application은 `main` 브랜치의 `helm/`을 배포합니다.

현재 이미지 저장소 설정은 다음과 같습니다. 태그의 실제 값은 `helm/values.yaml`을 확인합니다.

```yaml
image:
  registry: nexus.deploy-history.local:8082
  namespace: deploy-history
  backendRepository: backend
  frontendRepository: frontend
```

이미지 경로는 `<registry>/<namespace>/<repository>:<tag>` 형식입니다. 두 Application 모두 자동 동기화, `prune`, `selfHeal`을 사용하고 대상 namespace를 자동 생성합니다.

## 애플리케이션 구성과 접속

| 구성 요소 | 기본 replica 수 | Service 포트 | 주요 설정 |
| --- | --- | --- | --- |
| Backend | 2 | 8080 | Spring profile `prod`, DB 사용자 `deploy_user` |
| Frontend | 2 | 80 | `/` 경로 제공 |
| PostgreSQL | 1 | 5432 | PostgreSQL 16, DB `deploy_history` |

애플리케이션 namespace는 `deploy-history`입니다. Backend는 PostgreSQL Service에 JDBC로 연결하며, Backend와 PostgreSQL Secret의 DB 비밀번호는 같아야 합니다.

Ingress는 `nginx` 클래스를 사용하며 **호스트 조건 없이** 요청 경로로 라우팅합니다.

| 경로 | 대상 |
| --- | --- |
| `/api` (`Prefix`) | Backend Service의 8080 포트 |
| `/` (`Prefix`) | Frontend Service의 80 포트 |

NGINX Ingress Controller가 설치되어 있어야 합니다. 컨트롤러가 노출된 IP 또는 도메인으로 접속하며, NodePort로 노출했다면 해당 포트도 지정합니다. 이 차트는 Ingress Controller 자체를 설치하지 않습니다.

Backend의 readiness/liveness probe는 각각 `/actuator/health/readiness`, `/actuator/health/liveness`를 사용합니다. Frontend는 두 probe 모두 `/`를 사용합니다.

현재 PostgreSQL Deployment에는 PVC나 데이터 볼륨이 설정되어 있지 않습니다. Pod가 재생성되면 데이터를 잃을 수 있으므로 영속 데이터가 필요하면 스토리지 구성이 필요합니다.

## Secret 관리

Secret 값은 이 저장소에 저장하지 않습니다.

비밀번호, JWT secret, Nexus 계정 정보는 Kubernetes 클러스터에 직접 생성합니다. 이 저장소에는 Secret 이름과 key 이름만 저장합니다.

```bash
kubectl create namespace deploy-history

kubectl -n deploy-history create secret generic backend-secret \
  --from-literal=DB_PASSWORD='replace-me' \
  --from-literal=JWT_SECRET='replace-me'

kubectl -n deploy-history create secret generic postgres-secret \
  --from-literal=POSTGRES_PASSWORD='replace-me'
```

현재 Backend와 Frontend는 `nexus-registry-secret`을 image pull secret으로 참조합니다. 배포 전에 같은 namespace에 생성합니다.

```bash
kubectl -n deploy-history create secret docker-registry nexus-registry-secret \
  --docker-server='nexus.deploy-history.local:8082' \
  --docker-username='replace-me' \
  --docker-password='replace-me'
```

각 노드에서 `nexus.deploy-history.local:8082`를 해석하고 Registry에 접근할 수 있어야 합니다. Registry 주소를 바꾸면 `helm/values.yaml`의 `image.registry`와 image pull secret의 서버 주소를 함께 맞춥니다.

## Argo CD 적용

Argo CD Application은 `argocd/`에 있으며, Application 리소스가 생성되는 namespace는 `argocd`입니다. Argo CD와 외부 values 저장소 참조를 지원하는 multi-source 기능을 사용할 수 있어야 합니다.

```yaml
repoURL: https://github.com/IMjaeyongpark/deploy-history-manifest.git
```

애플리케이션 Secret과 아래 모니터링 사전 조건을 준비한 뒤 적용합니다.

```bash
kubectl apply -f argocd/application.yaml
kubectl apply -f argocd/monitoring-application.yaml
```

## 모니터링

Monitoring Application은 Prometheus Community의 `kube-prometheus-stack` Helm Chart **88.6.1**과 이 저장소 `main` 브랜치의 `monitoring/values-monitoring.yaml`을 multi-source 방식으로 결합해 `monitoring` namespace에 배포합니다.

- Prometheus, Grafana, Alertmanager를 활성화합니다.
- Grafana는 NodePort `32000`으로 노출합니다. 노드 주소와 포트에 접근할 수 있는 환경에서 `http://<NODE_IP>:32000`으로 접속합니다.
- Grafana 기본 대시보드를 활성화하고, dashboard/datasource sidecar는 모든 namespace를 검색합니다.
- Prometheus 보존 기간은 `7d`이며 CPU 요청/제한은 `200m`/`500m`, 메모리 요청/제한은 `512Mi`/`1Gi`입니다.
- Grafana, Prometheus, Alertmanager, Prometheus Operator, kube-state-metrics는 `kubernetes.io/hostname: k8s-worker-1`인 노드에 배치합니다. 해당 라벨을 가진 노드가 필요하며, 다른 클러스터에서는 nodeSelector를 수정합니다.
- Prometheus Operator의 admission webhook과 patch 작업을 활성화합니다.

CRD 설치는 `helm.skipCrds: true`, `crds.enabled: false`로 비활성화되어 있습니다. **동기화 전에 해당 차트 버전과 호환되는 Prometheus Operator CRD를 별도로 설치해야 합니다.** CRD 설치·업데이트는 이 Application이 관리하지 않습니다.

Monitoring Application에는 `ServerSideApply=true`, `SkipDryRunOnMissingResource=true`가 설정되어 있습니다. `SkipDryRunOnMissingResource`는 리소스가 없는 경우의 dry-run을 건너뛰는 옵션이며, CRD 설치를 대신하지 않습니다.

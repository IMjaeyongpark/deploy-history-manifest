# deploy-history-manifest

`Deploy History` 프로젝트의 Kubernetes GitOps manifest 저장소입니다.

Jenkins가 애플리케이션 이미지를 빌드해서 Nexus Docker Registry에 push한 뒤, 이 저장소의 Helm `values.yaml` 이미지 태그를 수정합니다. Argo CD는 이 저장소를 감시하다가 변경사항을 Kubernetes 클러스터에 반영합니다.

## 구조

```text
.
├── argocd/
│   └── application.yaml
└── helm/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── backend-deployment.yaml
        ├── backend-service.yaml
        ├── frontend-deployment.yaml
        ├── frontend-service.yaml
        ├── postgres-deployment.yaml
        ├── postgres-service.yaml
        ├── postgres-pvc.yaml
        ├── ingress.yaml
        └── _helpers.tpl
```

## 배포 흐름

```text
Jenkins
-> Nexus Docker Registry
-> deploy-history-manifest
-> Argo CD
-> Kubernetes
```

Jenkins는 `helm/values.yaml`의 이미지 태그를 수정합니다.

```yaml
image:
  registry: NEXUS_REGISTRY_PLACEHOLDER
  namespace: deploy-history
  backendRepository: backend
  backendTag: latest
  frontendRepository: frontend
  frontendTag: latest
```

Argo CD는 변경된 Helm chart를 Kubernetes 클러스터에 동기화합니다.

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

Nexus Registry 인증이 필요하면 같은 namespace에 image pull secret을 생성합니다.

```bash
kubectl -n deploy-history create secret docker-registry nexus-registry-secret \
  --docker-server='NEXUS_REGISTRY_PLACEHOLDER' \
  --docker-username='replace-me' \
  --docker-password='replace-me'
```

## Registry 주소

Nexus 내부 IP는 GitHub에 올리지 않습니다.

`helm/values.yaml`에는 placeholder만 둡니다.

```yaml
image:
  registry: NEXUS_REGISTRY_PLACEHOLDER
```

실제 Nexus Registry 주소는 Argo CD 또는 클러스터 쪽 설정에서 주입합니다.

## Argo CD 적용

Argo CD Application은 `argocd/application.yaml`에 있습니다.

```yaml
repoURL: https://github.com/IMjaeyongpark/deploy-history-manifest.git
```

적용 명령:

```bash
kubectl apply -f argocd/application.yaml
```

# ArgoCD Example Project - App of Apps & Blue-Green Rollout

## 1. 환경 설정

### 1.1 ArgoCD 서버에 cluster-admin 권한 부여
아래 YAML 파일을 사용하여 ArgoCD 서버의 ServiceAccount에 cluster-admin 권한을 부여합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-cluster-admin
subjects:
- kind: ServiceAccount
  name: argocd-server
  namespace: argocd
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### 1.2 RoleBinding 확인
`argocd-rbac-cm` ConfigMap의 RoleBinding이 정확히 설정되었는지 확인합니다.

```bash
kubectl edit configmap argocd-rbac-cm -n argocd -o yaml
```

### 1.3 기타 설정
포트포워딩 및 ArgoCD CLI로 로그인합니다.

```bash
kubectl port-forward --address 127.0.0.1 svc/argocd-server -n argocd 8080:443
argocd login localhost:8080 --username admin --password <password>
```

애플리케이션 동기화 상태 확인 및 동기화 실행:
```bash
argocd app sync helm-guestbook
argocd app get apps-of-apps
```

---

## 1.4 배포를 위한 YAML 파일 구성
### 1.4.1 App of Apps 구성
ArgoCD App of Apps를 위한 YAML:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/wrjang96/argocd-test-apps.git
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 1.4.2 Blue-Green Rollout 구성
ArgoCD Rollout 애플리케이션을 위한 YAML:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: blue-green
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/wrjang96/argocd-test-apps.git
    targetRevision: HEAD
    path: blue-green
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 2. 주요 ArgoCD 명령어

### 2.1 애플리케이션 목록 조회
```bash
kubectl get applications -n argocd
```

### 2.2 애플리케이션 동기화
```bash
argocd app sync <application-name>
```

### 2.3 Replica 수 조절
아래 명령어로 Deployment 또는 Rollout의 Replica 수를 조정할 수 있습니다.

```bash
argocd app patch-resource <application-name> \
  --resource-name <resource-name> \
  --kind <Deployment/Rollout> \
  --namespace <namespace> \
  --patch '{"spec": {"replicas": <desired-count>}}' \
  --patch-type "application/merge-patch+json" \
  --project <argocd-project>
```

### 2.4 Project Sync Window 수정
특정 프로젝트에 대한 Sync Window를 설정합니다.

```bash
argocd proj windows add <project-name> \
  --kind deny \
  --schedule "* * * * *" \
  --duration "10m" \
  --timezone "Asia/Seoul"
```

> **참고**: Project 내의 Cluster > Namespace > Application 조건은 OR 조건으로 적용됩니다. 즉, 하나의 조건만 만족해도 적용됩니다.

### 2.5 Auto-Sync 활성화
특정 애플리케이션에 대해 Auto-Sync와 Self-Heal을 활성화합니다.

```bash
argocd app set <application-name> \
  --sync-policy automated \
  --self-heal \
  --auto-prune \
  --project <argocd-project>
```


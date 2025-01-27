https://github.com/argoproj/argocd-example-apps/tree/master

- Repo에 Argocd example Project 수정하여 정리함.
- App of Apps 구성 + Blue-Green(Rollout 구성) 및 명령어 정리

1. 환경 설정
1.1 argo-server sa에 cluster-admin 권한 rolebinding

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

1.2 Rolebinding 정확히 되어있는지 확인
kubectl edit configmap argocd-rbac-cm -n argocd -o yaml

1.3 기타 
kubectl port-forward --address 127.0.0.1 svc/argocd-server -n argocd 8080:443
argocd login localhost:8080 --username admin --password
argocd app sync helm-guestbook
argocd app get apps-of-apps

1.4 배포를 위한 yaml 파알 구성
- 해당 파일 배포시 apply -f 

1.4.1 App of Apps
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

1.4.2 Rollout
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


2. ArgoCD 명령어

kubectl get applications -n argocd

argocd app sync <application-name>

1. spec.Rpelica 수 조절
argocd app patch-resource <application-name> \
  --resource-name <resource-name> \
  --kind <Deployment/Rollout> \
  --namespace <ns> \
  --patch '{"spec": {"replicas": 0}}' \
  --patch-type "application/merge-patch+json" \
  --project <argocd-project>

2. Project Sync Window 수정

// 여기서 Project 내의 Cluster > NS >APPS 순으로 AND가 아닌 OR 조건 걸려서 하나만 만족해도 조건 걸림
argocd proj windows add helm-guestbook \
  --kind deny \
  --schedule "* * * * *" \
  --duration "10m" \
  --timezone "Asia/Seoul"

3. argocd로 특정 프로젝트의  enable auto-sync 진행해서 self-heal로 자동 스케쥴

argocd app set <application-name>
  --sync-policy automated
  --self-heal
  --auto-prune
  --project acc 


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



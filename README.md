# Kubernetes GitOps Practice

Argo CD로 동기화하는 Kubernetes 실습 저장소입니다. 현재 단계는 가장 작은
애플리케이션 단위인 **Namespace → Deployment → Service** 입니다.

## 디렉터리 구조

```text
apps/
└── hello-k8s/
    └── base/
        ├── namespace.yaml
        ├── configmap.yaml
        ├── deployment.yaml
        ├── service.yaml
        └── kustomization.yaml
```

## 1. 매니페스트 사전 확인

```bash
kubectl kustomize apps/hello-k8s/base
kubectl apply --dry-run=client -k apps/hello-k8s/base
```

> Argo CD가 배포를 담당할 예정이라면 실제 `apply`는 하지 않아도 됩니다.

## 2. Argo CD 애플리케이션 생성

Argo CD UI에서 새 Application을 만들고 다음 값을 사용합니다.

| 항목 | 값 |
| --- | --- |
| Application Name | `hello-k8s` |
| Project | `default` |
| Repository URL | 이 저장소의 GitHub HTTPS URL |
| Revision | `main` |
| Path | `apps/hello-k8s/base` |
| Destination cluster | `https://kubernetes.default.svc` |
| Namespace | `hello-k8s` |
| Sync policy | 처음에는 Manual 권장 |

생성 후 **Sync**를 실행합니다. `Namespace` 매니페스트가 포함되어 있으므로
`hello-k8s` 네임스페이스는 별도 생성할 필요가 없습니다.

## 3. 배포 확인

```bash
kubectl get all -n hello-k8s
kubectl get pods -n hello-k8s -w
kubectl port-forward -n hello-k8s service/hello-k8s 8080:80
curl http://localhost:8080
```

`Hello from Kubernetes!`가 표시되면 ConfigMap의 `index.html`이 NGINX에
정상적으로 mount된 것입니다.

## 4. Ingress로 접근

기존 NGINX Ingress Controller와 독립 호스트 `hello.k8s.core`를 사용합니다.
로컬 DNS 또는 hosts 파일에서 `hello.k8s.core`을 마스터 노드 IP로 연결해야 합니다.
이 환경의 HTTP/HTTPS NodePort는 각각 `31080`/`31443`입니다.

```bash
curl http://hello.k8s.core:31080/
curl https://hello.k8s.core:31443/
kubectl get ingress -n hello-k8s
```

로컬 인증서를 신뢰하지 않아 `curl`의 인증서 검증이 실패하면, 학습 환경에서만
`curl -k https://hello.k8s.core:31443/`로 확인합니다.

`hello.k8s.core`의 `/` 요청은 Ingress에서 `hello-k8s` Service에 직접
전달됩니다.

## 다음 실습 순서

1. ConfigMap으로 NGINX 응답 페이지 변경
2. Deployment replica와 rolling update 관찰
3. Service 유형과 내부 DNS 확인
4. Ingress 경로 rewrite와 host 기반 라우팅 확장

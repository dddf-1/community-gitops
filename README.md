# community-gitops

[howareyoujake](https://github.com/dddf-1/community_crud) 백엔드를 Kubernetes에 배포하는 Helm Chart 저장소입니다. ArgoCD가 이 저장소를 바라보고 있어서, 여기서 값을 바꾸면 클러스터에 자동으로 반영됩니다.

## 배포 흐름

1. 백엔드 저장소의 GitHub Actions가 테스트 후 이미지를 빌드해 사설 레지스트리에 push
2. 이 저장소 `values.yaml`의 `image.tag`를 새 버전으로 수정하고 커밋
3. ArgoCD가 변경을 감지해 클러스터에 롤링 배포

애플리케이션 코드와 배포 설정을 나눠서, 클러스터에 무엇이 떠 있는지 이 저장소의 커밋 기록만 보면 알 수 있게 했습니다.

## 구성

```text
helm/community-backend
├── Chart.yaml
├── values.yaml            # 이미지 태그, 복제 수, 리소스, 도메인
└── templates
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── configmap.yaml
```

| 리소스 | 설정 |
| --- | --- |
| Deployment | Pod 2개, 롤링 업데이트 `maxSurge 1`, `maxUnavailable 0`으로 교체 중에도 2개 유지 |
| Probe | `/health`로 startup(최대 150초 대기), readiness, liveness 확인 |
| 리소스 | 요청 CPU 250m, 메모리 512Mi / 상한 CPU 1, 메모리 1Gi |
| Service | ClusterIP 8080 |
| Ingress | nginx, `api.howareyoujake.com` |
| 설정 | DB 주소는 ConfigMap, DB 계정과 JWT 키는 클러스터에 미리 만든 Secret으로 주입 |

이미지는 사설 레지스트리(`registry.howareyoujake.com`)에서 받으며, 인증 정보도 Secret(`registry-credentials`)으로 둡니다. 비밀 값은 이 저장소에 넣지 않았습니다.

## 확인한 것

- 이미지 태그 v2 → v3 변경 커밋만으로 새 버전 배포
- 복제 수를 바꿨다가 되돌리는 커밋으로 ArgoCD 자동 동기화 확인

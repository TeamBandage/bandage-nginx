# bandage-nginx

FE/BE 리버스 프록시를 위한 Nginx 단독 배포 레포.
(기존에 FE 레포가 nginx 를 함께 배포하던 구조에서 분리)

## 구조

```
deploy/
├── docker-compose.yml      # nginx 단독 compose (bandage_default 네트워크에 external join)
└── nginx/
    ├── nginx.conf          # 전역 설정 (resolver 127.0.0.11 — 요청 시점 DNS 해석)
    └── conf.d/
        └── bandage.conf    # 라우팅: / → fe-web:3000, /api/v1/ 등 → bandage-band-manager:8080
```

- 이미지는 `nginx:1.27-alpine` 공식 이미지를 그대로 사용하고, conf 만 볼륨 마운트한다.
- 변수형 `proxy_pass` + resolver 조합이라 FE/BE 컨테이너가 없어도 nginx 는 기동한다 (해당 요청만 502).
- `/nginx-health` — nginx 자체 헬스 엔드포인트. 업스트림 생사와 무관하게 응답한다.

## 배포 (CD)

`develop` 브랜치에 `deploy/**` 변경이 push 되면 `.github/workflows/develop_deploy.yml` 이 실행된다.

1. **CI 게이트**: `nginx -t` 로 conf 문법 검증 — 잘못된 설정은 EC2 에 도달하기 전에 차단
2. `deploy/*` 를 EC2 `~/bandage-nginx/` 로 scp (FE 레포의 `~/bandage/` 와 분리)
3. EC2 에서 `docker compose up -d` (idempotent) → 컨테이너 내 `nginx -t` 통과 시에만 reload (무중단)
4. nginx 자체 헬스체크(`/nginx-health`)로 배포 성공 판정 — FE/BE 상태는 정보성 출력만

필요한 secrets: `EC2_HOST`, `EC2_SSH_KEY`

## 전제 조건

- BE compose 스택(`~/bandage`)이 먼저 떠 있어야 한다 — `bandage_default` 네트워크를 BE 프로젝트가 소유.
- FE 레포(bandage-fe-web)에서 nginx 서비스 / `deploy/nginx/` 제거가 후속으로 필요하다.
  정리 전까지는 FE 배포가 `~/bandage/nginx/` 에 conf 를 올리지만, 이 레포의 nginx 는
  `~/bandage-nginx/` 만 마운트하므로 동작에는 영향 없다.

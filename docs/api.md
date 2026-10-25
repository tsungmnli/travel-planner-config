# api 배포 가이드

사용자가 보는 화면(web)이 아니라, **데이터를 처리하는 서버**를 올리는 방법입니다.
공통 내용(DNS, 인증서, Cloudflare 토큰)은 [루트 README](../README.md)의 6장에 있습니다. **그 장을 먼저 끝내고** 이 문서를 따라 하세요.

## 1. api가 하는 일

- 주소: `https://travel.tsungmn.com/api/...`
- 실행되는 곳: 클러스터의 `api-deployment` 파드 (서비스 이름 `api-service`)
- 데이터를 저장하는 곳: **홈서버**의 PostgreSQL

```
요청 → ingress-nginx → api 파드 → 홈서버의 DB
```

`/api/readyz`는 "서버가 일할 준비가 되었나요?"라고 묻는 주소입니다. 이 주소는 DB에 실제로 연결해 보기 때문에, **DB까지 살아 있어야** `{"ok":true}`가 돌아옵니다. 그래서 세팅은 DB부터 합니다.

## 2. 관련 파일

| 파일 | 내용 |
| --- | --- |
| `base/api/` | api의 기본 설계도 (Deployment, Service, Ingress) |
| `overlays/api/configmap.yaml` | 일반 설정 (운영 환경 표시, 공개 주소, 메일 보내기 설정 등) |
| `overlays/api/secret.yaml` | Vault의 비밀값을 가져오는 연결 정보 |
| `overlays/api/kustomization.yaml` | 이미지 버전(`newTag`)과 도메인 설정 |

## 3. 처음 세팅하는 순서

### ① 홈서버에서 DB 켜기

api 저장소의 `docker-compose.db.yml`로 PostgreSQL을 실행합니다.

DB의 문(5432번 포트)은 **인터넷 전체에 열면 안 됩니다.** 비밀번호를 무작위로 찍어 보는 프로그램이 곧바로 찾아오기 때문입니다. 클러스터 서버의 IP만 허용하도록 방화벽을 설정합니다.

클러스터에서 DB 문이 열려 있는지 확인합니다.

```bash
kubectl run dbtest --image=postgres:17 --restart=Never --command -- \
  sh -c 'pg_isready -h <홈서버IP> -p 5432; echo "exit code: $?"'
kubectl logs dbtest        # accepting connections 가 나오면 성공
kubectl delete pod dbtest
```

### ② Vault에 비밀값 6개 넣기

| 이름 | 내용 |
| --- | --- |
| `DATABASE_URL` | `postgres://app:<비밀번호>@<홈서버IP>:5432/travel` |
| `SESSION_SECRET` | 32자 이상의 무작위 문자열 |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | Google 로그인용 값 |
| `GEOAPIFY_API_KEY` | 장소 검색용 키 |
| `SMTP_PASSWORD` | Resend의 API 키 (아래 ②-1 참고) |

이 중 하나라도 비어 있으면 서버는 일부러 시작하지 않습니다. 잘못된 설정으로 켜지는 것보다 안전하기 때문입니다.

**`SMTP_PASSWORD`는 `secret.yaml`을 push하기 전에 Vault에 먼저 넣으세요.** Vault에 없는 값을 가리키는 설정을 push하면 ArgoCD가 설정을 만들지 못해 배포가 멈춥니다.

### ②-1 메일 보내기 준비 (Resend)

회원가입 인증과 비밀번호 재설정 메일은 [Resend](https://resend.com)의 SMTP로 보냅니다.

1. Resend에 `tsungmn.com`을 등록하고, 알려 주는 DNS 값(SPF, DKIM)을 Cloudflare에 추가합니다. Resend 화면에서 `Verified`가 될 때까지 기다립니다.
2. Resend에서 API 키를 만듭니다. 발송 권한만 있는 키를 권장합니다.
3. API 키를 Vault의 `SMTP_PASSWORD`로 넣습니다.

나머지 값은 `overlays/api/configmap.yaml`에 이미 들어 있습니다.

| 이름 | 값 | 설명 |
| --- | --- | --- |
| `SMTP_HOST` | `smtp.resend.com` | Resend의 SMTP 서버 |
| `SMTP_PORT`, `SMTP_SECURITY` | `465`, `ssl` | 처음부터 암호화하는 방식. 587을 쓰려면 `starttls` |
| `SMTP_USER` | `resend` | Resend는 항상 이 이름입니다 |
| `SMTP_FROM` | `no-reply@tsungmn.com` | 보내는 주소. **Resend에 등록한 도메인의 주소**여야 합니다 |
| `SMTP_FROM_NAME` | `여행길` | 받는 사람 메일함에 보이는 보낸 사람 이름 |

Resend 무료 요금제는 **한 달 3,000통, 하루 100통**까지입니다.

### ③ ArgoCD로 배포하기

이 저장소를 push하면 ArgoCD가 `overlays/api`를 읽어 클러스터에 반영합니다. 파드가 떴는지 확인합니다.

```bash
kubectl -n travel-planner-prod get pods
kubectl -n travel-planner-prod logs deploy/api-deployment --tail=30
```

파드가 `Running`이면 성공입니다. 계속 다시 시작된다면 로그에서 **어떤 설정값이 비었는지** 알려 줍니다.

### ④ DB에 테이블 만들기 (한 번만)

```bash
kubectl -n travel-planner-prod exec deploy/api-deployment -- python -m app.migrate
```

`readyz`는 테이블이 없어도 통과합니다. 하지만 로그인이나 여행 저장 같은 실제 기능에는 테이블이 필요합니다.

**이메일 인증이 들어 있는 버전(마이그레이션 `008`)을 처음 올릴 때:**

- 새 파드가 `Running`이 된 것을 확인한 **직후에** 위 명령을 실행하세요. 새 코드는 `008`이 만드는 칸을 읽기 때문에, 실행하기 전에는 로그인과 가입이 오류가 납니다.
- `kubectl exec deploy/...`는 파드를 마음대로 고릅니다. 예전 파드에서 실행하면 `008` 파일이 없어서 적용되지 않으니, `kubectl get pods`로 새 파드만 남았는지 보고 실행하세요.
- 이미 가입한 계정은 이 마이그레이션이 **모두 인증 완료로 처리**합니다. 따로 할 일은 없습니다.

## 4. 잘 되었는지 확인하기

**1단계: 파드 안에서 직접 확인** (바깥 길과 상관없이 api와 DB만 시험합니다)

```bash
kubectl -n travel-planner-prod exec deploy/api-deployment -- \
  python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8000/api/readyz').read())"
```

`b'{"ok":true}'`가 나오면 api와 DB 연결이 끝난 것입니다.

**2단계: 바깥에서 확인**

```bash
curl -sS -i https://travel.tsungmn.com/api/readyz
```

`HTTP/2 200`과 `{"ok":true}`가 나오면 성공입니다. 서버와 다른 네트워크(집 PC, 휴대폰 데이터)에서도 시험하면 더 확실합니다.

## 5. 문제가 생겼을 때 (api)

| 증상 | 뜻 | 확인할 곳 |
| --- | --- | --- |
| 503 `{"ok":false}` | api는 켜졌지만 DB에 연결하지 못합니다. | DB 방화벽, `DATABASE_URL` |
| 파드가 계속 다시 시작됨 | 필수 설정값이 비어 있습니다. | `kubectl logs deploy/api-deployment` |
| nginx 화면의 502, 503 | api 파드가 떠 있지 않습니다. | `kubectl get pods` |
| nginx 화면의 404 | Ingress의 주소나 경로가 맞지 않습니다. | `kubectl get ingress` |

DNS, 인증서 문제는 [루트 README](../README.md)의 7장을 봅니다.

---

## 6. 보안 설정 (권장)

### ① 접속자 IP가 제대로 보이는지 확인

로그인, 가입, 메일 요청의 횟수 제한은 접속자 **IP별**로 셉니다. api는 앞단의 ingress-nginx가 알려 주는 접속자 주소(`X-Forwarded-For`의 맨 오른쪽 값)를 사용합니다. 이 값이 서버 내부 주소(10.x.x.x 같은 사설 주소)로 보이면 **모든 사용자가 한 사람처럼 세어져서** 서로의 제한에 걸립니다.

새 버전을 배포한 뒤 로그인을 한 번 시도하고 확인합니다.

```bash
kubectl -n travel-planner-prod logs deploy/api-deployment | grep clientip
```

- 아무것도 나오지 않으면 정상입니다.
- `클라이언트 IP 가 사설 주소` 경고가 나오면, 접속자 주소가 서버에서 바뀌어 들어오고 있는 것입니다. ingress-nginx가 접속자 주소를 그대로 받도록 바꿉니다.

```bash
kubectl -n ingress-nginx get svc        # ingress-nginx 서비스 이름 확인
kubectl -n ingress-nginx patch svc ingress-nginx-controller \
  -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

> 나중에 Cloudflare의 구름을 주황색(프록시)으로 바꾸면 접속자 주소가 한 단계 더 거쳐서 들어옵니다. 그때는 이 항목을 다시 봐야 하니 api 저장소의 `TRUSTED_PROXY_HOPS` 설명을 확인하세요.

### ② 봇 방지(Cloudflare Turnstile) 켜기 (선택)

가입, 비밀번호 찾기, 인증 메일 재전송에 "사람인지 확인"을 붙입니다. 평소에는 보이지 않고, 의심스러울 때만 확인 창이 뜹니다. **순서가 중요합니다.** api가 먼저 검사를 켜면, 아직 확인 위젯이 없는 웹에서는 가입이 전부 실패합니다.

1. Cloudflare 대시보드 → Turnstile → 위젯 추가. 도메인은 `travel.tsungmn.com`, 모드는 Managed로 합니다. **사이트 키**(공개)와 **시크릿 키**(비밀)가 나옵니다.
2. web 저장소의 GitHub 설정 → Secrets and variables → Actions → **Variables**에 `TURNSTILE_SITE_KEY`로 사이트 키를 넣고, web의 main을 다시 빌드합니다. (사이트 키는 빌드할 때 화면에 들어갑니다)
3. Vault의 `kv/travel-planner-tsungmn`에 `TURNSTILE_SECRET_KEY`로 시크릿 키를 넣습니다.
4. 이 저장소의 `overlays/api/secret.yaml`에 `TURNSTILE_SECRET_KEY` 줄을 추가해서 push합니다. **Vault에 값이 없는 상태에서 push하면 ArgoCD가 멈춥니다.** (②의 주의와 같습니다)
5. api가 새 값을 읽도록 다시 시작합니다.

```bash
kubectl -n travel-planner-prod rollout restart deploy/api-deployment
```

확인은 브라우저에서 가입을 한 번 해 보는 것입니다. 키를 넣기 전과 똑같이 동작해야 하고, 개발자 도구의 Network 탭에서 `register` 요청 본문에 `turnstile_token`이 보이면 켜진 것입니다. 문제가 생겨서 끄려면 Vault와 `secret.yaml`에서 `TURNSTILE_SECRET_KEY`를 지우고 api를 다시 시작하면 됩니다.

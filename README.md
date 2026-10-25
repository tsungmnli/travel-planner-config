# travel-planner-config

여행 플래너를 인터넷에 올리기 위한 **배포 설정 저장소**입니다.
이 저장소의 파일을 고쳐서 push하면, ArgoCD가 그 내용을 읽고 서버에 자동으로 반영합니다.

이 저장소는 두 부분으로 이루어져 있습니다.

| 부분 | 하는 일 | 문서 |
| --- | --- | --- |
| **api** | 데이터를 처리하는 서버 (`/api` 주소) | [docs/api.md](docs/api.md) |
| **web** | 사용자가 보는 화면 (`/` 주소) | [docs/web.md](docs/web.md) |

이 문서(루트 README)에는 **api와 web이 함께 쓰는 공통 내용**을 적었습니다.
처음이라면 이 문서의 6장(공통 세팅)을 먼저 끝낸 뒤, `docs/api.md`와 `docs/web.md`로 이동하세요.

---

## 1. 폴더 구조

```
base/api, base/web     서버의 기본 설계도 (Deployment, Service, Ingress)
overlays/api           api 전용 설정 (도메인, 비밀값 연결)
overlays/web           web 전용 설정 (도메인, 이미지 버전)
docs/                  api, web 각각의 상세 문서
```

---

## 2. 공통 세팅 (api, web 모두에 필요합니다)

### ① Cloudflare에서 주소 연결하기

Cloudflare 대시보드의 DNS 메뉴에서 레코드를 하나 추가합니다.

| 항목 | 값 |
| --- | --- |
| Type | A |
| Name | `travel` |
| 값 | ingress-nginx의 EXTERNAL-IP (`kubectl get svc -n ingress-nginx`로 확인) |
| 구름 아이콘 | 회색 (DNS only) |

구름을 회색으로 두는 이유는, Cloudflare가 중간에서 요청을 가로채지 않고 서버로 곧바로 보내게 하기 위해서입니다. 확인은 `nslookup travel.tsungmn.com`으로 합니다.

### ② Cloudflare 토큰 Secret 만들기

인증서 발급기(`ClusterIssuer letsencrypt-tsungmn`)는 이 저장소가 아니라 **`tsungmn-config` 저장소**에서 관리하고, skku-notice와 함께 씁니다.
Cloudflare 토큰 Secret도 그 저장소의 README를 따라 `cert-manager` 네임스페이스에 한 번만 만들면 됩니다. 이 저장소에서 따로 만들 것은 없습니다.

다른 컴퓨터(홈서버 등)에서 하려면 맨 아래 **부록 A**를 따릅니다.

### 인증서가 잘 만들어졌는지 확인

```bash
kubectl get clusterissuer                         # READY 가 True
kubectl -n travel-planner-prod get certificate    # READY 가 True
```

---

## 부록 A. 다른 컴퓨터(홈서버)에서 kubectl 사용하기

클러스터 서버(master)가 아닌 **다른 컴퓨터**에서 `kubectl`을 쓰는 방법입니다.

가장 중요한 원칙은 **master의 관리자 파일을 복사해 가지 않는 것**입니다. 그 파일은 클러스터 전체를 지울 수 있는 만능 열쇠입니다. 대신 *이 일만 할 수 있는 작은 출입증*을 새로 만들어 사용합니다.

필요한 것은 세 가지입니다. ① 도구(`kubectl`), ② 출입증(권한), ③ 길(네트워크 연결).

### A-1. master에서: 작은 출입증 만들기 (한 번만)

아래 계정은 `cert-manager` 안에서 Secret을 **만들 수만** 있습니다. 읽거나 지울 수는 없습니다.

```bash
kubectl -n cert-manager create serviceaccount secret-maker
kubectl -n cert-manager create role secret-creator --verb=create --resource=secrets
kubectl -n cert-manager create rolebinding secret-maker-bind --role=secret-creator --serviceaccount=cert-manager:secret-maker

# 24시간만 쓸 수 있는 출입증(토큰). 메모장에만 임시로 복사하고, 채팅이나 git에는 붙여넣지 않습니다.
kubectl -n cert-manager create token secret-maker --duration=24h

# 클러스터 주소 (https://<IP>:6443 모양)
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}{"\n"}'

# 클러스터 인증서 (-----BEGIN CERTIFICATE----- 로 시작하는 여러 줄)
kubectl config view --minify --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d
```

### A-2. 홈서버에서: 도구 설치와 길 확인

```bash
sudo apt update && sudo apt install -y curl ca-certificates
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# 길이 열려 있는지 확인합니다. 글자가 나오거나 Unauthorized 가 나오면 열려 있는 것입니다.
curl -k -m 5 https://<master서버IP>:6443/version
```

5초 뒤에도 아무 응답이 없으면 길이 막힌 것입니다. 이때는 이 방법을 쓸 수 없으니 master에서 직접 작업합니다.

### A-3. 홈서버에서: 인증서와 출입증 등록하기

A-1에서 복사한 인증서를 `ca.crt` 파일로 저장하고, kubectl에 등록합니다.

```bash
cat > ca.crt <<'EOF'
(여기에 -----BEGIN 부터 -----END CERTIFICATE----- 까지 붙여넣기)
EOF

kubectl config set-cluster travel --server=https://<master서버IP>:6443 --certificate-authority=ca.crt --embed-certs=true
kubectl config set-credentials secret-maker --token=<토큰>
kubectl config set-context travel --cluster=travel --user=secret-maker --namespace=cert-manager
kubectl config use-context travel

kubectl auth can-i create secrets      # yes 가 나오면 준비가 끝난 것입니다.
```

**자주 하는 실수:** 서버 주소에서 `https://`와 `:6443`을 빼먹으면, kubectl이 엉뚱하게 `http://...:80`을 두드려서 `connection refused`가 나옵니다. 80번 포트가 막힌 것이 아니라 주소가 틀린 것입니다. `kubectl config view --minify`의 `server:` 줄로 확인합니다.

### A-4. 홈서버에서: Cloudflare 토큰 Secret 만들기

```bash
read -s CF_TOKEN
printf %s "$CF_TOKEN" | kubectl create secret generic cloudflare-api-token --from-file=api-token=/dev/stdin
unset CF_TOKEN
```

이 출입증은 읽기 권한이 없으므로, 만들어졌는지는 master에서 확인합니다.

```bash
kubectl -n cert-manager get secret cloudflare-api-token
```

### A-5. 홈서버에서: 서비스 안쪽 시험

바깥 길(DNS)을 건너뛰고 서버 입구 IP로 바로 요청해서, Ingress와 인증서만 시험합니다.

```bash
curl -sS -i --resolve travel.tsungmn.com:443:<ingress-EXTERNAL-IP> https://travel.tsungmn.com/api/readyz
```

`HTTP/2 200`과 `{"ok":true}`가 나오면 앱, Ingress, 인증서가 모두 정상입니다.

### A-6. 작업이 끝난 뒤: 임시 출입증 지우기

홈서버에서:

```bash
rm -f ca.crt
kubectl config delete-context travel
kubectl config delete-cluster travel
kubectl config delete-user secret-maker
```

master에서:

```bash
kubectl -n cert-manager delete rolebinding secret-maker-bind
kubectl -n cert-manager delete role secret-creator
kubectl -n cert-manager delete serviceaccount secret-maker
```
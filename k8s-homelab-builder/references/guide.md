# 쿠버네티스 베어메탈 홈랩 구축 가이드 (2026 최신판)

이 가이드는 MetalLB, Longhorn, Bind9, External-DNS 및 Cert-Manager를 활용하여 클라우드 환경과 유사한 자동화된 인프라를 구축하는 과정을 다룹니다. 특히 내부망 전용 Root CA를 구축하여 개인 인증서를 발급하고 관리하는 방법이 포함되어 있습니다.

---

## 0. 사전 요구사항 (Prerequisites)

모든 워커 노드에는 다음의 패키지가 설치되어 있어야 합니다.

*   **OS**: Ubuntu 22.04+ 또는 Talos Linux 1.7+
*   **패키지**: `open-iscsi`, `nfs-common`, `bash`, `curl`, `jq`
*   **커널 모듈**: `nbd`, `iscsi_tcp`

### 필수 패키지 설치 (Ubuntu 기준)
```bash
sudo apt update && sudo apt install -y open-iscsi nfs-common jq
sudo systemctl enable --now iscsid
```

---

## 1. 네트워크 로드밸런서: MetalLB 설정

베어메탈 클러스터에서 `Type: LoadBalancer` 서비스를 사용하기 위해 MetalLB를 설치합니다.

### 1.1 설치
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

### 1.2 IP 주소 풀 및 L2 광고 설정
홈 네트워크 대역 중 DHCP와 겹치지 않는 정적 IP 범위를 지정합니다.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: home-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.250 # [중요] 사용자의 공유기 및 네트워크 환경에 맞게 수정 필수
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-adv
  namespace: metallb-system
spec:
  ipAddressPools:
  - home-pool
```

---

## 2. 분산 스토리지: Longhorn 설정

데이터 고가용성을 위해 모든 노드의 디스크를 하나로 묶습니다.

### 2.1 환경 검증
설치 전 모든 노드가 요구사항을 충족하는지 확인합니다.
```bash
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/scripts/environment_check.sh | bash
```

### 2.2 설치 및 기본 스토리지 클래스 지정
```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/deploy/longhorn.yaml

# 설치 후 기본 SC로 설정
kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}"
```

---

## 3. 내부 DNS 및 자동화: Bind9 & External-DNS

External-DNS가 내부 DNS 레코드를 자동으로 업데이트하려면 RFC 2136을 지원하는 DNS 서버가 필요합니다.

### 3.1 Bind9 DNS 서버 구축 (Docker 예시)
클러스터 외부(예: 시놀로지 NAS, 별도 라즈베리파이 등)에 Bind9을 설치한다고 가정합니다.

**1. TSIG 키 생성**
```bash
tsig-keygen -a HMAC-SHA256 externaldns-key
# 출력된 secret 값을 아래 설정 파일들에 복사해서 사용하세요.
```

**2. 설정 파일 작성 (`named.conf`)**
```bind9
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    listen-on { any; };
    forwarders { 8.8.8.8; 8.8.4.4; }; # 외부 도메인 쿼리용
};

key "externaldns-key" {
  algorithm hmac-sha256;
  secret "여기에_TSIG_키_Secret_입력";
};

zone "homelab.local" {
  type master;
  file "/var/lib/bind/db.homelab.local";
  allow-update { key "externaldns-key"; }; # 키를 가진 사용자만 업데이트 허용
};
```

**3. 초기 Zone 파일 작성 (`db.homelab.local`)**
```bind9
$TTL 604800
@       IN      SOA     homelab.local. root.homelab.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.homelab.local.
@       IN      A       192.168.1.53  ; DNS 서버 자신의 IP
ns      IN      A       192.168.1.53
```

**4. Bind9 실행**
```bash
docker run -d --name bind9 
  -p 53:53/udp -p 53:53/tcp 
  -v $(pwd)/named.conf:/etc/bind/named.conf 
  -v $(pwd)/db.homelab.local:/var/lib/bind/db.homelab.local 
  ubuntu/bind9
```

### 3.2 External-DNS 설치
인그레스 생성 시 Bind9에 레코드를 자동 등록합니다.

```bash
helm install external-dns external-dns/external-dns 
  --set provider=rfc2136 
  --set rfc2136.host=192.168.1.53 \ # [중요] 위에서 구축한 Bind9 DNS 서버의 IP 주소
  --set rfc2136.zone=homelab.local 
  --set rfc2136.tsigKeyname=externaldns-key 
  --set rfc2136.tsigSecret="여기에_TSIG_키_Secret_입력" 
  --set rfc2136.tsigAlgorithm=hmac-sha256
```

---

## 4. 프라이빗 인증 체계: Cert-Manager & Private Root CA

외부 Let's Encrypt 대신 내부망에서 신뢰할 수 있는 자체 Root CA를 구축하고 인증서를 발급하는 과정입니다.

### 4.1 Cert-Manager 설치
```bash
helm install cert-manager jetstack/cert-manager 
  --namespace cert-manager 
  --create-namespace 
  --set installCRDs=true
```

### 4.2 Root CA 부트스트래핑 (Self-Signed)
먼저 자신을 증명할 수 있는 Self-Signed Issuer를 만든 뒤, 이를 통해 실제 Root CA 인증서를 생성합니다.

```yaml
# 1. 임시 Self-Signed ClusterIssuer 생성
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
# 2. 실제 사용할 Root CA 인증서 생성 (Secret으로 저장됨)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: homelab-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "Homelab Root CA"
  secretName: homelab-ca-secret # 이 Secret에 ca.crt, tls.key가 저장됨
  privateKey:
    algorithm: RSA
    size: 2048
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
---
# 3. 위에서 생성된 Secret을 사용하는 내부 전용 CA ClusterIssuer 생성
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: homelab-ca-secret
```

### 4.3 Trust-Manager를 통한 인증서 배포 (Bundle)
생성된 Root CA를 모든 네임스페이스의 포드들이 신뢰할 수 있도록 `trust-manager`를 사용해 `ca-certificates.crt`를 배포합니다.

> **Note**: `apiVersion`은 설치 시점의 최신 버전(예: `v1beta1` 또는 `v1`)을 확인 후 사용하세요.

```bash
# trust-manager 설치
helm install trust-manager jetstack/trust-manager --namespace cert-manager

# 모든 네임스페이스에 Root CA 배포 설정
cat <<EOF | kubectl apply -f -
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: homelab-ca-bundle
spec:
  sources:
  - secret:
      name: homelab-ca-secret
      key: tls.crt
  target:
    configMap:
      key: "trust-bundle.pem" # 모든 NS에 이 ConfigMap이 생성됨
EOF
```

---

## 5. 통합 테스트 및 클라이언트 설정

### 5.1 내부 인증서 사용 인그레스 배포
이제 `internal-ca-issuer`를 사용하여 개인 인증서를 자동으로 발급받을 수 있습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: internal-app
  annotations:
    cert-manager.io/cluster-issuer: internal-ca-issuer
    external-dns.alpha.kubernetes.io/hostname: internal.homelab.local
spec:
  tls:
  - hosts:
    - internal.homelab.local
    secretName: internal-app-tls
  rules:
  - host: internal.homelab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### 5.2 클라이언트(PC/모바일) 신뢰 설정
개인 인증서이므로 사용자 PC 브라우저에서 '안전하지 않음' 경고가 뜨지 않게 하려면 Root CA를 설치해야 합니다.

1.  **CA 인증서 추출**:
    ```bash
    kubectl get secret homelab-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | base64 -d > homelab-ca.crt
    ```
2.  **Windows**: `certmgr.msc` 실행 -> '신뢰할 수 있는 루트 인증 기관'에 `homelab-ca.crt` 가져오기
3.  **macOS**: '키체인 접근' 앱 -> '시스템' -> 인증서 드래그 앤 드롭 후 '항상 신뢰' 설정
4.  **Linux**: `sudo cp homelab-ca.crt /usr/local/share/ca-certificates/` 후 `sudo update-ca-certificates` 실행

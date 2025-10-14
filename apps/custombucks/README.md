# Custombucks GitOps Deployment

이 디렉토리는 Custombucks 애플리케이션의 Kubernetes 배포 매니페스트를 포함합니다.

## 구조

```
custombucks/
├── namespace.yaml              # Namespace 정의
├── secret.yaml                 # Database 비밀번호 등 시크릿
├── backend-deployment.yaml     # Backend (Spring Boot) Deployment
├── backend-service.yaml        # Backend Service
├── frontend-deployment.yaml    # Frontend (Next.js) Deployment
├── frontend-service.yaml       # Frontend Service
└── ingress.yaml               # Frontend & Backend Ingress 설정
```

## 배포 전 설정 사항

### 1. Docker 이미지 빌드 및 푸시

#### Backend
```bash
cd custombucks/backend
./gradlew build
docker build -t your-registry/custombucks-backend:latest .
docker push your-registry/custombucks-backend:latest
```

#### Frontend
```bash
cd custombucks/frontend
docker build -t your-registry/custombucks-frontend:latest .
docker push your-registry/custombucks-frontend:latest
```

### 2. YAML 파일 수정

다음 파일들에서 `TODO` 마크가 있는 부분을 실제 값으로 변경해야 합니다:

#### backend-deployment.yaml
- `image`: 실제 Docker 이미지 레지스트리 주소로 변경
- `SPRING_DATASOURCE_URL`: MySQL 호스트가 다르면 변경
- 환경 변수 추가 (필요한 경우)

#### frontend-deployment.yaml
- `image`: 실제 Docker 이미지 레지스트리 주소로 변경
- `NEXT_PUBLIC_API_URL`: 실제 백엔드 API 도메인으로 변경
- 추가 환경 변수 설정 (필요한 경우)

#### ingress.yaml
- `custombucks.dailyapp.fun`: 실제 프론트엔드 도메인으로 변경
- `custombucks-api.dailyapp.fun`: 실제 백엔드 API 도메인으로 변경
- CORS origin 주소 변경

#### secret.yaml
- `db-password`: mysql-service의 비밀번호와 동일하게 설정 (현재: "0810")
- 다른 비밀번호를 사용하려면 mysql-service의 auth-secret도 함께 변경해야 함

### 3. 도메인 설정

DNS에서 다음 도메인들을 Kubernetes Ingress Controller의 IP로 설정:
- `custombucks.dailyapp.fun` → Frontend
- `custombucks-api.dailyapp.fun` → Backend API

## MySQL 데이터베이스 설정

이 애플리케이션은 `mysql-service` namespace의 MySQL을 사용합니다.

### custombucks 데이터베이스 생성

MySQL에 custombucks 데이터베이스를 생성하는 방법:

#### 방법 1: 자동 생성 (권장)
`mysql-service` 디렉토리에 `init-db-configmap.yaml`이 추가되어 있습니다.
MySQL이 처음 시작될 때 자동으로 custombucks 데이터베이스를 생성합니다.

```bash
kubectl apply -f ../mysql-service/init-db-configmap.yaml
kubectl rollout restart deployment/mysql -n mysql-service
```

#### 방법 2: 수동 생성
```bash
kubectl exec -it deployment/mysql -n mysql-service -- mysql -uroot -p0810 -e "CREATE DATABASE IF NOT EXISTS custombucks CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

### 비밀번호 설정
- custombucks의 `mysql-secret`은 mysql-service의 `auth-secret`과 동일한 비밀번호를 사용합니다
- 현재 비밀번호: `0810`
- 비밀번호 변경 시 두 secret을 모두 업데이트해야 합니다

## 배포 순서

ArgoCD의 sync-wave를 사용하여 다음 순서로 배포됩니다:

1. **Wave 0**: MySQL Service (mysql-service namespace)
2. **Wave 1**: Namespace & Secret (custombucks)
3. **Wave 2**: Backend Deployment & Service
4. **Wave 3**: Frontend Deployment & Service
5. **Wave 4**: Ingress Resources

## 수동 배포 (kubectl 사용)

ArgoCD를 사용하지 않고 수동으로 배포하려면:

```bash
# 1. MySQL 초기화 스크립트 적용 (처음 한 번만)
kubectl apply -f ../mysql-service/init-db-configmap.yaml
kubectl rollout restart deployment/mysql -n mysql-service

# 2. Namespace 생성
kubectl apply -f namespace.yaml

# 3. Secret 생성
kubectl apply -f secret.yaml

# 4. Backend 배포
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml

# 5. Frontend 배포
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml

# 6. Ingress 생성
kubectl apply -f ingress.yaml
```

## 확인

```bash
# Pod 상태 확인
kubectl get pods -n custombucks

# Service 확인
kubectl get svc -n custombucks

# Ingress 확인
kubectl get ingress -n custombucks

# 로그 확인
kubectl logs -f deployment/custombucks-backend -n custombucks
kubectl logs -f deployment/custombucks-frontend -n custombucks
```

## 트러블슈팅

### Backend가 시작되지 않는 경우
1. MySQL Service 연결 확인
2. Secret이 제대로 생성되었는지 확인
3. 환경 변수 확인

```bash
kubectl describe pod <pod-name> -n custombucks
kubectl logs <pod-name> -n custombucks
```

### Frontend가 Backend에 연결되지 않는 경우
1. `NEXT_PUBLIC_API_URL` 환경 변수 확인
2. CORS 설정 확인
3. Ingress가 제대로 설정되었는지 확인

### TLS 인증서 문제
cert-manager가 제대로 설치되어 있는지 확인:
```bash
kubectl get certificate -n custombucks
kubectl describe certificate custombucks-frontend-tls -n custombucks
kubectl describe certificate custombucks-backend-tls -n custombucks
```

## 리소스 요청사항

### Backend
- CPU: 250m ~ 500m
- Memory: 512Mi ~ 1Gi

### Frontend
- CPU: 100m ~ 250m
- Memory: 256Mi ~ 512Mi

## Health Checks

### Backend
- Liveness: `/actuator/health` (Port 8080)
- Readiness: `/actuator/health` (Port 8080)

### Frontend
- Liveness: `/` (Port 3000)
- Readiness: `/` (Port 3000)

## 업데이트

새로운 버전을 배포하려면:

```bash
# 이미지 빌드 및 푸시
docker build -t your-registry/custombucks-backend:v1.1.0 .
docker push your-registry/custombucks-backend:v1.1.0

# Deployment YAML 업데이트 (이미지 태그 변경)
# ArgoCD가 자동으로 감지하고 배포
```

또는 kubectl을 사용:
```bash
kubectl set image deployment/custombucks-backend custombucks-backend=your-registry/custombucks-backend:v1.1.0 -n custombucks
```

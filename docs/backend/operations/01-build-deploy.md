# 빌드 및 배포

---

## 빌드/배포 명령어

```bash
# Docker 이미지 빌드 & ECR Push
./infrastructure/scripts/build-and-push-to-ecr.sh

# EC2 배포 (기존 서버)
./infrastructure/scripts/deploy-to-ec2.sh <EC2_IP>

# 새 EC2 프로비저닝 + 배포
./infrastructure/aws/provision-and-deploy-ecr.sh
```

## 배포 워크플로우

```
코드 수정 → Docker 이미지 빌드 → ECR Push → EC2 배포
→ 자동 마이그레이션 + 헬스체크 → API: http://<IP>:8080/docs
```

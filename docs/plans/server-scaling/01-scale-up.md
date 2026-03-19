# 방안 A: 단일 서버 스케일업 (Scale-Up)

> 코드 변경 없이 즉시 적용 가능한 방안

---

## 개요

```
기존 EC2 → 더 큰 인스턴스 + 더 큰 디스크
```

| 항목 | 현재 | 변경 후 |
|---|---|---|
| 인스턴스 | t3.medium | t3.xlarge (4vCPU, 16GB RAM) |
| 디스크 | 96GB EBS | 200GB+ EBS |
| `max_concurrent_jobs` | 3 | 5~8 |
| 코드 변경 | — | **없음** |

---

## 장단점

| | 내용 |
|---|---|
| ✅ 장점 | 코드 수정 0, 즉시 적용 가능 |
| ❌ 단점 | 수직 확장 한계, 단일 장애점(SPOF) 해결 안 됨 |

---

## 즉시 적용 명령어

```bash
# EC2 인스턴스 타입 변경 (다운타임 5분)
aws ec2 stop-instances --instance-ids i-xxxxx
aws ec2 modify-instance-attribute --instance-id i-xxxxx --instance-type t3.xlarge
aws ec2 start-instances --instance-ids i-xxxxx

# EBS 볼륨 확장 (무중단)
aws ec2 modify-volume --volume-id vol-xxxxx --size 200
# EC2 내부에서:
sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1
```

이것만으로도 동시 크롤링 수를 **3 → 8개**까지 늘릴 수 있고, 디스크 여유도 확보됩니다.

# DB 관리

---

## 마이그레이션

```bash
# 상태 확인
alembic -c shared/alembic/alembic.ini current

# 적용
alembic -c shared/alembic/alembic.ini upgrade head

# Docker
docker-compose exec -T backend python -m alembic -c /app/alembic.ini upgrade head
```

## 백업/복원

```bash
# 백업
docker-compose exec db pg_dump -U vibe_user vibe_crawling_db > backup_$(date +%Y%m%d).sql

# 복원
cat backup.sql | docker-compose exec -T db psql -U vibe_user -d vibe_crawling_db
```

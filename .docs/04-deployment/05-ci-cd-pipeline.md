# CI/CD Pipeline (Placeholder — chưa triển khai)

## Kế hoạch đề xuất (khớp với hạ tầng ECS Fargate + ECR đã xác nhận)
1. Build: `mvn clean package -DskipTests` từng service (độc lập, không phụ thuộc chéo).
2. Test: chạy theo `docs/03-testing/01-testing-strategy.md` — Testcontainers cần Docker-in-Docker trên CI runner.
3. Docker build + push: build image theo Dockerfile mỗi service, tag theo git SHA, push lên Amazon ECR (1 repository/service).
4. Deploy: update ECS Task Definition (revision mới trỏ image tag mới) → `aws ecs update-service --force-new-deployment`.

## Chưa chốt
- CI tool: GitHub Actions / GitLab CI / CodePipeline — cần xác nhận trước khi viết file pipeline thật.
- Có cần staging environment (ECS Service riêng) trước khi deploy production không, hay deploy thẳng.

> File này sẽ cập nhật khi chọn xong CI tool cụ thể — hiện KHÔNG có pipeline chạy thật.
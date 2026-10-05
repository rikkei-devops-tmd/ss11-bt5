BAI 5: NOI JOB TU DONG BUILD VA PUSH DOCKER IMAGE SAU KHI TEST GRADLE XANH

1. MUC TIEU VA BOI CANH
Trong quy trinh CI/CD chuan doanh nghiep, pipeline duoc chia thanh nhieu giai doan (Stages / Jobs) doc lap: Stage 1 kiem tra chat luong ma nguon (Unit Tests, Packaging), Stage 2 dong goi Docker Image va xuat ban len Registry. Dieu kien tien quyet la Stage 2 chi duoc phep chay khi Stage 1 dat trang thai thanh cong (Success - Tich xanh). Tu khoa needs trong GitHub Actions cho phep thiet lap do thi phu thuoc (DAG) giua cac Jobs.

2. NGUYEN LY DIEU PHOI WORKFLOW VOI TU KHOA NEEDS (DAG PIPELINE)
- GitHub Actions mac dinh chay cac Job song song (parallel) neu khong co rang buoc.
- Khi khai bao needs: test trong Job docker-build:
  - Job docker-build se chuyen sang trang thai Waiting (Cho) trong khi Job test dang thuc thi.
  - Neu Job test ket thuc voi ma loi (Failed): Job docker-build se lap tuc bi danh dau la Skipped (Bo qua), ngan chan tuyet doi viec dong goi va phat hanh cac phien ban loi.
  - Neu Job test ket thuc thanh cong (Success): Job docker-build se duoc kick-off thuc thi tu dong.

3. CAU TRUC DOCKERFILE DONG GOI UNG DUNG
Tep Dockerfile toi uu su dung base image sieu nhe:

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

4. CAU HINH GITHUB ACTIONS WORKFLOW TOAN DIEN (.github/workflows/ci.yml)
Noi dung file ci.yml voi 2 Job lien ket chat che qua needs: test:

```yaml
name: Java Spring Boot CI/CD Pipeline (Gradle)

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read
  packages: write

jobs:
  test:
    name: Run Unit Tests and Build JAR
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Thực thi JUnit Test với Gradle
        run: ./gradlew test

      - name: Đóng gói tệp JAR với Gradle
        run: ./gradlew bootJar

      - name: Lưu trữ thành phẩm JAR Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar

  docker-build:
    name: Build and Push Docker Image
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Biên dịch tệp JAR
        run: ./gradlew bootJar

      - name: Thiết lập Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Đăng nhập vào GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN || secrets.GITHUB_TOKEN }}

      - name: Build và Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

5. QUY TRINH KIEM CHUNG VA DANH GIA PIPELINE
- Kiem tra bieu do truc quan (Workflow Visualization Graph):
  - Truy cap tab Actions tren GitHub Repository rikkei-devops-tmd/ss11-bt5.
  - Quan sat so do quy trinh: Co mui ten lien ket noi tu Job test sang Job docker-build.
- Kiem chung truong hop Pass (Happy Path):
  - Khi code chuan va tat ca JUnit test deu pass -> Job test xanh -> Job docker-build chay thanh cong va push image len GHCR tai dia chi ghcr.io/<owner>/<repo>:latest.
- Kiem chung truong hop Fail (Quality Gate Protection):
  - Khi co loi test trong ApplicationTests -> Job test chuyen sang mau do (Failed) -> Job docker-build lap tuc bi bo qua (Skipped). He thong duoc bao ve an toan.

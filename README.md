
# 🌟 Docker 이미지 최적화 가이드 🌟

목표는 Docker 이미지 최적화를 통해 더 작은 크기의 이미지로 빠른 배포와 향상된 보안성을 확보하는 것입니다. 이를 위해 여러 가지 최적화 기법을 차례로 실습할 것입니다.

---

## 🛠️ **1. Minimal Base Image - Alpine**

- **목표**: 가벼운 베이스 이미지를 선택하여 불필요한 라이브러리를 줄이고 이미지 크기를 최소화합니다.
- **해야 할 일**:
    - 기본적인 이미지로 `ubuntu`나 `debian`을 사용하는 대신, 더 작은 `alpine` 또는 `scratch` 같은 경량 이미지를 사용합니다.
    
    ```dockerfile
    FROM nginx:alpine
    ```

---

## 🛠️ **2. Minimal Base Image - Distroless**

- **목표**: 더 작은 베이스 이미지를 사용하여 Docker 이미지의 크기를 줄입니다.
- **해야 할 일**:
    - `ubuntu`나 `debian` 이미지를 사용하는 대신, 더 작은 `alpine` 또는 `distroless` 이미지를 사용합니다.
    
    ```dockerfile
    FROM gcr.io/distroless/static-debian11
    ```

---

## 📦 **3. Single Responsibility Principle 적용**

- **목표**: Docker 이미지가 단일 책임만을 가지도록 설계합니다.
- **해야 할 일**:
    - 여러 서비스를 하나의 Docker 이미지에 묶지 않고, 각 서비스별로 이미지를 분리.
    - 예: 웹 서버와 데이터베이스를 한 이미지로 만들지 않고, 별도의 이미지를 생성하여 Docker Compose로 관리합니다.

---

## 🏗️ **4. Multi-Stage Build 활용**

- **목표**: Multi-Stage Build를 사용해 빌드 환경과 실제 배포 환경을 분리하고 불필요한 파일을 제거하여 이미지 크기 감소합니다.
- **해야 할 일**:
    - `node:14-alpine`을 사용하여 먼저 애플리케이션을 빌드한 후, 최종 배포 이미지만 유지합니다.
    
    ```dockerfile
    # 빌드 단계
    FROM node:14-alpine as build
    WORKDIR /app
    COPY package*.json ./
    RUN npm install
    COPY . .
    RUN npm run build
    
    # 프로덕션 단계
    FROM node:14-alpine as production
    WORKDIR /app
    COPY --from=build /app/dist ./dist
    CMD ["npm", "start"]
    ```

---

## 📉 **5. 레이어 최소화**

- **목표**: 여러 명령어를 하나로 결합하여 레이어 수를 줄이고 빌드 속도를 향상시킵니다.
- **해야 할 일**:
    - 여러 `RUN` 명령어를 하나로 결합하여 레이어 수를 줄입니다.
    
    ```dockerfile
    RUN apt-get update && \
        apt-get install -y git && \
        apt-get clean && \
        rm -rf /var/lib/apt/lists/*
    ```

---

## 📂 **6. .dockerignore 파일 사용**

- **목표**: 불필요한 파일이 Docker 이미지에 포함되지 않도록 제외하여 빌드 성능 개선합니다.
- **해야 할 일**:
    - 프로젝트 루트에 `.dockerignore` 파일을 생성하고, 빌드에 필요 없는 파일들을 제외합니다.
    
    ```text
    node_modules
    .git
    Dockerfile
    .dockerignore
    README.md
    ```

---

## 🏷️ **7. 이미지 태그 고정**

- **목표**: 이미지를 안정적으로 재현 가능하게 만들기 위해 특정 태그를 사용합니다
- **해야 할 일**:
    - `latest` 태그 대신 구체적인 버전 태그를 사용하여 예측 가능한 환경을 만듭니다.
    
    ```dockerfile
    FROM nginx:1.19.6-alpine
    ```

---

## 🧑‍💻 **실습1 - Minimal Base Image**

### 실습 목표:
1. **Ubuntu** 이미지를 사용하여 Docker 이미지를 빌드
2. **Distroless** 이미지를 사용하여 Docker 이미지를 빌드
3. 두 이미지의 **크기**와 **레이어 수**를 비교

### Dockerfile_ubuntu (Ubuntu 기반):
```dockerfile
FROM openjdk:17-slim-buster

COPY . /usr/src/myapp
WORKDIR /usr/src/myapp

RUN apt-get update && \
    apt-get install -y git curl && \
    javac Main.java && \
    rm -rf /var/lib/apt/lists/*

CMD ["java", "Main"]
```

### Dockerfile_distroless (Distroless 기반):
```dockerfile
FROM openjdk:17-slim-buster AS build

COPY . /usr/src/myapp
WORKDIR /usr/src/myapp

RUN javac Main.java

FROM gcr.io/distroless/java17-debian11
WORKDIR /app
COPY --from=build /usr/src/myapp /app

CMD ["java", "Main"]
```

### **단계 1: Dockerfile 빌드**
```bash
docker build -t img_ubuntu -f Dockerfile_ubuntu .
docker build -t img_distroless -f Dockerfile_distroless .
```

### **단계 2: 이미지 크기 확인**
```bash
docker images
```
![15](https://github.com/user-attachments/assets/6ab87ca6-9318-4bc0-a9e8-815f4fc3155f)


---

## 🧑‍💻 **실습2 - 레이어 최소화**

### 실습 목표:
1. **레이어 최소화 없이**  Docker 이미지를 빌드
2. **레이어 최소화해**  Docker 이미지를 빌드
3. 두 이미지의 **크기**와 **레이어 수**를 비교

### Dockerfile_non_optimized (최적화되지 않은 버전):
```dockerfile
FROM openjdk:17-slim-buster

COPY . /usr/src/myapp
WORKDIR /usr/src/myapp

RUN apt-get update
RUN apt-get install -y git
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get install -y unzip
RUN apt-get install -y wget
RUN apt-get install -y nano
RUN apt-get install -y htop
RUN javac Main.java
RUN echo "Setup complete"
RUN rm -rf /var/lib/apt/lists/*

CMD ["java", "Main"]
```

### Dockerfile_optimized (최적화된 버전):
```dockerfile
FROM openjdk:17-slim-buster

COPY . /usr/src/myapp
WORKDIR /usr/src/myapp

RUN apt-get update && \
    apt-get install -y git curl vim unzip wget nano htop && \
    javac Main.java && \
    echo "Setup complete" && \
    rm -rf /var/lib/apt/lists/*

CMD ["java", "Main"]
```

---

### **단계 1: Dockerfile 빌드**
```bash
docker build -t img_non_optimized -f Dockerfile_non_optimized .
docker build -t img_optimized -f Dockerfile_optimized .
```

### **단계 2: 이미지 크기 확인**
```bash
docker images
```
![16](https://github.com/user-attachments/assets/69d85522-8faa-4f4a-be21-bba47d956dac)


# Docker Image Optimization

실서비스 두 프로젝트(ValanSe, Animal Diary)의 Dockerfile을 분석하고, JDK → JRE 전환 / non-root 실행 / 멀티스테이지 빌드를 단계적으로 적용하여 이미지 크기와 재빌드 속도를 개선하였다.

---

## 실험 결과 요약

| 프로젝트 | 개선 기법 | Before | After | 개선율 |
|----------|-----------|:------:|:-----:|:------:|
| ValanSe | JDK → JRE, non-root | 222.3 MB | 124.9 MB | **−43.8%** |
| Animal Diary | JDK → JRE, `--no-install-recommends` | 497.0 MB | 308.8 MB | **−37.9%** |
| Animal Diary | + 멀티스테이지 빌드 (Warm Rebuild) | 11,841 ms | 10,203 ms | **−13.8%** |

---

## 실험 1: ValanSe — JDK → JRE, non-root

ValanSe는 Spring Boot 3.2.5 기반 밸런스 게임 플랫폼으로, AWS EC2 + Docker + Nginx 구성으로 운영 중인 실서비스다.

### 문제 정의

| 항목 | Before 상태 | 문제 |
|------|------------|------|
| 베이스 이미지 | `eclipse-temurin:17-jdk-alpine` | CI에서 컴파일을 완료한 JAR을 실행하는 환경에 컴파일러·개발 도구 포함 |
| 실행 유저 | root (USER 미지정) | 컨테이너 프로세스가 root 권한으로 실행됨 |
| JAR 패턴 | `*SNAPSHOT.jar` | 릴리즈 빌드 시 파일명 불일치로 `COPY` 실패 가능성 존재 |

### Dockerfile

<details>
<summary>Before</summary>

```dockerfile
FROM eclipse-temurin:17-jdk-alpine
COPY ./build/libs/*SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar", "--spring.profiles.active=default"]
```

</details>

<details>
<summary>After</summary>

```dockerfile
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY ./build/libs/*.jar app.jar

RUN addgroup -S appgroup && \
    adduser -S appuser -G appgroup

USER appuser

ENTRYPOINT ["java", "-jar", "app.jar", "--spring.profiles.active=default"]
```

</details>

### 레이어 구조 변화

![Layer Structure Comparison](./images/layer-structure.svg)

JDK → JRE 전환으로 컴파일러·개발 도구 레이어가 제거된다. 최종 이미지에는 JVM 런타임과 애플리케이션 JAR만 포함된다.

### 벤치마크 결과

측정 조건: N=5, 베이스 이미지 Pull 포함 완전 초기화, Ubuntu 서버 환경

![Benchmark Results](./images/benchmark-results.svg)

| 항목 | Before (JDK) | After (JRE) | 변화 |
|------|:---:|:---:|:---:|
| 평균 빌드 시간 | 8,736 ms | 8,572 ms | −1.9% (오차 범위 내) |
| **이미지 크기** | **222.3 MB** | **124.9 MB** | **−43.8%** |

빌드 시간 차이는 통계적으로 유의미하지 않다. 두 Dockerfile 모두 실질적인 빌드 처리가 `COPY` 명령 하나이므로 레이어 처리량이 동일하기 때문이다.

---

## 실험 2: Animal Diary — FFmpeg 환경, 멀티스테이지 빌드

Animal Diary는 사진·동영상 처리를 위해 FFmpeg가 필요한 환경이다. FFmpeg는 설치 시 다수의 종속 패키지를 포함하는 대형 라이브러리로, 이미지 크기에 지배적인 영향을 미친다.

### 문제 정의

| 항목 | Before 상태 | 문제 |
|------|------------|------|
| 베이스 이미지 | `eclipse-temurin:17.0.12_7-jdk` | 실행 환경에 컴파일러 포함 |
| FFmpeg 설치 | `apt-get install -y ffmpeg` | 추천 패키지(recommends) 일괄 설치로 불필요한 패키지 포함 |
| JAR 레이어 | 단일 `COPY` | 소스 코드 변경 시 JAR 전체 레이어 무효화, 완전 재빌드 발생 |
| 실행 유저 | root (USER 미지정) | 컨테이너 프로세스가 root 권한으로 실행됨 |

### Dockerfile

<details>
<summary>Before — JDK, root 실행, 단일 JAR 레이어</summary>

```dockerfile
FROM eclipse-temurin:17.0.12_7-jdk
RUN apt-get update && \
    apt-get install -y ffmpeg && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
ARG JAR_FILE=/build/libs/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

</details>

<details>
<summary>After 1 — JRE, --no-install-recommends, non-root</summary>

```dockerfile
FROM eclipse-temurin:17.0.12_7-jre

WORKDIR /app

RUN apt-get update && \
    apt-get install -y --no-install-recommends ffmpeg && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

ARG JAR_FILE=/build/libs/*.jar
COPY ${JAR_FILE} app.jar

RUN groupadd -r appgroup && \
    useradd -r -g appgroup appuser

USER appuser

ENTRYPOINT ["java", "-jar", "/app.jar"]
```

</details>

<details>
<summary>After 2 — 멀티스테이지 빌드 + Spring Boot layertools</summary>

```dockerfile
# Stage 1: JAR 레이어 분리 (최종 이미지에 포함되지 않음)
FROM eclipse-temurin:17.0.12_7-jdk AS builder
WORKDIR /app
ARG JAR_FILE=/build/libs/*.jar
COPY ${JAR_FILE} app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# Stage 2: 실행 이미지 (배포 대상)
FROM eclipse-temurin:17.0.12_7-jre
RUN apt-get update && \
    apt-get install -y --no-install-recommends ffmpeg && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    groupadd -r appuser && useradd -r -g appuser appuser
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
USER appuser
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

</details>

### 레이어 구조 변화

![Layer Structure Comparison Exp2](./images/exp2-layer-structure.svg)

Spring Boot `layertools`는 JAR을 변경 빈도 기준으로 4개 레이어로 분리한다. 소스 코드 변경 시 `application/` 레이어만 교체되고, 용량이 큰 의존성 레이어 3개는 캐시를 재사용한다.

| 레이어 | 내용 | 변경 빈도 |
|--------|------|---------|
| `dependencies/` | 외부 라이브러리 (Spring, Jackson 등) | 낮음 |
| `spring-boot-loader/` | Spring Boot 실행기 | 매우 낮음 |
| `snapshot-dependencies/` | SNAPSHOT 의존성 | 낮음 |
| `application/` | 애플리케이션 소스 코드 | **높음** |

### 벤치마크 결과

**Phase 1: Cold Build** — 측정 조건: N=3, 베이스 이미지 Pull 포함 완전 초기화, Ubuntu 서버 환경

![Benchmark Results Exp2](./images/exp2-benchmark.svg)

| 항목 | Before (JDK) | After 1 (JRE) | After 2 (Multi-stage) |
|------|:---:|:---:|:---:|
| 평균 빌드 시간 | 212,644 ms | 130,845 ms | 120,472 ms |
| **이미지 크기** | **497.0 MB** | **308.8 MB** | **308.6 MB** |
| 크기 절감율 | 기준 | **−37.9%** | **−37.9%** |

After 1과 After 2의 이미지 크기가 거의 동일한 이유: FFmpeg가 이미지 크기의 약 180 MB를 차지하기 때문이다. 멀티스테이지 빌드의 효과는 이미지 크기가 아닌 재빌드 속도에서 나타난다.

**Phase 2: Warm Rebuild** — 애플리케이션 코드 변경 후 재빌드

![Warm Rebuild Comparison](./images/exp2-warm-rebuild.svg)

| 항목 | After 1 (단일 레이어) | After 2 (layertools 분리) | 개선율 |
|------|:---:|:---:|:---:|
| 재빌드 시간 | 11,841 ms | 10,203 ms | **−13.8%** |

After 1은 소스 코드가 변경되면 JAR 전체를 단일 레이어로 재처리한다. After 2는 `application/` 레이어만 교체하고 의존성 레이어 3개는 캐시를 재사용한다. CI/CD 파이프라인에서 배포 사이클마다 발생하는 재빌드에 직접 적용되는 개선이다.
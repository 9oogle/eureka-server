# Eureka Service Discovery 설정 가이드

팀원들이 각자의 클라이언트 서버에 Eureka를 적용할 수 있도록 작성된 가이드입니다.

---

## 목차

1. [서버 구성 개요](#1-서버-구성-개요)
2. [로컬 환경에서 서버 2개 실행하기](#2-로컬-환경에서-서버-2개-실행하기)
3. [Eureka Dashboard 확인](#3-eureka-dashboard-확인)
4. [클라이언트 서버에 Eureka 적용하기](#4-클라이언트-서버에-eureka-적용하기)
5. [자주 묻는 것들](#5-자주-묻는-것들)

---

## 1. 서버 구성 개요

이 프로젝트는 고가용성을 위해 Eureka 서버를 2개 운영합니다.  
두 서버는 서로를 peer로 등록하여, 한 쪽이 다운되어도 서비스 디스커버리가 유지됩니다.

```
[eureka-server-1 :9001] <--> [eureka-server-2 :9002]
         ^                            ^
    클라이언트 서비스들이 두 서버 모두에 등록
```

### 로컬 환경에서 환경변수가 필요 없는 이유

`${hostname1:localhost}`에서 `:localhost` 부분이 기본값입니다.  
환경변수 `hostname1`이 없으면 자동으로 `localhost`로 대체됩니다.

```yaml
eureka:
  instance:
    hostname: ${hostname1:localhost}                              # hostname1 없으면 localhost
  client:
    service-url:
      defaultZone: http://${hostname2:localhost}:9002/eureka/    # hostname2 없으면 localhost
```

운영/스테이징 환경에서는 실제 호스트명을 환경변수로 주입하면 됩니다.

---

## 2. 로컬 환경에서 서버 2개 실행하기
### IntellJ로 실행하는 경우
IntelliJ에서 Run Configuration을 2개 만들어 각각 다른 프로파일로 실행합니다.
1. 상단 메뉴 `Run > Edit Configurations` 클릭
2. 좌측 상단 `+` 버튼 클릭 후 `Spring Boot` 선택
3. 아래 표를 참고해 각각 설정 후 `Apply`

### Eureka Server 1
 
| 항목 | 값 |
|------|-----|
| Name | `EurekaServer-1` |
| Main class | `com.example.EurekaServerApplication` |
| VM options | `-Dspring.profiles.active=eureka1` |
| Port | `9001` |
 
### Eureka Server 2
 
| 항목 | 값 |
|------|-----|
| Name | `EurekaServer-2` |
| Main class | `com.example.EurekaServerApplication` |
| VM options | `-Dspring.profiles.active=eureka2` |
| Port | `9002` |
 
#### VM options 입력란이 보이지 않으면 `Modify options > Add VM options`를 클릭해 활성화합니다.
<img width="700" height="400" alt="image" src="https://github.com/user-attachments/assets/c7272a98-f289-4249-ae89-347ca405a34b" />


### CLI로 실행하는 경우

```bash
# 터미널 1
./gradlew bootRun --args='--spring.profiles.active=eureka1'

# 터미널 2
./gradlew bootRun --args='--spring.profiles.active=eureka2'
```
---

## 3. Eureka Dashboard 확인

서버 실행 후 아래 주소로 접속하면 등록된 서비스 목록을 확인할 수 있습니다.

| 서버 | URL |
|------|-----|
| Eureka Server 1 | http://localhost:9001 |
| Eureka Server 2 | http://localhost:9002 |

정상 실행 시 확인 사항:
- `Instances currently registered with Eureka` 섹션에 클라이언트 서비스 목록이 표시됨
- 두 서버가 서로를 `DS Replicas`로 인식하고 있는지 확인
<img width="800" height="120" alt="image" src="https://github.com/user-attachments/assets/296f92c3-35de-4548-a5d9-2b0b968b22a3" />


---

## 4. 클라이언트 서버에 Eureka 적용하기

### Step 1. 의존성 추가

**Gradle**

```gradle
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2023.0.0"
    }
}
```

Spring Cloud 버전은 본인의 Spring Boot 버전에 맞게 확인 후 적용하세요.  
참고: https://spring.io/projects/spring-cloud#overview

### Step 2. 어노테이션 추가

메인 애플리케이션 클래스에 `@EnableDiscoveryClient`를 추가합니다.

```java
@SpringBootApplication
@EnableDiscoveryClient
public class YourServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(YourServiceApplication.class, args);
    }
}
```

최신 Spring Cloud 버전에서는 자동 등록되지만, 명시적으로 선언하는 것을 권장합니다.

### Step 3. application.yml 설정

```yaml
spring:
  application:
    name: your-service-name   # Eureka에 표시될 서비스 이름 (필수)

eureka:
  instance:
    prefer-ip-address: true
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://${EUREKA_URL1:localhost}:9001/eureka/,http://${EUREKA_URL2:localhost}:9002/eureka/
```

`spring.application.name`은 반드시 설정해야 합니다.  
Eureka 대시보드와 서비스 간 통신에서 이 이름으로 식별됩니다.

### Step 4. 등록 확인

클라이언트 서버 실행 후 Eureka 대시보드(http://localhost:9001 또는 http://localhost:9002)에 접속합니다.  
`Instances currently registered with Eureka` 목록에 서비스 이름이 표시되면 등록 성공입니다.

로그에서도 확인할 수 있습니다.

```
DiscoveryClient_YOUR-SERVICE - registration status: 204
```

---

## 5. 자주 묻는 것들

**Q. 클라이언트가 Eureka에 등록되는 데 시간이 걸립니다.**  
기본적으로 등록 및 갱신에 최대 30~90초 정도 소요됩니다. 정상적인 동작입니다.

**Q. Eureka 서버 없이 클라이언트만 테스트하고 싶습니다.**  
`application.yml`에 아래를 추가하면 Eureka 연결 시도를 비활성화할 수 있습니다.

```yaml
eureka:
  client:
    enabled: false
```

**Q. 운영 환경에서 hostname은 어떻게 설정하나요?**  
`EUREKA_URL1`, `EUREKA_URL2` 환경변수를 실제 호스트명 또는 IP로 설정합니다.

```bash
export EUREKA_URL1=eureka-server-1.internal
export EUREKA_URL2=eureka-server-2.internal
```

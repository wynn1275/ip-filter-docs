# ip-filter 성능 테스트 결과
* concurrent thread 30 + duration 60 초 간 [ip-filter](https://github.com/wynn1275/ip-filter) 의 성능을 측정

## 테스트 환경
![test-env](https://raw.githubusercontent.com/wynn1275/ip-filter-performace-docs/develop/image/%ED%85%8C%EC%8A%A4%ED%8A%B8%ED%99%98%EA%B2%BD.PNG)

| 구분 | CPU | MEM | 대 수 |
| --- | --- | --- | --- |
| 부하발생 및 성능 지표 수집 | 2core | 4GB | 1대 |
| App 서버 (성능 측정 대상) | 4core | 32GB | 1대 |  

### 사용 도구
* scouter
  - 어플리케이션 성능 모니터링 도구로, 오픈소스이며 JVM(WAS, Standalone application) 에 대한 지표를 제공
  - scouter docs : [https://github.com/scouter-project/scouter/blob/master/README_kr.md](https://github.com/scouter-project/scouter/blob/master/README_kr.md)
* gatling
  - 서버 성능 테스트 도구로, scala 기반의 netty, akka 로 구현되어 있음 (오픈소스/엔터프라이즈 의 두 가지 버전 존재)
  - 웹서버의 continuous load testing 에 적합
  - gatling home : [https://gatling.io/open-source](https://gatling.io/open-source)

### gatling script
#### gatling scenario 파일 내용
```text
package test

import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class IpFilterSimulation extends Simulation {

  val feeder = csv("deny300-cidr32.csv").eager.random // deny300-cidr-32.csv 파일에서 random 하게 값을 추출하여 request 의 X-Forwarded-For 헤더 값에 사용함
  val httpProtocol = http
    .baseUrl("http://52.78.50.82:8080") // App 서버 URL
    .header("X-Forwarded-For", "${clientIp}")

  val scn = scenario("Test IP-Filter") // A scenario is a chain of requests and pauses
    .feed(feeder)
    .exec(
      http("req_deny300-cidr32")
        .get("/ipv4")
        .check(status.is(403), jsonPath("$.resultMessage").is("Deny")) // response 값 검증, 차단 IP 로 성능을 측정하므로 response 의 status_code 는 403, response body 의 resultMessage 는 "Deny" 일치 여부 검사 
    )

  setUp(scn.inject(constantConcurrentUsers(30) during(60 seconds)) // constantConcurrentUser(30): active request(session) 를 항상 30개로 일정하게 유지 
    .protocols(httpProtocol))
}
```
#### gatling resource (deny300-cidr32.csv) 파일 내용
```text
clientIp
0.0.0.1
0.0.0.2
0.0.0.3
0.0.0.4
0.0.0.5
0.0.0.6
0.0.0.7
0.0.0.8
0.0.0.9
0.0.0.10
0.0.0.11
0.0.0.12
...
```
* 성능 측정에 사용한 "deny300-cidr32.csv" 파일 내 clientIp 는 ip-filter 의 deny properties 파일에서 추출 (3000개 정도)

### 어플리케이션 실행 환경
```shell script
$JAVA_HOME/bin/java -jar -server -Xms28g -Xmx28g ip-filter-0.0.1-SNAPSHOT-limit.jar --spring.config.location=[300만개/3000만개 등록한 properties file]
```
* 3000만 개 차단 IP 목록 파일이 600MB 이므로, 로드 시에 많은 메모리 요구함
  * 실 서비스인 경우에는 properties 파일을 DB 로 분리하거나, Redis 와 같은 캐시 서버를 두어야 할 것으로 보임

## 테스트 결과 및 해석
### 차단 IP 목록 300만 개 등록 + concurrent 30 + 60초 동안 3회 수행 결과
* TPS : 2200 ~ 2300
* CPU : 20% ~ 40% 사용
* response time : 평균 14ms
![scouter-300-cidr32](https://raw.githubusercontent.com/wynn1275/ip-filter-performace-docs/develop/image/300-cidr32-final.png)
* 첫 번째 테스트: 21:38:10 ~ 21:39:10
![gatling-300-cidr32_1](https://raw.githubusercontent.com/wynn1275/ip-filter-performace-docs/develop/image/gatling-300-cidr32_1.PNG)
* 두 번째 테스트: 21:39:45 ~ 21:40:45
![gatling-300-cidr32_2](https://raw.githubusercontent.com/wynn1275/ip-filter-performace-docs/develop/image/gatling-300-cidr32_2.PNG)
* 세 번째 테스트: 21:41:30 ~ 21:42:30
![gatling-300-cidr32_3](https://raw.githubusercontent.com/wynn1275/ip-filter-performace-docs/develop/image/gatling-300-cidr32_3.PNG)


### 차단 IP 목록 3000만 개 등록 + concurrent 30 + 60초 동안 3회 수행 결과
* TPS : 2100 ~ 2200
* CPU : 15% ~ 50% 사용
* response time : 평균 15ms
![scouter-3000-cidr32](https://raw.githubusercontent.com/wynn1275/ip-filter-performace-docs/develop/image/3000-cidr32-final.png)
* 첫 번째 테스트: 20:49:15 ~ 20:50:15
![gatling-3000-cidr32_1](https://raw.githubusercontent.com/wynn1275/ip-filter-performace-docs/develop/image/gatling-3000-cidr32_1.PNG)
* 두 번째 테스트: 20:50:50 ~ 20:51:50
![gatling-3000-cidr32_2](https://raw.githubusercontent.com/wynn1275/ip-filter-performace-docs/develop/image/gatling-3000-cidr32_2.PNG)
* 세 번째 테스트: 20:52:42 ~ 20:53:42
![gatling-3000-cidr32_3](https://raw.githubusercontent.com/wynn1275/ip-filter-performace-docs/develop/image/gatling-3000-cidr32_3.PNG)

### 성능 테스트 결과 해석
* 300만 개 deny rule 등록과 3000만 개 deny rule 등록 시 response time 및 TPS 면에서는 큰 차이가 나지 않았습니다.
* 측정 간 CPU 사용률 및 TPS 그래프를 확인해보면, 현재 어플리케이션의 최대 성능은 아닌 것으로 생각합니다.
  * concurrent session 개수를 늘려 테스트한다면 더 높은 TPS 수치를 확인할 수 있을 것으로 보입니다.
* 성능테스트 간 실패 발생(connection timeout, 10만 request 중 3건 가량 발생)한 건에 대해서는 테스트 환경 간의 이슈, 혹은 GC 수행 간의 request 처리 미흡으로 발생한 건으로 보입니다.

# Spring Jsoup Recipe Crawler

- **Spring Boot**, **Jsoup** 기반 사이트 레시피 데이터 크롤러

## 크롤링 대상 사이트 목록

| 사이트        | Collection Name   | URL                         |
| ------------- | ----------------- | --------------------------- |
| 오 키친       | okitchen-data     | https://www.okitchen.co.kr  |
| 만개의 레시피 | recipe-10000-data | https://www.10000recipe.com |
| 메뉴판닷컴    | menupan-data      | https://www.menupan.com     |
| 삼양          | samyang-data      | https://m.serveq.co.kr      |
| 한식진흥원    | hansik-data       | https://www.hansik.or.kr    |

## 주요 기능

- **CSS Selector 기반 파싱**
  - `Jsoup` 활용 DOM 기반 탐색 및 데이터 추출
  - 사이트 별 고유 HTML 구조에 맞춰 `CSS Selector` 적용
- **`trigger.http` 활용**
  - 외부 툴 없이 IDE 내부에서 API 테스트 및 크롤링 실행 가능
  - `.http` 파일을 통한 API 명세 및 테스트 시나리오 공유 
- **설정 관리**
  - `application.yml`을 통해 Target URL, Collection Name, Selector 등을 중앙 제어
- **기능 별 아키텍쳐 분리**
  - 사이트별 독립 패키지(`hansik`, `menupan` 등) 구성
  - 개별 서비스(`Service Class`) 단위로 로직이 격리
- **공통 아키텍쳐 흐름**
  - **Indexing:** 목록 페이지에서 게시글 URL 수집
  - **수집:** 수집된 URL을 기반으로 상세 레시피 데이터 파싱 및 저장

## 기술 스택

- Java 17, Spring Boot 3.5.3
- MongoDB (Schemaless document store), Jsoup 1.21.1

## Configuration (`application.yml`)

```yaml
# 크롤링 사이트 설정 예시
recipe:
  sites:
    hansik:
      url: "https://www.hansik.or.kr/board/re/view/..."
      collection-name: "hansik-data"
  indexUrl:
    hansik:
      css-selector: "a.stretched-link" # 사이트 인덱스 관련 
```

## API Trigger & Usage

- Spring Web 환경에서 구동되며, HTTP 요청을 통해 크롤링 작업을 제어

### 1. IDE 활용 시 (Recommended)

`trigger.http` 파일 트리거 활용.

```http
### [한식진흥원] 레시피 크롤링 요청
POST http://localhost:8080/hansik/crawl

### [만개의레시피] URL 인덱스 수집 요청
POST http://localhost:8080/tenth/url
```

### 2. curl 명령어 활용

```bash
curl -X POST http://localhost:8080/hansik/crawl
```

## Project Structure

```text
src/main/java/HeoJin/crawling_spring
├── common          # Shared Config, Entity, Exception, Utils
├── hansik          # [한식진흥원] Crawler Implementation
├── menupan         # [메뉴판닷컴] Crawler Implementation
├── okitchen        # [오키친] Crawler Implementation
├── samyang         # [삼양] Crawler Implementation
└── tenth           # [만개의레시피] Crawler Implementation
```

## Data Schema 예시

```json
{
  "recipeName": "돼지고기 김치찌개",
  "ingredientList": [
    { "ingredient": "익은 김치 1/4포기" },
    { "ingredient": "돼지고기 200g" }
  ],
  "cookingOrderList": [
    { "step": 1, "instruction": "김치를 썬다." },
    { "step": 2, "instruction": "고기를 볶는다." }
  ],
  "sourceUrl": "https://www.hansik.or.kr/...",
  "siteIndex": "123"
}
```

## Database Configuration (`application-mongo.yaml`)

- `spring.profiles.active: mongo` 설정 시 활성화되는 MongoDB 연결 설정 예시 
- 해당 설정은 `src/main/java/HeoJin/crawling_spring/common/config/mongo/MongoConfig.java`에서 참조

```yaml
spring:
  data:
    mongodb:
      uri: "mongodb://<username>:<password>@<host>:<port>"
      database: "recipe_db"
```

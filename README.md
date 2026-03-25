# Spring Jsoup Recipe Crawler

- Spring Boot, Jsoup 기반 사이트 레시피 데이터 크롤러

## 크롤링 구현 사이트 목록

| 사이트        | Collection Name   | URL                         | 수집 방식             |
| ------------- | ----------------- | --------------------------- | --------------------- |
| 오 키친       | okitchen-data     | https://www.okitchen.co.kr  | 상세 URL 직접 파싱    |
| 만개의 레시피 | recipe-10000-data | https://www.10000recipe.com | URL 인덱스 기반 수집  |
| 메뉴판닷컴    | testMenuPan       | https://www.menupan.com     | 게시글 번호 루프 파싱 |
| 삼양          | testSamYang       | https://m.serveq.co.kr      | 카테고리별 상세 추출  |
| 한식진흥원    | testHansik        | https://www.hansik.or.kr    | 목록 페이지 순회 추출 |

## 크롤링 프로세스 

- **1. URL Indexing**: 대상 사이트의 목록 페이지를 순회하며 각 레시피의 상세 URL을 추출하여 MongoDB(`RecipeUrlIndex`)에 1차 저장
- **2. Data Parsing**: 저장된 URL 인덱스를 기반으로 상세 페이지에 접속하여 레시피 데이터(재료, 순서 등)를 추출 및 정제
- **3. Persistence**: 최종 가공된 데이터를 사이트별 전용 Collection에 저장하여 데이터 유실 방지 및 중복 체크 수행

## 주요 기능

### CSS Selector 기반 파싱

- `Jsoup`을 활용한 정적 HTML 데이터 파싱 및 추출
- 사이트 별 고유 HTML 구조에 맞춰 `CSS Selector` 적용

```java
// 예시: 클래스 및 태그 기반 정밀 타겟팅
Document doc = Jsoup.connect(url).get();
Element content = doc.select("div.content.detailBody").first();

// 텍스트 및 속성 추출
String recipeName = content.select("div.detailInfo h2").text();
String mainImg = content.select(".main_img img").attr("src");

// 목록 형태의 데이터 추출 (재료, 조리 순서 등)
Elements ingredients = content.select("div.ingredients p");
Elements steps = content.select("div.ContentArea p");

// 특정 텍스트를 포함하는 요소의 인접 요소 선택 (:contains 활용)
Element time = content.select("span:contains(조리시간) + h4").first();
```

### `trigger.http` 활용

- 외부 툴 없이 IDE 내부에서 API 테스트 및 크롤링 실행 가능
- `.http` 파일을 통한 API 명세 및 테스트 시나리오 공유 

```http
### [한식진흥원] 레시피 크롤링 요청
POST http://localhost:8080/api/crawling/hansik/data

### [만개의레시피] URL 인덱스 수집 요청
POST http://localhost:8080/api/crawling/tenthRecipes/urls?startPage=1&lastPage=265
```

### 설정 관리

- `application.yml` 기반 설정 관리: 타겟 URL, MongoDB 컬렉션명, CSS 셀렉터 등 중앙 제어

```yaml
# src/main/resources/application.yml
recipe:
  indexUrl:
    hansik:
      url: "https://www.hansik.or.kr/board/re/list/..."
      collection-name: "testHansik"
      css-selector: "a.stretched-link"
```

```java
// Java Service Layer (@Value 활용)
@Value("${recipe.indexUrl.hansik.url}")
private String baseUrl;

@Value("${recipe.indexUrl.hansik.css-selector}")
private String cssSelector;

// 설정값 기반으로 동적 크롤링 수행
crawlingUtil.crawlWithPagination(baseUrl, cssSelector, ...);
```


### 크롤링 부하 방지 및 지연 처리

- `Thread.sleep`을 이용한 요청 간격 조절로 대상 서버 부하 방지 및 차단 회피

```java
for (Long i = startIndex; i <= lastIndex; i++) {
    try {
        crawlSingleRecipe(baseUrl + i, i.intValue());
        Thread.sleep(500); // 0.5초 지연을 통한 Rate Limiting
    } catch (Exception e) {
        log.error("크롤링 실패 : 인덱스 -> {}, message : {} ", i, e.getMessage());
    }
}
```

### 데이터 정합성 및 복구

- 크롤링 중 누락되거나 실패한 데이터를 식별하여 재시도 수행
- `siteIndex`와 `hrefIndex`를 비교하여 유실된 레시피 데이터 자동 복구

```java
// TenthRecipeRecoveryService.java
public void recoveryData() {
    // 1. 수집된 데이터 인덱스 로드
    HashSet<String> dataIndex = siteIndexList.stream()...
    
    // 2. 전체 URL 인덱스와 대조하여 미수집 항목 식별
    for (Object object : hrefIndexList) {
        if (!dataIndex.contains(hrefIndex)) {
            // 3. 미수집 데이터에 대해 재수집 수행
            tenthRecipeService.crawlSingleRecipe(sourceUrl, hrefIndex);
        }
    }
}
```

### 예외 처리 및 안정성

- `Optional` 및 개별 `try-catch` 설계를 통해 특정 요소 누락 시에도 전체 프로세스 유지

```java
// Optional을 활용한 필수 요소 검증
Element content = Optional.ofNullable(doc.select("div.content.detailBody").first())
        .orElseThrow(() -> new CustomException("존재하지 않는 페이지"));

// 부분적 데이터 누락에 대한 예외 처리
try {
    minutes = Integer.parseInt(timeElement.text().replaceAll("[^0-9]", ""));
} catch (Exception e) {
    log.error("조리시간 파싱 실패, 기본값(0) 적용");
    minutes = 0; 
}
```

### 크롤링 데이터 스키마 예시

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


## 기술 스택

- Java 17, Spring Boot 3.5.3, Spring Data MongoDB
- MongoDB (NoSQL), Jsoup 1.21.1 (HTML Parsing Library)
- Lombok, Gradle, Git, GitHub Actions (CI/CD)
- JUnit 5, AssertJ (Unit & Integration Testing)

## 프로젝트 구조

```text
src/
├── main/
│   ├── java/.../crawling_spring/
│   │   ├── common/      # 공통 구성 (Config, Entity, Exception, Util)
│   │   ├── hansik/      # 한식 레시피 관련 서비스
│   │   ├── menupan/     # 메뉴판 레시피 관련 서비스
│   │   ├── okitchen/    # 오키친 레시피 관련 서비스
│   │   ├── samyang/     # 삼양 레시피 관련 서비스
│   │   └── tenth/       # Tenth 레시피 관련 서비스 및 복구
│   └── resources/
│       └── application.yml # 설정 파일
└── test/                # 단위 및 통합 테스트
```

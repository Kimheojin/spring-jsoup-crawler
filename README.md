# Spring Jsoup Recipe Crawler

- Spring Boot, Jsoup 기반 사이트 레시피 데이터 크롤러

## 크롤링 대상 사이트 목록

| 사이트        | Collection Name   | URL                         |
| ------------- | ----------------- | --------------------------- |
| 오 키친       | okitchen-data     | https://www.okitchen.co.kr  |
| 만개의 레시피 | recipe-10000-data | https://www.10000recipe.com |
| 메뉴판닷컴    | menupan-data      | https://www.menupan.com     |
| 삼양          | samyang-data      | https://m.serveq.co.kr      |
| 한식진흥원    | hansik-data       | https://www.hansik.or.kr    |

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

- `application.yml`을 통해 Target URL, Collection Name, Selector 등을 중앙 제어

```yaml
# src/main/resources/application.yml
recipe:
  indexUrl:
    hansik:
      url: "https://www.hansik.or.kr/board/re/list/..."
      collection-name: "hansik-data"
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

### 기능 별 아키텍쳐 분리
- 사이트별 독립 패키지 구성을 통해 도메인 로직 격리 및 확장성 확보

```text
src/main/java/.../crawling_spring
├── hansik
│   ├── controller
│   ├── dto
│   └── service
├── menupan
│   ├── controller
│   └── service
├── okitchen
│   ├── controller
│   └── service
└── ... (각 사이트별 동일 구조 적용)
```

### 공통 아키텍쳐 흐름

- 목록 페이지에서 게시글 URL 수집

```java
public void crawlRecipeUrls(int startPage, int endPage) {
    for (int page = startPage; page <= endPage; page++) {
        Document doc = Jsoup.connect(url + page).get();
        Elements links = doc.select(cssSelector);
        
        for (Element link : links) {
            String recipeUrl = link.attr("abs:href");
            mongoTemplate.save(new RecipeUrlIndex(recipeUrl), collectionName);
        }
    }
}
```

- 수집된 URL을 기반으로 상세 레시피 데이터 파싱 및 저장

```java
public void crawlSingleRecipe(String recipeUrl) {
    Document doc = Jsoup.connect(recipeUrl).get();
    
    String name = doc.select("h2.title").text();
    List<Ingredient> ingredients = parseIngredients(doc);
    List<CookingOrder> steps = parseSteps(doc);

    Recipe recipe = Recipe.builder()
            .recipeName(name)
            .ingredientList(ingredients)
            .cookingOrderList(steps)
            .sourceUrl(recipeUrl)
            .build();
            
    mongoTemplate.save(recipe, collectionName);
}
```

### Polite Crawling (Rate Limiting)
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

### 데이터 정제 및 전처리 (Data Cleaning)
- 정규표현식 및 문자열 파싱을 통한 로우 데이터 가공
```java
// 숫자 외 문자 제거 및 정수 파싱
String timeStr = timeElement.text().replaceAll("[^0-9]", "");
int minutes = Integer.parseInt(timeStr);

// 쉼표 구분자 기반 재료 데이터 분리 및 정제
List<Ingredient> ingredients = content.select("div.ingredients p")
        .stream()
        .flatMap(text -> Arrays.stream(text.split(",")))
        .map(String::trim)
        .collect(Collectors.toList());
```

### 예외 처리 및 안정성 (Robustness)
- `Optional` 및 개별 `try-catch` 설계를 통해 특정 요소 누락 시에도 전체 프로세스 유지
```java
// Optional을 활용한 필수 요소 검증
Element content = Optional.ofNullable(doc.select("div.content.detailBody").first())
        .orElseThrow(() -> new CustomException("존재하지 않는 페이지"));

// 부분적 데이터 누락에 대한 예외 처리 (Graceful Degradation)
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

- Java 17, Spring Boot 3.5.3
- MongoDB, Jsoup 1.21.1

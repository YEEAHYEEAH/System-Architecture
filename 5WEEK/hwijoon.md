# 8장. URL 단축기 설계

URL 단축기란?

긴 URL을 짧고 간결한 링크로 변환해주는 것.

### API 엔드포인트

1. URL 단축용 엔드포인트 : 새 단축 URL을 생성하고자 하는 클라이언트는 이 엔드포인트에 단축할 URL을 인자로 실어서 POST 요청을 보내야 한다.
    - POST  /api/v1/data/shorten
        - 인자 : {longUrl: longURLstring}
        - 반환 : 단축 URL
2. URL 리디렉션용 엔드포인트 : 단축 URL에 대해서 HTTP 요청이 오면 원래 URL로 보내주기 위한 용도의 엔드포인트. 다음과 같은 형태를 띈다.
    - GET /api/v1/shortUrl
        - 반환 : HTTP 리디렉션 목적지가 될 원래 URL

단축 URL을 받은 서버는 그 URL을 원래 URL로 바꾸어서 301 응답의 Location 헤더에 넣어 반환한다.

### 301 응답 302 응답 차이

301응답과 302 응답은 모두 클라이언트를 다른 URL로 이동시키는 리디렉션 상태 코드이지만 브라우저 및 검색 엔진의 동작이 다르다.

| 상태 코드 | 설명 | 캐싱 여부 | SEO 영향 |
| --- | --- | --- | --- |
| **301 Moved Permanently** | 원래 URL이 영구적으로 변경됨을 알림. 이후 같은 요청 시 새 URL로 접근해야 함. | **예 (영구 캐싱)** | **기존 페이지의 SEO 점수를 새 URL로 전달** |
| **302 Found (Temporary Redirect)** | 원래 URL이 일시적으로 다른 URL로 리디렉션됨. 이후에도 원래 URL을 유지해야 함. | **아니오 (캐싱 X)** | **SEO 점수가 원래 URL에 유지됨** |

**서버 부하를 줄이려면 301**

- 301은 브라우저가 리디렉션을 캐싱하기 때문에, 사용자가 동일한 단축 URL을 여러번 방문해도 서버에 계속 요청하지 않는다.
- 한 번만 서버에서 원래 URL을 조회하면, 브라우저가 캐싱된 URL을 직접 방문하게 되므로 데이터베이스 조회 부하가 줄어든다.

트래픽 분석하려면 302

- 302는 브라우저가 캐싱하지 않는다. 즉, 사용자가 단축 URL을 방문할 때마다 서버에 요청이 들어온다.
- 따라서 모든 클릭 수를 서버에서 기록할 수 있어, 정확한 트래픽 분석이 가능하다.

### 단축 URL을 통해 트래픽을 분석하는 이유

- 원본 URL은 내가 소유한게 아닐 수 있다.
    - 단축 URL은 여러 웹사이트로 연결될 수 있다.
    - 예를들어, `short.ly/xyz` 를 클릭했을 때 네이버, 유튜브, 블로그 등 다양한 사이트로 연결시킬 수 있다.
    - 하지만 원본 URL에서 직접 분석한다면, 해당 사이트의 관리자만 트래픽 데이터를 볼 수 있다.
    - 하지만 단축 URL에서 리디렉션하기 전에 기록하면 모든 트래픽 데이터를 내가 수집할 수 있다.
- 클릭한 사용자의 데이터 수집 가능
    - 단순히 방문 횟수뿐만 아니라, 어떤 사람이, 언제, 어떤 환경에서 클릭했는지를 추적할 수 있다.
- A/B 테스트 및 마케팅 분석
    - 예를 들어, 같은 페이지로 연결되지만 서로 다른 광고 캠페인(예: 페이스북 vs 트위터)에서 단축 URL을 사용하면, 어떤 채널에서 더 많은 트래픽이 발생하는지 분석 가능하다.

### 해시 함수

원래 URL을 단축 URL으로 변환하는데 해시 함수가 사용된다.

해시 함수 구현에 쓰이는 두 가지 방법

- 해시 후 충돌 해소
- base-62 변환

### 해시 값 길이

hashValue는 [0-9, a-z, A-Z]의 문자들로 구선된다. 따라서 10 + 26 + 26 = 62이다.

원래 URL을 몇 자의 단축 URL로 줄일지는 62^n ≥ 3650억 을 계산하면 나온다.

| n | URL 개수 |
| --- | --- |
| 1 | 62¹ = 62 |
| 2 | 62² = 3,844 |
| 3 | 62³ = 238,328 |
| 4 | 62⁴ = 14,776,336 |
| 5 | 62⁵ = 916,132,832 |
| 6 | 62⁶ = 56,800,235,584 |
| 7 | 62⁷ = 3,521,614,606,208 ≈ 3.5조(trillion) |
| 8 | 62⁸ = 218,340,105,584,896 |

**3560억이 기준이 된 이유**

- 쓰기 연산 : 매일 1억개의 단축 URL이 생성된다 가정
- 초당 쓰기 연산 : 1억 / 24 / 3600 = 1160
- 읽기 연산 : 읽기 연산과 쓰기 연산의 비율이 대략 10:1이라고 가정 → 읽기 연산 초당 11,600회 발생
- URL 단축 서비스 10년간 운영 : 1억 * 365 * 10 = 3650억 개의 레코드 보관 필요
- 축약 전 URL 평균 길이 100이라고 가정하면
- 10년동안 필요한 저장 용량은 3650억 * 100 바이트 = 36.5TB

### 해시 후 충돌 해소

긴 URL을 줄이려면, 원래 URL을 7글자 문자열로 줄이는 해시 함수가 필요하다. (여기서 7글자 문자열로 줄이는 이유는 URL을 짧고 가독성 좋게 유지하면서도 충분한 유니크 값을 보장하기 위해서이다.)

손쉬운 방법은 CRC32, MD5, SHA-1같이 잘 알려진 해시 함수를 이용하는 것이다.

그런데 위의 해시 함수들을 사용하면 7자보다 길다. 이를 어떻게 줄일 수 있을까?

- 첫 번째 방법 : 계산된 해시 값에서 처음 7개 글자만 이용
    - 해시 결과가 서로 충돌할 확률이 높아진다.
    - 충돌이 발생했을 때는, 충돌이 해소될 때까지 사전에 정한 문자열을 해시값에 덧붙인다.
    - 이 방법을 쓰면 충돌은 해소할 수 있지만 단축 URL을 생성할 때 중복된 해시가 있는지 최소 한 번 이상 데이터베이스 질의를 해야 하므로 오버헤드가 크다.
- 두 번째 방법 : 블룸 필터 사용
    - 데이터베이스 대신 블룸 필터를 사용하면 성능을 높일 수 있다.
    - 블룸 필터는 어떤 집합에 특정 원소가 있는지 검사할 수 있도록 하는, 확률론에 기초한 공간 효율이 좋은 기술이다.

### base-62 변환

진법 변환은 URL 단축기를 구현할 때 흔히 사용되는 접근법 중 하나다. 이 기법은 수의 표현 방식이 다른 두 시스템이 같은 수를 공유하여야 하는 경우에 유용하다.

62진법을 쓰는 이유는 hashValue에 사용할 수 있는 문자 개수가 62개이기 때문이다.([0-9, a-z, A-Z]의 문자들로 구선된다. 따라서 10 + 26 + 26 = 62이다.)

**가독성 증가**

base-62 변환을 하면 사람이 쉽게 읽고 기억할 수 있는 형태의 문자열이 생성되므로, 단축 URL에 적합하다.

**Base-62 변환 과정**

예를 들어, 10진수 `11157`을 62진수로 변환하는 과정을 살펴보겠습니다.

1. 62로 나눈 몫과 나머지를 구하기
    - `11157 ÷ 62` = 179, 나머지 59 (X)
    - `179 ÷ 62` = 2, 나머지 55 (T)
    - `2 ÷ 62` = 0, 나머지 2 (2)
2. 나머지를 거꾸로 읽기
    - 2TX(62진법에서 변환된 값)

즉, `11157₁₀` = "2TX"₆₂

이렇게 변환된 값을 활용하여 단축된 URL을 만들 수 있습니다.

예: `https://tinyurl.com/2TX`

**Base-62 변환 vs 해시 함수 방식 비교**

| 비교 항목 | 해시 후 충돌 해소 전략 | Base-62 변환 |
| --- | --- | --- |
| **충돌 해결 방식** | 충돌이 발생할 수 있어 추가적인 해소 전략이 필요 | ID가 유일하게 보장된 후 적용 가능 |
| **단축 URL 길이** | 고정된 길이 | 가변적 (ID가 커질수록 길어짐) |
| **유일성 보장** | 별도의 ID 생성 불필요 | ID 유일성 보장을 위한 별도 전략 필요 |
| **보안 문제** | 해시 기반이라 예측 어려움 | 증가하는 ID 기반이라 차후 URL 패턴이 예측될 가능성이 있음 |
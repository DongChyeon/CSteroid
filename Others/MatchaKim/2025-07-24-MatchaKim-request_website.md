---
date: 2025-07-24
user: MatchaKim
topic: request_website
---

우리가 웹사이트에 접속할때 뭘해??

## 웹사이트가 뭔데?

위키피디아에 의하면 웹사이트는

> 인터넷 프로토콜 기반의 네트워크에서 도메인 이름이나 IP 주소, 루트 경로만으로 이루어진 일반 URL을 통하여 보이는 웹 페이지들의 의미 있는 묶음

## 인터넷 프로토콜은 뭐야?

인터넷 프로토콜은 네트워크를 통해 이동하고 올바른 대상에 도착할 수 있도록 데이터패킷을 라우팅하고 주소를 지정하기 위한 규칙이야

## 데이터 패킷이 뭔데

인터넷을 통과하는 데이터를 더 작은 단위로 패킷 이라는 조각으로 나눠. 각각의 조각에 IP 정보가 첨부되어있는거지

인터넷에 있는 모든 장치, 도메인에는 IP가 할당되어있거든 그래서 데이터가 필요한곳에 보낼 수 있는거지

패킷이 목적지에 가면 IP가 아까 헤더에 있다고 했잖아? 거기에 프로토콜도 같이 있거든? 예를 들면 TCP나 UDP 그럼 패킷이 목적지에 가면 그거에 맞게 처리되는거지

## 라우터는 뭔데

IP 주소 기준으로 데이터가 가는 길을 정해주는 장비야

그렇게 계속 다음 라우터로 넘겨주는거지

인터넷은 수많은 라우터로 연결되어있어서 데이터가 돌아다니는 거거든

## 그럼 라우팅은 뭔데??

라우터가 어디로 보낼지 판단해서 다음 홉으로 넘겨주는 과정이야

## 다음 홉으로 어떤과정으로 넘겨주는데?

1. 먼저 라우팅 테이블을 봐
2. 목적지 IP에 맞는 다음홉을 찾아
3. MAC주소를 알아내서
4. 그 MAC주소를 가진 장비로 전송해

## 그럼 라우팅 테이블은 뭐야

라우팅 테이블은 패킷의 목적지 IP가 어느 대역인지 판단하고 가장 맞는 경로로 보내는거야

## MAC 주소는 그럼 어떻게 알아내 ??

ARP를 활용해서 알아내 이건 IP를 MAC주소로 변환하는 프로토콜이거든
브로드 캐스트로 외치는거야

> "192.168.1.1인 누구냐? 누구 아는사람 MAC 주소좀 알려줘!" 하고

그럼 해당 IP 가진 놈이 응답을 해줘

> "나야! 내 MAC 주소는 'xx:xx... 이야!'"
> 유니캐스트로 물어본 놈한테만 가는거지

그럼 MAC주소를 캐시에 올려서 다음에 또 안물어봐도 돼

> [ Ethernet 프레임 ]

    └── [ IP 패킷 ]
        └── [ TCP 세그먼트 ]
             └── [ 실제 데이터 (예: HTTP) ]

전송계층 구조가 이런 모양이거든

HTTP 요청도 결국 이더넷 프레임이라는 최하위 껍질에 싸여서 전송되는거지

## 오케이 이제 알겠다 인터넷 프로토콜에 대해서는 그럼 다시 돌아와서 URL 은 뭐야?

Uniform Resource Locator
자원에 접근하기 위한 주소를 말해

Uniform (일관된 방식): 정해진 하나의 규칙을 따라

Resource (자원): 웹 페이지, 이미지, 동영상 등과 같은

Locator (위치 지정자): 자원의 위치를 찾아갈 수 있도록 알려주는 주소

## URL 과 URI 의 차이는 뭐야?

URI (Uniform Resource Identifier): 가장 큰 개념
자원을 식별하기 위한 모든 방법을 포괄하는 총칭

URL : URI의 한 하위 종류, 자원의 위치를 알려주고 어떻게 접근하는지 명시

URI -> 식별자

URL -> 정확한 위치

## 그래 그럼 돌아가서 웹사이트가 웹페이지면 그게 뭔데

웹페이지는 월드와이드 웹 상에 있는 각각 문서를 말해

책 페이지랑 차이점은 서로 웹페이지는 하이퍼링크로 연결시킬 수 있는게 특징이야

## 하이퍼 링크가 뭔데

하이퍼링크는 하이퍼 텍스트 문서 안에서 직접 자료를 연결하고 가리키는 참조 고리야

## 그럼 하이퍼 텍스트는 뭔데

쉽게 설명하면 클릭 가능한 링크가 포함된 문서야

## 하이퍼(hyper)가 무슨뜻이길래?

초월하는, 다차원의 같은 뜻으로
기존 문서들은 순서대로 읽어야 해서 저자가 정한 흐름이 있었는데

하이퍼 텍스트는 달라 이문서 저문서를 자유롭게 넘나들며 이동 가능한거지

순서를 이렇게 초월한다는 의미에서 "하이퍼"가 붙는거야

## 다시 초기질문으로 돌아가서~ 우리가 웹사이트에 접속할때 뭘해??

브라우저에 주소를 입력하지

## 그럼 브라우저가 뭔데?

웹페이지에 접근하고 인터넷을 탑색하는 소프트웨어야 GUI 기반으로 구성되어있어

HTTP로 통신하는데 웹페이지를 가져오기도 하는데 웹서버에 정보를 보내기도 해

웹페이지 파일 포맷은 보통 HTML이었어

예전에는 ftp://example.com 식으로 접근하면
브라우저가 FTP 서버 접속해서 폴더도 보여주고 다운로드도 하고 그랬어 근데 이제는 불가능하지

왜냐면 FTP는 암호화가 안되고, 오래돼서 복잡하고, 사용자도 거의 없어서 2021년부터 다 제거됐지

## 그렇구나? HTML이 뭔데?

HyperText Markup Language 로 웹브라우저에 표시되도록 설계된 표준 마크업 언어야

## 마크업 언어는 뭔데?

태그등을 이용하여 문서나 데이터 구조를 명기하는 언어중 하나야

데이터에 설명(메타정보)를 추가 해주는거야

`<h1>안녕하세요</h1>`

여기서 h1은 이건 헤더야 하고 알려주는 마크업인거지

## 그렇구나 그럼 브라우저에 URL 을 입력하면 어떻게 되는데?

브라우저는 URL을 각각의 요소로 나누게 돼

- https: → 프로토콜

- www.example.com → 도메인

- / → 요청 경로

그리고 www.example.com 라는 도메인 이름을 IP주소로 바꿔야 한다고 했지??

그럼 이걸

1. 가장 먼저 브라우저 캐시를 확인 이전에 같은 URL로 접속한 적 있으면,
   자신이 갖고 있는 캐시에 저장된 IP 주소를 먼저 찾는거지

2. 브라우저에 없으면 운영체제에 저장된 DNS캐시를 확인해

3. 마지막으로 /etc/hosts 파일을 확인해서 수동으로 특정 도메인에 IP를 강제로 매핑해
   localhost 같은게 여기에 정의되어있어

4. 위에서도 못찾으면 로컬 네트워크의 DNS를 확인해 보통 공유기가 첫 DNS중계자거든 그래서 공유기도 내부에 DNS캐시가 있을 수 있으니까

5. 라우터에도 없으면 ISP DNS서버로 넘어가 (KT, SK 등...)

내부적으로 이 과정도

첫째 전세계에 13개의 루트서버가 있는데 물어봐

> "www.example.com의 IP가 뭐야?"

그럼 루트가 답해줘

> "나는 www.example.com은 모르겠고,
> .com 도메인(TLD)은 이 네임서버들이 관리해. 그쪽에 물어봐."

그럼 다시 TLD Top Level Domain 한테 가서 물어보면

> "example.com은 내가 관리 안 하고,
> 이 IP의 네임서버들이 알아. 거기로 가봐."

그럼 권한이 있는 DNS 서버가에 가서 다시 물어봄

> "www.example.com의 실제 IP 주소가 뭐야?"

그에대한 대답

> "www.example.com의 IP는 93.184.216.34야!"

## 캐싱은 어떻게 이루어져?

DNS 응답에는 TTL이라는 값이 있어

www.example.com IN A 93.184.216.34 TTL=3600

TTL=3600 이면 3600초동안 IP는 캐시 가능한거지

## 오케이 브라우저가 그럼 IP를 알면 그다음과정은 뭐야?

1. 먼저TCP 연결을 해 웹서버와 연결을 맺는데 3-Way Handshake를 하는거야 관련된 글은 https://github.com/DongChyeon/CSteroid/blob/%EA%B9%80%EC%A4%80%EC%8B%9D/Networks/MatchaKim/2025-07-13-MatchaKim-http.md 여기 참조해봐

2. HTTPS 인 경우 TLS 핸드쉐이크를 진행해(그 내용은 다음글에 남길게)

3. 그럼 이제 브라우저는 실제로 서버에게 요청을 보내 그리고 웹서버는 HTTP응답을 돌려줘

## 브라우저에 이제 응답 본문이 왔네 이부분부터 상세하게 설명해줘

응답 본문이 오면 요청된 페이지에 글꼴, 이미지, 스크립트, 광고 같이 서로다른 호스트들이 오잖아 ? 그럼 그거에대해서 DNS 를 또 조회하고 모두 위에 과정을 거쳐서 자원을 가져와

1. 초기에 웹서버로 연결이되면 HTTP GET 요청을 보내서 HTML 내용을 응답해

2. 요청에 대한 응답은 수신된 첫 바이트 데이터를 포함하고 있어 TTFB Time to First Byte는 요청을 보내고 HTML 첫 패킷을 받는데 걸린 시간이야 보통 14kb 정도이고

이후 브라우저가 링크를 만날때 까지는 링크가 걸린 자원들한테는 요청이 안가

3. 브라우저가 첫번째 데이터청크를 받으면 파싱을 시작해

## 데이터 청크? 패킷, 이더넷프레임, 세그먼트 다 한번 다시 정리해줘

1. 브라우저가 서버에 HTTP 요청 (애플리케이션 계층)

2. 응답은 HTML, 이미지 등의 데이터 → 필요 시 청크로 분할

3. 이 청크를 TCP가 받음 → 헤더 붙임 → 세그먼트

4. 세그먼트를 IP가 받음 → IP 헤더 붙임 → 패킷

5. 이걸 이더넷이 받음 → MAC 주소 등 이더넷 헤더 붙임 → 프레임

## 그래 다시 돌아와서 첫 청크를 받으면 어떻게 된다고?

어 청클를 받으면 수신된 구문을 분석하기 시작해

이떄 예측구문 분석이라는 과정을 거쳐

## 예측 구문분석이 뭔데??

```html
브라우저는 <script> 를 만나면 파싱을 멈춰
```

## 왜 멈추는데?

```html
<script> 안에 스크립트가 HTML의 구조를 바꿀 수 있다고 가정해서 그래
```

그래서 현재까지 DOM은 그려놓고 JS 를 실행하고 그 뒤에오는 HTML을 파싱하는거지

## 그럼 script 실행을 기다려야해서 지연되겠다..

맞아 그래서 예측구문을 하는거야 별도의 파서가 script 아래의 HTML을 미리 분석해서 이미지, css, 추가 스크립트를 사전에 다운로드 요청하는거지

그게 예측 구문 분석이야

## 아니 그러면 script는 가장 밑에두는게 좋겠다

응 맞아 아래 태그들을 공부해놓으면 좋아

```html
<script>: HTML 파싱을 중단하고 스크립트를 즉시 실행
```

```html
<script defer>: HTML 파싱이 끝난 뒤 실행 (렌더링은 안 멈춤)

defer -> 미루다
```

```html
<script async>: 다운로드되자마자 실행 → 파싱과 충돌 가능성 있음
```

| 속성    | HTML 파싱 중단 | 실행 순서 보장 | 실행 시점            | 적합한 경우                     |
| ------- | -------------- | -------------- | -------------------- | ------------------------------- |
| 없음    | ✅ 중단됨      | ✅ 됨          | 도달 즉시            | 간단한 스크립트, 아주 초기 코드 |
| `async` | ❌ 중단 가능   | ❌ 안 됨       | 다운로드 끝나면 즉시 | 외부 스크립트 (광고, 통계 등)   |
| `defer` | ❌ 안 중단됨   | ✅ 됨          | HTML 파싱 완료 후    | 대부분의 일반 스크립트 (UI 등)  |

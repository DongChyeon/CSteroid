---
date: 2025-07-31
user: Roel4990
topic: Web Programming(웹 기초, React, Next.js, TS ...) 시작하기
---

## ✨ 웹 개발, 뭐부터 알아야 할까?

### 1. HTML, CSS, JavaScript — 웹 개발의 기본 3요소

웹 개발의 가장 기초는 **HTML, CSS, JavaScript**야.  
쉽게 설명하자면:

- **HTML**은 웹 페이지의 구조와 뼈대를 만들고,
- **CSS**는 그 뼈대에 예쁘게 색칠을 입히고,
- **JavaScript**는 여기에 **생명을 불어넣는 역할**을 해. (즉, 변화와 반응)

간단한 예시 코드야

```html
<h1>나의 첫 웹페이지</h1>
<p>안녕하세요!</p>
```

```css
h1 {
  color: black;
  font-size: 32px;
}
```

```js
document.querySelector("h1").onclick = () => {
  alert("제목을 클릭했어요!");
};

$("h1").on("click", function() {
    alert("제목을 클릭했어요!");
});
```

---
### 2. React — 요즘 가장 많이 쓰는 UI 라이브러리

Vue, Angular, React 같은 프레임워크/라이브러리 중 요즘은 **React**가 가장 핫하고 널리 쓰이고 있어.

React의 핵심 개념은 “컴포넌트”야.  
작은 단위로 쪼갠 UI 조각들을 조립해서 전체 화면을 만들어가는 구조야.

```jsx
function Button() {
  return <button>Click me</button>;
}
```

> 📱 Jetpack Compose랑 비교하면,  
> `function Component()`는 `@Composable fun`처럼 생각하면 돼!

---

### 3. Next.js 

React는 UI만 담당하지만, Next.js는 여기에 **페이지 라우팅, 서버사이드 렌더링(SSR), SEO 최적화** 등을 얹어줘.  
즉, 더 실전적인 웹앱을 만들 수 있게 도와주는 프레임워크야.

```tsx
export default function Home() {
  return <h1>Home Page</h1>;
}
```

> 📱 Jetpack Compose 기준으로 보자면,  
> `NavController`에 `ViewModel`까지 초기화되어 있는 상태라고 보면 이해가 쉬워!

---

### 4. TypeScript — 안전한 자바스크립트

자바스크립트는 유연하지만 가끔 너무 자유로워서 에러도 잘 나지.  
그래서 등장한 게 **TypeScript**!  
정적 타입 시스템 덕분에 코드 작성 중간에 오류를 미리 잡을 수 있어.

```tsx
type User = {
    name: string;
    age: number;
};

function greet(user: User): string {
    return `Hello, ${user.name}. You are ${user.age} years old.`;
}

type ButtonProps = {
    text: string;
    onClick: () => void;
};

function MyButton({ text, onClick }: ButtonProps) {
    return <button onClick={onClick}>{text}</button>;
}
```

---


### 5. NPM / Yarn — 외부 도구 설치 창고

외부 라이브러리(axios, lodash, react-router 등)를 설치하고 관리할 수 있는 도구야.

```bash
npm install axios
```

> 📱 Android에서는 Gradle로 외부 라이브러리 추가하지?  
> 웹에서는 그걸 NPM이나 Yarn으로 하는 거야.

---

### 6. 반응형 웹 / 미디어쿼리 — 다양한 기기 대응

폰, 태블릿, 데스크탑에서 화면이 알아서 반응하도록 만들고 싶다면?

```css
@media (max-width: 600px) {
  h1 { font-size: 20px; }
}
```

> Jetpack Compose에서는 `BoxWithConstraints`로 디바이스 크기 따라 조정하는 것과 비슷해.

---

### 7. 테스트 — UI 동작 확인 자동화

작성한 UI가 잘 작동하는지 자동으로 검증할 수 있어.

```tsx
test("버튼 클릭 시 메시지 변경", () => {
  render(<App />);
  fireEvent.click(screen.getByText("Click me"));
  expect(screen.getByText("You clicked!")).toBeInTheDocument();
});
```

> Jetpack Compose의 `composeTestRule.setContent {}`랑 동일한 구조로 테스트할 수 있어!

---


### 8. 서버 통신 — 데이터를 가져오자!

보통 데이터를 가져올 땐 `fetch`나 `axios` 같은 걸 써.  
근데 `React Query`를 쓰면 **로딩, 에러, 캐싱, 리페칭**까지 다 자동으로 처리돼.

```tsx
const { data } = useQuery(["users"], fetchUsers);
```

> 📱 Compose 기준으로 보자면:
> - `axios`는 `Retrofit`,
> - `React Query`는 `ViewModel + Flow + Repository` 조합이랑 비슷한 역할이야.

---

## 🧩 요약표

| 개념          | 역할/비유         | Jetpack Compose 기준 비교     |
|---------------|------------------|-------------------------------|
| HTML          | 페이지 구조(캔버스)  | XML / Layout 구성             |
| CSS           | 디자인/스타일링     | Modifier, Theme               |
| JavaScript    | 동작과 반응 제어     | 이벤트 리스너                 |
| React         | UI 컴포넌트화       | @Composable                   |
| Next.js       | 페이지 + SSR       | Navigation + SSR 초기 상태     |
| TypeScript    | 안전한 JS 타입 시스템 | Kotlin 타입 시스템            |
| NPM/Yarn      | 외부 라이브러리 설치 | Gradle                        |
| 미디어쿼리       | 반응형 레이아웃      | BoxWithConstraints             |
| 테스트         | 자동 동작 검증       | composeTestRule               |
| React Query   | 서버통신 + 상태관리  | Retrofit + Flow + ViewModel 조합 |

---

간단하게 이거 기반으로 간단한 프로젝트는 할 수 있을 거야.

조금 더 공부하자면 React 기반 웹 프로젝트는 어떤 폴더 구조를 사용하고 어떤 아키텍처를 사용하는지 공부해보면 좋을 거 같아.

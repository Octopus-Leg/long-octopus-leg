# 7. 사용자 인터페이스 설계하기

## 1️⃣ React 기초
- 리액트는 격리된 작은 컴포넌트를 포함하는 대화형, 동적 UI를 빌드하기 위한 **선언적 라이브러리**임.
- 흔히 프레임워크라고도 불리지만, 실제로는 리액트 라우터나 리덕스 등 주변 라이브러리와 함께 생태계를 구성하는 라이브러리임.
- 자바스크립트 문법 내에서 HTML과 유사한 마크업을 작성할 수 있는 JSX를 사용해 템플릿을 작성함.
- 동적 변경 사항에 효율적으로 대응하기 위해 메모리 내에 실제 DOM을 복사한 **가상 문서 객체 모델(VDOM)** 을 활용함.
### 리액트 앱 만들기
```shell
npx create-react-app <app name>
```
- **npm 패키지 실행기(NPX)** 는 CLI 도구로, npm 레지스트리의 실행 파일을 로컬에 설치하지 않고도 즉시 사용하게 해줌.
```shell
yarn start
```
- 해당 명령어로 개발 서버를 실행하며, 주요 프로젝트 구조는 아래와 같음.
  - **node_modules 폴더:** 응용 프로그램에 필요한 모든 외부 의존성 패키지의 로컬 복사본을 보관함.
  - **public 폴더:** index.html, 파비콘, 로고 등 앱의 정적 리소스가 포함됨.
  - **src 폴더:** 실제 리액트 로직이 담긴 코드와 CSS 파일 등이 저장되는 핵심 폴더임.
  - **package.json:** 프로젝트 메타데이터와 설치된 패키지 목록, 실행 스크립트 정의를 포함함.
### package.json 파일에 대한 이해
- 자바의 `build.gradle`처럼 프로젝트의 모든 의존성을 관리하는 설정 파일임.
- 핵심 라이브러리인 react는 리액트의 기능을, react-dom은 가상 DOM 관리를 담당함.
- `scripts` 필드에는 `react-scripts` CLI 패키지를 통해 실행되는 주요 명령어들이 정의되어 있음.
   ```json
  "scripts": {
      "start": "react-scripts start",
      "build": "react-scripts build",
      "test": "react-scripts test",
      "eject": "react-scripts eject"
  },
  ```
- `start`: Node.js 환경에서 개발용 서버를 구동할 때 사용함.
- `build`: 프로덕션 배포를 위해 애플리케이션을 최적화 및 패키징함.
- `test`: 제스트(Jest, 테스트 프레임워크)를 테스트 러너로 사용해 테스트 코드를 실행함.
- `eject`: 숨겨진 빌드 설정을 노출시켜 사용자가 직접 세부 설정을 커스터마이징할 수 있게 함. 되돌릴 수 없는 단방향 작업이라 주의해야 함.
### React 앱의 부트스트랩
- 웹 페이지는 HTML 문서이며 브라우저는 이를 트리 구조인 DOM으로 관리하지만, 직접적인 DOM 조작은 리소스 소모가 큼.
- 리액트는 이를 해결하기 위해 메모리 기반의 가상 DOM(VDOM)을 활용하며, `react-dom` 패키지가 이를 유지 및 관리함.
- 데이터나 상태 변경 시 전체를 다시 그리는 대신, 가상 DOM과 실제 DOM을 비교해 꼭 필요한 변경 사항만 실제 화면에 반영함.

#### 첫 번째 렌더링
- `index.html`이 메인 뼈대가 되며, `src/index.js`가 **애플리케이션의 진입점 역할**을 함.
- 렌더링 시 `index.html`의 `root` 엘리먼트를 찾아 `ReactDOM.createRoot`에 전달하여 루트 객체를 생성함.
```javascript
const root = ReactDOM.createRoot(document.getElementById('root'));

root.render(
    <React.StrictMode>
      <App />
    </React.StrictMode>
);
```
- 리액트 컴포넌트는 단일 구조일 수도 있고, 여러 계층의 자식 컴포넌트(ex. App > Cart > Item)로 구성된 부모 구조일 수도 있음.
- `<React.StrictMode>`는 개발 모드에서 베스트 프랙티스 준수 여부를 확인하고 잠재적 위험을 감지하기 위해 두 번 렌더링하며 경고를 남김.(프로덕션 빌드에는 무관함.)
- 최종적으로 `render` 함수는 JSX를 HTML로 변환해 `root` 태그 안에 삽입하고, 가상 DOM 비교를 통해 도출된 변경점을 실제 화면에 적용함.

## 2️⃣ 리액트 컴포넌트 및 기타 기능에 대해 알아보자
- 각 페이지는 여러 개의 **리액트 컴포넌트**(헤더, 푸터, 컨텐츠 등)를 조합해 구성됨.
- 리액트에서는 **자바스크립트 클래스**를 사용하거나 **함수**를 사용하는 두 가지 방법으로 컴포넌트를 만들 수 있음.
- ES6(ECMAScript 6)에서 제공하는 **화살표 함수**를 사용한 스타일로 작성된 것을 **리액트 함수형 컴포넌트**라고 부름.
```javascript
const Header = (props) => {
    return (
        <div>
          <h1>{props.title}</h1>
        </div>
    )
}

export default Header;
```
```javascript
export default class Header extends React.Component {
    render() {
        return (
            <div>
              <h1>{this.props.title}</h1>
            </div>
        )
    }
}
```
- 두 방식 모두 JSX를 반환하며, 각각 함수와 클래스를 export하기 때문에 다른 컴포넌트에서 import하여 재사용 가능함.
- **props** 는 외부에서 컴포넌트로 전달되는 데이터이며, JSX 내에서 HTML 태그 속성을 사용하는 것처럼 작성함.
- 클래스형 컴포넌트에는 render() 함수가 필요한 반면 함수형 컴포넌트는 단순히 return 구문만 있으면 됨.
### JSX에 대해 알아보자
- JSX는 HTML과 유사하나 자바스크립트 기반이므로 **HTML 애트리뷰트** 작성 시 차이가 있음.
- `class`는 **className**으로, `for`는 **htmlFor**로, `fill-rule`은 **fillRule**로 수정해야 함.
- `React.StrictMode` 컴포넌트를 사용하면 HTML 애트리뷰트 사용이나 오타에 대해 경고와 제안을 받을 수 있음.
- 컴포넌트를 동적으로 만들기 위해 **중괄호**`{}` 안에 **자바스크립트 표현식**을 넣어 변수 값을 할당하거나 이벤트를 처리함.
  ```javascript
  <img className="h-24" src={item?.imageUrl} alt="" />
  ```
### 리액트 훅에 대해 이해해보자
- 컴포넌트는 동적으로 동작하며 **상태**를 포함함.
- 리액트 16.8 버전 이전에는 클래스형만 상태를 지원했으나, 이제는 함수형 컴포넌트에서도 **훅(hook)** 을 통해 상태를 지원함.
- **useState:** 상태를 정의하고 유지할 수 있게 하며, `[상태값, setter함수]` 형태로 선언함.
  ```javascript
  import { useState } from "react";
  const [total, setTotal] = useState(0);
  ```
  - total 상태는 number 타입이므로 0으로 초기화함. setTotal(100)을 호출해 total의 상태값을 100으로 업데이트할 수 있음.
- **useEffect:** 컴포넌트 **렌더링 후 추가 작업**(API 로드, 이벤트 리스너 추가 등)을 수행할 때 사용함.
- **useContext:** **props 전달 과정** 없이 한 컴포넌트에서 다른 하위 컴포넌트로 데이터를 직접 전달함.
- **useReducer:** `useState`의 고급 버전으로, `reducer` 함수를 통해 복잡한 상태 관리 로직을 구현함.
  ```javascript
  const [state, dispatch] = useReducer(reducer, initialState);
  ```
### 테일윈드(Tailwind)를 사용해 컴포넌트를 스타일링하기
- 테일윈드 CSS는 반응형 UI를 디자인하는데 도움이 되는 **유틸리티 CSS 프레임워크**임.
- 아래 명령어를 통해 패키지를 설치하고 **설정 파일(tailwind.config.js)** 을 생성함.
  ```shell
  npm install -D tailwindcss
  npx tailwindcss init
  ```
  ```JavaScript
  /** @type {import('tailwindcss').Config} */
  module.exports = {
      content: [],
      theme: {
          extend: {},
      },
      plugins: [],
  }
  ```

## 3️⃣ 프로덕션 배포에 불필요한 스타일을 제거하도록 설정
- `tailwind.config.js` 파일의 **content** 블록에 경로 필터를 추가하면 불필요한 스타일을 제거할 수 있음. ➡️ 빌드 결과물의 크기가 줄어서 성능이 향상됨.
  ```javascript
  module.exports = {
      content: ["./src/**/*.{js,jsx,ts,tsx}",
              "./public/index.html"],
      theme: {
          extend: {},
      },
      plugins: [],
  }
  ```
### 리액트에 테일윈드 포함시키기
- `src/index.css` 파일에 테일윈드의 **base**, **components** 및 **utilities** 스타일을 임포트함.
  ```css
  @tailwind base;
  @tailwind components;
  @tailwind utilities;
  ```
- `src/index.js` 파일에서 위의 CSS 파일을 임포트함.
  ```javascript
  import './index.css';
  ```
#### 기본 컴포넌트 추가하기
- `src/components` 디렉토리에 **Header**, **Container**, **Footer** 컴포넌트를 생성함.
  - **Header:** 상단에 표시되며 앱 이름 및 Login/Logout 버튼과 같은 항목을 포함함.
  - **Container:** 제품 목록과 같은 메인 콘텐츠를 포함함.
  - **Footer:** 하단에 표시되며 저작권과 같은 항목을 포함함.
- 위에서 생성한 컴포넌트를 `src/App.js` 파일에 추가함.
  ```javascript
  import Header from "./components/Header";
  import Footer from "./components/Footer";
  import Container from "./components/Container";
  
  function App() {
      return (
          <div className="flex flex-col min-h-screen h-full">
            <Header />
            <Container />
            <Footer />
          </div>
      );
  }
  
  export default App;
  ```

## 4️⃣ 전자상거래 앱 컴포넌트 디자인하기
- **제품 목록 컴포넌트:** 모든 제품 데이터를 화면에 그리며, 앱의 메인 접속 시 가장 먼저 보이는 **홈페이지 역할**을 수행함.
- **제품 상세 컴포넌트:** 사용자가 특정 제품을 선택했을 때 해당 제품의 **세부 정보를 시각화**하여 표시함.
- **로그인 컴포넌트:** 사용자로부터 이름과 비밀번호를 입력받아 **인증 절차를 처리**함.
- **장바구니 컴포넌트:** 사용자가 **장바구니에 담아둔 모든 항목을 나열**하고 관리함.
- **주문 컴포넌트:** 사용자의 **모든 주문을 테이블 형태**로 한눈에 확인하게 함.

## 5️⃣ Fetch를 이용해 API 호출하기
- 브라우저 내장 라이브러리인 **Fetch API**를 사용하여 REST API를 호출함.
- axios와 같은 써드파티 라이브러리를 사용할 수도 있으나 Fetch만 사용하면 써드파티 라이브러리의 의존성도 줄일 수 있음.

- 모든 API 통신에서 공통으로 사용할 설정 파일인 **`src/api/Config.js`** 를 생성함.
  - **Config.js:** URL 상수 및 인증 관련 공통 메서드를 포함하는 클래스임.
  - **defaultHeaders():** 모든 호출에 필요한 기본 헤더를 반환함.
  - **headersWithAuthorization():** 로컬 스토리지의 토큰을 포함한 인증 헤더를 반환함.
  - **tokenExpired():** 로컬 저장소에 보관된 토큰의 만료 여부를 확인해줌.
  - **storeAccessToken():** 로그인을 통해 받은 액세스 토큰과 만료 시간을 저장함.
- `ProductClient.js` 등에서 `Config` 클래스를 인스턴스화하여 사용함.

  ```javascript
  import Config from "./Config";
  
  class ProductClient {
      constructor() {this.config = new Config(); }
      async fetchList() {
          return fetch(this.config.PRODUCT_URL, {
              method: "GET",
              mode: "cors",
              headers: {
                  ...this.config.defaultHeaders(),
              }
          })
          .then(res => res.json());
      }
  }
  ```
- **Promise:** 브라우저에서 `fetch()`를 호출하면 즉시 결과가 오는 것이 아니라, 미래에 완료될 작업을 약속하는 **Promise 객체**를 반환함.
- **.then():** Promise 작업이 성공적으로 완료되었을 때 실행할 로직을 `.then()` 메서드를 통해 연결하여 처리함.(async/await 방식도 있음.)

#### 라우팅 설정
- **라우팅:** 단일 페이지 내의 각 부분으로 요청을 매핑하는 메커니즘임.
- 라우팅 관리를 위해 `react-router-dom` 패키지를 사용함.

## 6️⃣ 인증 기능 구현하기
- 토큰 저장소는 보안 수준과 사용자 편의성에 따라 쿠키, 세션 저장소, 로컬 저장소 중 선택하여 사용할 수 있음.
- **세션 저장소(Session Storage):** 동일한 탭 내에서만 유효하며, 탭을 닫거나 새로고침 시 데이터가 삭제됨. 보안이 중요한 앱에 적합함.
- **로컬 저장소(Local Storage):** 탭을 전환하거나 페이지를 새로고침해도 로그인 상태가 영구적으로 유지되어 사용자 편의성이 높음.

### 커스텀 useToken 후크 만들기
- `src/hooks/useToken.js`: 토큰의 상태를 효율적으로 관리하기 위해 생성함.
- 내부적으로 **`useState`** 훅을 사용하여 토큰 상태를 유지하고 관리함.
- `src/api/Auth.js`: 로그인, 로그아웃, 액세스 토큰 갱신 등 실제 서버와의 인증 통신을 전담함.

### 루트(App) 컴포넌트 작성
- 애플리케이션의 **전체 레이아웃**과 **라우팅 정보**를 포함하는 최상위 컴포넌트임.
- `react-router-dom` 패키지를 사용하며, 모든 `Route`는 `BrowserRouter` 컴포넌트 내부에 정의함.
- 일치하는 경로가 없을 경우를 대비해 와일드카드(`*`) 경로를 설정하여 **`NotFound`** 컴포넌트를 연결함.
```javascript
// App.js 일부
return (
    <div className="flex flex-col min-h-screen h-full ">
      <Router>
        <Header userInfo={token} auth={auth} />
          <div className="flex-grow flex-shrink-0 p-4">
            <CartContextProvider>
              <Routes>
                <Route path="/" exact element={<ProductListComponent />} />
                <Route path="/login" element={token ? <ProductListComponent /> : <LoginComponent />} />
                <Route path="/cart" element={token ? <Cart auth={auth} />  : <LoginComponent />} />
                <Route path="/orders" element={token ? <Orders auth={auth} /> : <LoginComponent />} />
                <Route path="/products/:id" element={<ProductDetail auth={auth} />} />
                <Route path="/*" exact element={<NotFound />} />
              </Routes>
            </CartContextProvider>
          </div>
        <Footer />
      </Router>
    </div>
);
```
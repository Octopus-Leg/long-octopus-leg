# 7. 사용자 인터페이스 설계하기

## 7.1 React 기초

- 리액트는 격리된 작은 컴포넌트를 포함하는 대화형, 동적 UI를 빌드하기 위한 선언적 라이브러리임. 템플릿 작성을 위해 **JSX(JavaScript Syntax Extension)** 를 사용하며, 동적 변경과 상호 작용을 위해 **가상 문서 객체 모델(VDOM)** 을 활용함.
- VDOM은 실제 DOM을 메모리에 복사한 것으로, 변경이 필요한 부분만 실제 DOM에 적용하여 성능을 최적화함.

### 7.1.1 리액트 앱 만들기

- **create-react-app** 유틸리티를 사용하면 기본 앱 구조를 빠르게 생성할 수 있음. **NPX(npm package executor)** 를 이용해 앱을 생성하고 개발 서버를 시작함.
```bash
npx create-react-app ecomm-ui
cd ecomm-ui
yarn start
```

- 서버가 성공적으로 시작되면 브라우저에서 **localhost:3000** 으로 새 탭이 열림.

### 7.1.2 기본 구조와 파일에 대해 알아보자

자동 생성된 리액트 앱의 디렉터리와 파일은 다음과 같음.

- `node_modules`: 모든 의존성 패키지의 로컬 복사본을 보관함. 변경하지 않음.
- `public`: index.html, 이미지, favicon 아이콘 및 robots.txt를 포함한 모든 정적 리소스가 위치함.
- `src`: 리액트 코드, CSS, 테스트 코드가 저장됨.
- `package.json`: 메타데이터, scripts 필드에 정의된 명령, 의존성 패키지 정의가 포함됨. 스프링 부트의 build.gradle 파일과 유사함.

### 7.1.3 package.json 파일에 대한 이해

- **react-scripts**에는 웹팩(Webpack), 제스트(Jest), ESLint, 바벨(Babel) 등 주요 도구가 포함됨.

**스크립트 명령**

- `start`: node 환경에서 개발 서버를 시작하며 **핫 리로드(hot reload)** 기능을 제공함.
- `build`: 프로덕션 배포를 위해 애플리케이션 코드를 패키징하고 최소화함.
- `test`: Jest를 테스트 러너로 사용해 테스트를 실행함.
- `eject`: 숨겨진 기본 빌드 구성을 노출시키는 **단방향 작업**임. 되돌릴 수 없으므로 주의가 필요함.

### 7.1.4 React 앱의 부트스트랩

- 리액트는 **react-dom** 패키지를 사용해 VDOM을 관리함.
- 리액트 앱을 초기화할 때 제일 먼저 루트 HTML 엘리먼트의 ID를 ReactDOM 객체의 render 함수에 전달함. 이 부분이 **리액트 앱의 진입점**임.
```javascript
// src/index.js
import React from 'react';
import ReactDOM from 'react-dom/client';
import './index.css';
import App from './App';

const root = ReactDOM.createRoot(document.getElementById('root'));

root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

- **React.StrictMode**: deprecated 메서드를 찾고 잠재적 위험을 발견하기 위해 두 번 렌더링하며, 개발 모드에서만 작동하므로 프로덕션 빌드에는 영향을 미치지 않음.
- render 함수는 App 컴포넌트의 JSX를 HTML로 변환하고, 생성된 VDOM과 실제 HTML DOM을 비교해 필요한 변경만 적용함.

## 7.2 리액트 컴포넌트 및 기타 기능에 대해 알아보자

- 각 페이지는 리액트 컴포넌트를 사용해 구성됨.
- 리액트에서는 **클래스형 컴포넌트**와 **함수형 컴포넌트** 두 가지 방법으로 컴포넌트를 만들 수 있으며, 주로 **화살표 함수 스타일의 함수형 컴포넌트**를 사용함.
```javascript
// 함수형 컴포넌트
export default const Header = (props) => {
  return (
    <div>
      <h1>{props.title}</h1>
    </div>
  )
}

// 클래스형 컴포넌트
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

- 두 코드 모두 JSX를 반환하며, export하기 때문에 다른 컴포넌트에서 import해서 사용할 수 있음.
- 둘 다 **props**를 가짐. 함수형은 아규먼트로, 클래스형은 `this.props`로 접근함.
- 클래스형 컴포넌트에는 `render()` 함수가 필요한 반면 함수형 컴포넌트는 return 구문만 있으면 됨.

### 7.2.1 JSX에 대해 알아보자

- JSX는 HTML과 매우 유사하지만 HTML 애트리뷰트 부분에서 차이가 있음.
- `class`는 `className`으로, `for`는 `htmlFor`로, `fill-rule`은 `fillRule`로 수정해야 함.
- 컴포넌트를 동적으로 만들기 위해 JSX 내에 자바스크립트 표현식을 넣을 수 있는데, 이때는 표현식을 **중괄호({})** 로 감싼 형태로 사용함.
```javascript
<img src={item?.imageUrl} alt="" />
<Link to={`/products/${item.id}`}>{item?.name}</Link>
<button onClick={() => removeItem(item.id)}>Remove</button>
```

- **?. 연산자(옵셔널 체이닝)**: null 또는 undefined 객체의 프로퍼티를 읽으려 할 때 발생하는 오류를 방지함.
- **onClick**: 커스텀 함수는 이 방법으로 이벤트와 연결함. 아규먼트를 전달하지 않거나 여러 명령문을 사용하는 경우에는 화살표 함수 대신 함수 이름을 직접 전달함.

### 7.2.2 리액트 훅에 대해 이해해보자

- 훅은 리액트가 제공하는 내장 함수 또는 사용자 정의 함수로서 상태를 저장하거나 부가적인 효과를 관리하는 데 사용함.

**훅 종류**

- **useState**: 컴포넌트 로컬 상태를 정의하고 유지함. setter 함수가 호출될 때마다 컴포넌트가 리렌더링됨. setter 함수 명명 규칙은 상태 이름 앞에 set을 붙이고 첫 글자를 대문자로 만드는 것임.
```javascript
import {useState} from "react";
const [total, setTotal] = useState(0); // 초기값 0으로 선언
```

- **useEffect**: 렌더링 완료 후 추가 작업(API 호출, 이벤트 리스너 등)을 수행함. 빈 배열(`[]`)을 전달하면 최초 1회만 실행됨.
- **useContext**: 컴포넌트 트리 전체에서 공유할 데이터(예: 로그인 상태, 테마)를 props 없이 전달할 때 사용함.
- **useReducer**: useState의 고급 버전. reducer 함수를 통해 복잡한 상태를 체계적으로 관리함.
```javascript
const [state, dispatch] = useReducer(reducer, initialState);
```

### 7.2.3 테일윈드(Tailwind)를 사용해 컴포넌트 스타일링하기

- **테일윈드 CSS**는 반응형 UI를 디자인하는 데 도움이 되는 유틸리티 CSS 프레임워크임.
- 테마, 애니메이션, 미리 정의된 패딩과 여백, 플렉스, 그리드 등을 지원함.
```bash
npm install -D tailwindcss
npx tailwindcss init  # tailwind.config.js 생성
```

## 7.3 프로덕션 배포에 불필요한 스타일을 제거하도록 설정

- 프로덕션 환경에서는 스타일 시트 용량을 작게 유지하여 애플리케이션 성능을 향상시킴.
- `tailwind.config.js` 파일의 content 블록에 아래 필터를 추가하면 테일윈드가 프로덕션 빌드 시 사용하지 않는 스타일을 자동으로 제거함.
```javascript
// tailwind.config.js
module.exports = {
  content: ["./src/**/*.{js,jsx,ts,tsx}", "./public/index.html"],
  theme: { extend: {} },
  plugins: [],
}
```

### 7.3.1 리액트에 테일윈드 포함시키기

- `src/index.css` 파일을 열고 테일윈드의 base, components, utilities 스타일을 임포트하도록 수정함.
```css
/* src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## 7.4 전자상거래 앱 컴포넌트 디자인하기

예제 전자 상거래 앱은 아래 컴포넌트들을 포함함.

- **제품 목록 컴포넌트**: 모든 제품을 표시하고 홈페이지 역할도 함. 각 제품은 이름, 가격, Buy now와 Add to bag 버튼이 포함된 카드 형태의 UI로 표시됨.
- **제품 상세 컴포넌트**: 사용자가 클릭한 제품의 상세정보를 표시함. 제품 이미지, 이름, 설명, 태그, Buy now 및 Add to bag 버튼이 표시됨.
- **로그인 컴포넌트**: 6장에서 구현한 `/api/v1/auth/token` API를 호출해 JWT를 발급받음. 로그인 실패 시 에러 메시지를 표시하며, Cancel을 클릭하면 제품 목록 페이지로 돌아감.
- **장바구니 컴포넌트**: 장바구니에 추가된 모든 항목을 나열함. 각 항목은 제품 이미지, 이름, 설명, 가격, 수량 및 합계를 표시함.
- **주문 컴포넌트**: 사용자의 모든 주문을 테이블 형태로 렌더링하며, 주문 날짜, 주문 항목, 주문 상태 및 주문 금액이 표시됨.
```javascript
// src/App.js
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

## 7.5 Fetch를 이용해 API 호출하기

- 샘플 전자 상거래 앱에서는 axios 같은 서드파티 라이브러리 대신 **Fetch 브라우저 내장 라이브러리**를 사용해 REST API를 호출함.
- Fetch는 샘플 앱에 필요한 작업을 수행하기에 충분하고 서드파티 라이브러리에 대한 의존성도 줄여줌.

### 7.5.1 제품 정보를 가져오는 API 클라이언트 작성하기

- 모든 API 클라이언트가 공통으로 사용하는 **Config.js**를 `src/api` 디렉터리 아래에 생성함. Config는 URL 상수와 헤더 생성 메서드를 포함하는 JavaScript 클래스임.
- 6장에서 구현한 JWT 토큰은 로컬 저장소에 저장하고, 인증이 필요한 API 호출 시 Authorization 헤더에 담아 전송함.
```javascript
// src/api/Config.js
class Config {
  SCHEME = process.env.SCHEME ?? "http";
  HOST   = process.env.HOST   ?? "localhost";
  PORT   = process.env.PORT   ?? "8080";

  defaultHeaders() {
    return { "Content-Type": "application/json", Accept: "application/json" };
  }

  storeAccessToken(token) {
    localStorage.setItem(this.ACCESS_TOKEN, `Bearer ${token}`);
    localStorage.setItem(this.EXPIRATION, this.getExpiration(token));
  }

  tokenExpired() {
    const expDate = Number(localStorage.getItem(this.EXPIRATION));
    return expDate <= Date.now();
  }
}
```

- `ProductClient.js`의 `fetchList()`는 모든 제품 목록을, `fetch()`는 제품 ID로 단일 제품 정보를 가져옴.
```javascript
// src/api/ProductClient.js
import Config from "./Config";
class ProductClient {
  constructor() { this.config = new Config(); }

  async fetchList() {
    return fetch(this.config.PRODUCT_URL, {
      method: "GET",
      mode: "cors",
      headers: { ...this.config.defaultHeaders() },
    })
    .then((res) => Promise.all([res, res.json()]))
    .then(([res, json]) => {
      if (!res.ok) return { success: false, error: json };
      return { success: true, data: json };
    })
    .catch((e) => this.handleError(e));
  }

  handleError(error) {
    return new Map([
      [TypeError,   "Problem fetching the response."],
      [SyntaxError, "Problem parsing the response."],
      [Error,       error.message],
    ]).get(error.constructor);
  }
}
export default ProductClient;
```

- fetch는 promise 객체를 반환하며, **response.ok**는 상태코드가 200~299 범위인 경우에만 true를 반환함.
- **handleError()**: `error.constructor`를 사용해 에러 유형을 확인하고 적절한 오류 메시지를 반환함.
- CartClient, CustomerClient, OrderClient도 동일한 패턴으로 `src/api/` 아래에 작성함.

### 7.5.2 제품 목록 페이지 코딩하기

- **ProductList**는 ProductClient를 사용해 제품 정보를 가져와서 하위 컴포넌트인 Products에 전달함. **useEffect** 훅을 사용하며, API가 한 번만 호출되도록 빈 배열(`[]`)을 전달함.
- **ProductCard** 컴포넌트는 Buy now 버튼과 Add to bag 링크가 있으며, 로그인하지 않은 사용자는 로그인 페이지로 리다이렉션함. react-router-dom의 **Link**와 **useNavigate()** 훅을 사용함.
- **ProductDetail** 컴포넌트는 경로에 포함된 ID를 **useParams()** 로 추출해 백엔드에서 제품 세부 정보를 로드함.
```javascript
// src/components/ProductList.js
const ProductList = ({ auth }) => {
  const [productList, setProductList] = useState();
  const [noRecMsg, setNoRecMsg] = useState("Loading...");
  const { dispatch } = useCartContext();

  useEffect(() => {
    async function fetchProducts() {
      const res = await new ProductClient().fetchList();
      if (res?.success) setProductList(res.data);
      else setNoRecMsg(res);
    }
    async function fetchCart(auth) {
      const res = await new CartClient(auth).fetch();
      if (res?.success) dispatch(updateCart(res.data.items));
    }
    if (auth?.token) fetchCart(auth); // 로그인한 경우 장바구니도 조회
    fetchProducts();
  }, []); // 빈 배열: 최초 1회만 실행
};
```

## 7.6 인증 기능 구현하기

- Login 컴포넌트 개발 전에 6장에서 구현한 JWT 기반 인증 API를 연동하기 위해 토큰 저장 위치를 결정해야 함.
- 서버 측에서는 쿠키를 사용하거나 스테이트풀(stateful) 통신을 하지 않으므로, 탭 전환이나 페이지 리프레시 후에도 로그인 상태가 유지되도록 **브라우저의 로컬 저장소**를 사용함.

### 7.6.1 커스텀 useToken 후크 만들기

- `src/hooks/useToken.js` 파일을 생성해 로컬 저장소 기반의 토큰 관리 훅을 구현함.
```javascript
// src/hooks/useToken.js
import { useState } from "react";
export default function useToken() {
  const getToken = () => {
    const saved = localStorage.getItem("tokenResponse");
    return saved ? JSON.parse(saved) : "";
  };

  const [token, setToken] = useState(getToken());

  const saveToken = (tokenResponse) => {
    localStorage.setItem("tokenResponse", JSON.stringify(tokenResponse));
    setToken(tokenResponse);
  };

  return { setToken: saveToken, token };
}
```

- Auth.js 클라이언트는 백엔드 REST API를 사용해 **로그인**, **로그아웃**, **액세스 토큰 갱신** 작업을 수행함.
- 로그인 시 액세스 토큰, 리프레시 토큰, 사용자 ID 및 사용자 이름을 로컬 저장소에 저장하고, 로그아웃 시 토큰을 제거하고 만료시간을 0으로 설정함.

### 7.6.2 Login 컴포넌트 작성

- `src/components/Login.js`를 생성함. `auth`와 `uri` 두 가지 prop을 아규먼트로 받으며, **PropTypes**를 사용해 prop의 유형을 검증함.
```javascript
// src/components/Login.js
import PropTypes from "prop-types";
Login.propTypes = { auth: PropTypes.object.isRequired };

const Login = ({ uri, auth }) => {
  const [username, setUserName] = useState();
  const [password, setPassword] = useState();
  const [errMsg, setErrMsg]     = useState();
  const navigate = useNavigate();

  const handleSubmit = async (e) => {
    e.preventDefault();
    const res = await auth.loginUser({ username, password });
    if (res?.success) {
      setErrMsg(null);
      navigate(uri ?? "/");          // 로그인 성공 시 지정 경로로 이동
    } else {
      setErrMsg(typeof res === "string" ? res : "Invalid Username/Password");
    }
  };

  const cancel = () => {
    navigate.length > 2 ? navigate(-1) : navigate("/");
  };
};
```

### 7.6.3 커스텀 cart context의 구현

- 리액트 라이브러리가 제공하는 **createContext**, **useReducer**, **useContext** 훅을 활용해 Redux와 유사한 커스텀 훅으로 cart 전역 상태를 관리함.
```javascript
// src/hooks/CartContext.js
import React, { createContext, useReducer, useContext } from "react";

export const CartContext = createContext();
export const useCartContext = () => useContext(CartContext);

export const UPDATE_CART = "UPDATE_CART";
export const REMOVE_ITEM = "REMOVE_ITEM";

export const updateCart = (items) => ({ type: UPDATE_CART, payload: items });
export const removeItem = (idx)   => ({ type: REMOVE_ITEM, payload: idx });

function reducer(state, action) {
  switch (action.type) {
    case UPDATE_CART:
      return { ...state, cartItems: action.payload };
    case REMOVE_ITEM:
      const items = [...state.cartItems];
      items.splice(action.payload, 1);
      return { ...state, cartItems: items };
    default:
      return state;
  }
}

export function CartContextProvider({ children }) {
  const [cartData, dispatch] = useReducer(reducer, { cartItems: [] });
  return (
    <CartContext.Provider value={{ ...cartData, dispatch }}>
      {children}
    </CartContext.Provider>
  );
}
```

- **reducer**: state와 action 두 개의 아규먼트를 받아, action을 기반으로 상태를 변경하고 반환하는 함수임.
- **CartContextProvider**는 useReducer 훅에서 reducer 함수를 사용하고, cartData를 CartContext.Provider의 value 애트리뷰트로 전달함.
- App 컴포넌트에서 CartContextProvider를 래퍼로 사용하면 내부의 모든 컴포넌트에서 `cartItems`와 `dispatch`에 접근할 수 있음.

### 7.6.4 Cart 컴포넌트 작성하기

- Cart 컴포넌트는 여러 개의 CartItem 컴포넌트를 포함하는 상위 컴포넌트임.
- `useCartContext`를 사용해 cartItems와 dispatch에 접근하며, CartClient, OrderClient, CustomerClient를 사용해 체크아웃 작업을 진행함.
- checkout 함수는 백엔드 서버에서 고객 정보를 가져온 뒤 주문 페이로드를 구성하고 주문 API를 POST 방식으로 호출하며, 성공 시 Orders 컴포넌트로 리다이렉션함.

### 7.6.5 Order 컴포넌트 작성하기

- 백엔드 서버에서 가져온 주문 세부 정보를 날짜, 상태, 금액, 항목을 포함해 표 형식으로 보여줌.
- `useEffect` 훅을 사용해 첫 번째 렌더링에서 주문 세부 정보를 로드하고 orders 상태로 관리함.
- 주문 날짜는 사용자의 현지 시간으로 표시되지만 서버에는 UTC 형식으로 저장됨.

### 7.6.6 루트(App) 컴포넌트 작성

- App 컴포넌트는 리액트 애플리케이션의 루트 컴포넌트임. react-router-dom 패키지의 **BrowserRouter(Router)**, **Route**, **Routes** 컴포넌트를 사용함.
```javascript
// src/App.js
import { BrowserRouter as Router, Routes, Route } from "react-router-dom";

function App() {
  const { token, setToken } = useToken();
  const auth = new Auth(token, setToken);

  return (
    <div className="flex flex-col min-h-screen h-full">
      <Router>
        <Header userInfo={token} auth={auth} />
        <div className="flex-grow flex-shrink-0 p-4">
          <CartContextProvider>
            <Routes>
              <Route path="/"             exact element={<ProductListComponent />} />
              <Route path="/login"        element={token ? <ProductListComponent /> : <LoginComponent />} />
              <Route path="/cart"         element={token ? <Cart auth={auth} />   : <LoginComponent />} />
              <Route path="/orders"       element={token ? <Orders auth={auth} /> : <LoginComponent />} />
              <Route path="/products/:id" element={<ProductDetail auth={auth} />} />
              <Route path="*"             exact element={<NotFound />} />
            </Routes>
          </CartContextProvider>
        </div>
        <Footer />
      </Router>
    </div>
  );
}
export default App;
```

- 모든 컴포넌트가 **CartContextProvider** 내부에 래핑되어 있기 때문에 `useCartContext` 커스텀 훅을 사용하면 어디서든 `cartItems`와 `dispatch`에 접근할 수 있음.
- `/cart`, `/orders`는 **token**이 있을 때만 접근 가능하며, 없으면 Login 컴포넌트로 리다이렉션함. 6장에서 보호한 API 엔드포인트와 동일한 접근 제어 개념임.
- 일치하는 경로가 없으면 NotFound 컴포넌트를 렌더링함.

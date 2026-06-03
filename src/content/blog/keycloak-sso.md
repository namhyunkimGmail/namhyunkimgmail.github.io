---
title: 'Keycloak으로 SSO 로그인 구현하기 (React + Spring Boot)'
description: 'Docker로 Keycloak을 띄우고 React 프론트엔드와 Spring Boot 백엔드에 OAuth2 기반 SSO 로그인/로그아웃을 구현한 경험을 정리했습니다.'
pubDate: '2026-06-03'
heroImage: '/blog-placeholder-3.jpg'
---

사내 프로젝트에서 **Keycloak**을 이용해 SSO(Single Sign-On) 로그인을 구현했다.  
Docker로 Keycloak 서버를 띄우고, React 프론트엔드와 Spring Boot 백엔드를 연동하는 전체 흐름을 정리해본다.

---

## 전체 아키텍처

```
사용자 브라우저
    │
    ▼
React (keycloak-js)
    │  ① 로그인 요청 → Keycloak 로그인 페이지로 redirect
    │  ② 로그인 성공 → JWT Access Token 발급
    │  ③ API 호출 시 Authorization: Bearer {token} 헤더 첨부
    ▼
Spring Boot (OAuth2 Resource Server)
    │  ④ Keycloak JWK로 토큰 서명 검증
    │  ⑤ 토큰에서 userId 추출 → DB 역할 조회
    ▼
Oracle DB
```

로그인 페이지를 직접 만들지 않아도 된다는 게 Keycloak의 가장 큰 장점이다.  
인증은 Keycloak이 전담하고, 우리 서버는 토큰만 검증하면 된다.

---

## 1. Docker로 Keycloak 설치

```bash
docker run -d \
  --name keycloak \
  -p 8180:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest \
  start-dev
```

`http://localhost:8180` 으로 접속하면 관리자 콘솔이 뜬다.

---

## 2. Keycloak Realm / Client 설정

관리자 콘솔에서 아래 순서로 설정한다.

**Realm 생성**
- Realm name: `idea-contest`

**Client 생성**
- Client ID: `idea-contest-app`
- Client type: `OpenID Connect`
- Valid redirect URIs: `http://localhost:5173/*`
- Web origins: `http://localhost:5173`

**사용자 생성**
- Users 메뉴에서 계정 생성 후 Credentials 탭에서 비밀번호 설정

---

## 3. 프론트엔드 (React + keycloak-js)

### 패키지 설치

```bash
npm install keycloak-js
```

### keycloak.js — 인스턴스 생성

```js
import Keycloak from 'keycloak-js';

const keycloak = new Keycloak({
    url: import.meta.env.VITE_KEYCLOAK_URL,      // http://localhost:8180
    realm: import.meta.env.VITE_KEYCLOAK_REALM,  // idea-contest
    clientId: import.meta.env.VITE_KEYCLOAK_CLIENT_ID, // idea-contest-app
});

export default keycloak;
```

### main.jsx — 앱 진입 시 로그인 강제

```jsx
import keycloak from './keycloak';

keycloak.init({ onLoad: 'login-required' }).then(async (authenticated) => {
    if (authenticated) {
        const userId = keycloak.tokenParsed.preferred_username;

        // 로그인 이력 저장 (새로고침마다 중복 호출 방지)
        if (!sessionStorage.getItem('loginHistorySaved')) {
            axios.post('/api/login-history', {
                userId,
                sessionId: keycloak.tokenParsed.sid,
                tokenId: keycloak.tokenParsed.jti,
            }, {
                headers: { Authorization: `Bearer ${keycloak.token}` }
            });
            sessionStorage.setItem('loginHistorySaved', 'true');
        }

        createRoot(document.getElementById('root')).render(
            <App />
        );
    }
});
```

`onLoad: 'login-required'` 옵션을 주면 비로그인 상태로 접근 시 Keycloak 로그인 페이지로 자동 redirect된다.

### api.js — 모든 API 호출에 토큰 자동 첨부

```js
import axios from 'axios';
import keycloak from './keycloak';

const api = async (url, options = {}) => {
    await keycloak.updateToken(30); // 만료 30초 전이면 자동 갱신
    return axios({
        url: `http://localhost:8080${url}`,
        headers: { Authorization: `Bearer ${keycloak.token}` },
        ...options,
    });
};

export default api;
```

`updateToken(30)` 이 핵심이다. 토큰 만료 30초 전에 자동으로 갱신해주기 때문에 사용자가 오래 작업하다 API를 호출해도 401이 나지 않는다.

### 로그아웃

```jsx
const handleLogout = () => {
    sessionStorage.removeItem('loginHistorySaved');
    keycloak.logout({ redirectUri: 'http://localhost:5173/login' });
};
```

---

## 4. 백엔드 (Spring Boot + OAuth2 Resource Server)

### build.gradle 의존성

```groovy
implementation 'org.springframework.boot:spring-boot-starter-security'
implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
```

### application.properties

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:8180/realms/idea-contest
```

이 설정 하나로 Spring이 Keycloak의 JWK(공개키)를 자동으로 가져와서 토큰 서명을 검증해준다.

### SecurityConfig.java

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwkSetUri(
                    "http://localhost:8180/realms/idea-contest/protocol/openid-connect/certs"
                ))
            );
        return http.build();
    }
}
```

### Controller에서 토큰 사용

```java
@GetMapping
public ApiResponse<?> getContests(@AuthenticationPrincipal Jwt jwt) {
    String userId = jwt.getClaimAsString("preferred_username");
    // userId로 DB에서 역할 조회 후 비즈니스 로직 처리
}
```

`@AuthenticationPrincipal Jwt jwt` 로 검증된 토큰 객체를 주입받아 userId, 권한 등의 클레임을 꺼낼 수 있다.

---

## 5. 로그인 이력 저장

로그인 성공 시 IP, User-Agent, 세션ID를 DB에 기록한다.

```java
@PostMapping("/api/login-history")
public ResponseEntity<Void> saveLoginHistory(
        @RequestBody LoginHistory loginHistory,
        HttpServletRequest request) {

    loginHistory.setLoginIp(request.getRemoteAddr());
    loginHistory.setUserAgent(request.getHeader("User-Agent"));
    loginHistory.setLoginType("WEB");
    loginHistoryService.saveLoginHistory(loginHistory);
    return ResponseEntity.ok().build();
}
```

---

## 정리

| 역할 | 기술 |
|------|------|
| 인증 서버 | Keycloak (Docker) |
| 프론트엔드 토큰 관리 | keycloak-js |
| 백엔드 토큰 검증 | Spring Security OAuth2 Resource Server |
| 토큰 자동 갱신 | `keycloak.updateToken(30)` |

직접 로그인 페이지를 만들거나 세션을 관리할 필요 없이 Keycloak에 위임하니 코드가 훨씬 단순해졌다.  
다음 글에서는 Keycloak의 역할(Role) 기능을 활용해 **역할 기반 접근 제어(RBAC)** 를 구현하는 방법을 다룰 예정이다.

---
date: 2025-08-15
user: Roel4990
topic: 소셜 로그인을 이용한 자동 로그인 구현 중 문제 발생 및 경합 상태(Race Condition) 해결
---

# 소셜 로그인을 이용한 자동 로그인 처리

## 1. 토큰 기반 인증과 자동 로그인

### 1.1 기본 흐름
사용자가 카카오와 같은 소셜 로그인을 사용하면, 클라이언트는 소셜 플랫폼을 통해 인증을 완료하고 서버로부터 Access Token과 Refresh Token을 발급받습니다.
- Access Token: 실제 API 요청 시 사용되는 토큰으로, 보안을 위해 짧은 유효 기간을 가집니다.
- Refresh Token: Access Token이 만료되었을 때, 새로운 Access Token을 발급받기 위해 사용되는 토큰으로, 상대적으로 긴 유효 기간을 가집니다.
클라이언트는 이 두 토큰을 안전하게 저장하고, Access Token이 만료되면 Refresh Token을 이용해 자동으로 갱신함으로써 사용자의 로그인 상태를 유지합니다.


### 1.2 자동 로그인 처리 로직
1. 클라이언트가 Access Token을 사용하여 서버에 API 요청을 보냅니다.
2. 서버는 Access Token의 유효성을 검증합니다.
3. Access Token이 유효한 경우: 정상적으로 요청을 처리하고 응답을 반환합니다.
4. Access Token이 만료된 경우: 서버는 401 Unauthorized 오류를 반환합니다. 클라이언트는 이 응답을 감지하고, 저장된 Refresh Token을 사용해 Access Token 갱신을 요청합니다.
5. 갱신에 성공하면, 클라이언트는 새로운 Access Token을 저장하고 실패했던 원래 요청을 다시 시도합니다.

---

## 2. 자동 로그인 과정의 함정: 경합 상태 (Race Condition)

### 2.1 문제 상황: 동시 토큰 재발급

여러 API 요청을 처리하는 코루틴(Thread)들이 거의 동시에 Access Token 만료에 따른 401 Unauthorized 응답을 받게 되면, 모든 코루틴이 각자 토큰 재발급 로직을 실행하려 합니다.
- 코루틴 A가 먼저 토큰 재발급을 요청하여 새로운 Access/Refresh Token을 받아 저장합니다.
- 코루틴 B는 아직 이 사실을 모르고, 만료된 Refresh Token으로 재발급을 요청합니다.
- 서버는 이미 사용된(혹은 무효화된) Refresh Token을 감지하고 요청을 거부합니다.
결과적으로 클라이언트의 토큰 상태가 꼬이거나, 서버가 보안 정책에 따라 모든 토큰을 무효화하여 사용자가 강제 로그아웃되는 문제가 발생합니다.

---

## 3. 해결책: Mutex를 이용한 동기화

이러한 경합 상태 문제를 해결하기 위해 뮤텍스(Mutex)를 사용해 토큰 재발급 로직을 동기화해야 합니다. Mutex는 한 번에 하나의 코루틴만 접근할 수 있는 **임계 구역(Critical Section)**을 만들어 공유 자원(토큰)의 일관성을 보장합니다.

### 3.1 Mutex 적용 원리

토큰 재발급 로직을 Mutex로 감싸면 다음과 같이 동작합니다.
- 여러 코루틴이 동시에 401 에러를 받습니다.
- 모두 토큰 재발급 로직에 진입하려 하지만, Mutex가 잠겨있어 오직 하나의 코루틴만 접근할 수 있습니다.
- 코루틴 A가 Mutex를 획득하고 재발급 로직을 실행합니다. 나머지 코루틴은 락이 해제될 때까지 대기합니다.
- 코루틴 A가 새 토큰을 받아 저장하고 락을 해제합니다.
- 코루틴 B가 락을 획득하고 진입하면, 이미 토큰이 갱신된 것을 확인하고 재발급을 다시 시도하지 않고 새로운 토큰으로 원래의 요청을 진행합니다.

### 3.2 안드로이드 (Kotlin/Jetpack Compose) 구현 예시
kotlinx.coroutines.sync.Mutex를 사용하여 토큰 갱신 로직을 보호하는 예시입니다.

TokenManager.kt
```kotlin
import kotlinx.coroutines.flow.firstOrNull
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock
import okhttp3.Authenticator
import okhttp3.Request
import okhttp3.Response
import okhttp3.Route

/**
 * OkHttp의 Authenticator 인터페이스를 구현하여 401 Unauthorized 응답을 처리합니다.
 * 토큰이 만료되었을 때, 자동으로 Refresh Token을 사용해 Access Token을 갱신합니다.
 */
class TokenAuthenticator(
    private val tokenRepository: TokenRepository,
    private val authRepository: AuthRepository
) : Authenticator {

    // 토큰 재발급 로직에 대한 동시 접근을 막기 위한 Mutex
    private val tokenRefreshMutex = Mutex()

    /**
     * 401 응답을 받았을 때 OkHttp에 의해 호출되는 메서드입니다.
     */
    override fun authenticate(route: Route?, response: Response): Request? {
        return runBlocking {
            handleAuthenticator(response)
        }
    }

    private suspend fun handleAuthenticator(response: Response): Request? {
        // Mutex를 획득하여 토큰 갱신 로직이 한 번에 하나씩만 실행되도록 보장합니다.
        return tokenRefreshMutex.withLock {
            val currentAccessToken = tokenRepository.getAccessToken().firstOrNull()
            val requestToken = getRequestToken(response.request)

            // 1. 이미 다른 코루틴이 토큰을 재발급했는지 확인합니다.
            //    현재 요청의 토큰과 저장된 토큰이 다르다면, 이미 갱신된 것입니다.
            if (isTokenRefreshed(requestToken, currentAccessToken)) {
                // 갱신된 토큰으로 새 요청을 만들어서 반환합니다.
                return@withLock buildRequestWithToken(response.request, currentAccessToken.orEmpty())
            }

            // 2. Refresh Token이 있는지 확인합니다. 없으면 로그아웃 상태이므로 null을 반환합니다.
            val refreshToken = tokenRepository.getRefreshToken().firstOrNull()
            if (refreshToken.isNullOrBlank()) {
                // 토큰이 없으므로 로그아웃 처리
                tokenRepository.clearTokens()
                return@withLock null
            }

            // 3. Refresh Token으로 새로운 토큰을 발급받습니다.
            val newToken = authRepository.refreshToken(refreshToken).getOrNull()

            // 4. 토큰 재발급에 실패한 경우
            if (newToken == null) {
                // Refresh Token도 만료되었을 가능성이 높으므로 토큰을 모두 제거합니다.
                tokenRepository.clearTokens()
                return@withLock null
            }

            // 5. 토큰 재발급 성공
            tokenRepository.updateTokens(newToken)
            // 새로운 Access Token으로 요청을 다시 만듭니다.
            return buildRequestWithToken(response.request, newToken.accessToken)
        }
    }

    // 요청 헤더에서 토큰을 추출하는 헬퍼 함수
    private fun getRequestToken(request: Request): String? =
        request.header(AUTHORIZATION)?.removePrefix(BEARER)?.trim()

    // 토큰이 이미 갱신되었는지 확인하는 헬퍼 함수
    private fun isTokenRefreshed(requestToken: String?, currentAccessToken: String?): Boolean {
        if (currentAccessToken.isNullOrBlank()) return false
        return requestToken != currentAccessToken
    }

    // 새로운 토큰으로 요청 헤더를 업데이트하는 헬퍼 함수
    private fun buildRequestWithToken(request: Request, token: String): Request =
        request.newBuilder()
            .removeHeader(AUTHORIZATION)
            .addHeader(AUTHORIZATION, "$BEARER $token")
            .build()

    companion object {
        private const val AUTHORIZATION = "Authorization"
        private const val BEARER = "Bearer"
    }
}
```

---

## 4. 결론
소셜 로그인 자동 로그인 구현은 단순히 API 통신을 넘어 동기화(Synchronization)라는 운영체제의 핵심 개념을 필요로 합니다. Mutex를 활용해 경합 상태를 해결함으로써, 우리는 사용자에게 안정적인 로그인 경험을 제공하고 불필요한 강제 로그아웃을 방지할 수 있습니다.

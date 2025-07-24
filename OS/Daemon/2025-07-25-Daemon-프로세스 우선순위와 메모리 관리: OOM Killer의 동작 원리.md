---
date: 2025-07-25
user: Daemon
topic: "프로세스 우선순위와 메모리 관리: OOM Killer의 동작 원리"
---

## 메모리 부족은 왜 발생할까?

보통 메모리(RAM)이 부족한 상황은 다중 프로그램 실행시 크롬 브라우저 탭을 무리하게 XX개 띄워놓는다든지 말 그대로 동시에 여러 프로그램을 실행시키면 발생하는 상황이다.

## OS의 메모리 관리 전략

메모리가 부족해지면 운영체제는 다음과 같은 순서로 대응한다.

1. 캐시 정리 -> 캐시는 일시적인 데이터일뿐이기 때문에 먼저 가벼운 것부터 없애버림
   - 파일 시스템 캐시 해제(파일 Read, Write 동작시 메모리에 캐시해둔 것을 해제시킴)
   - 이미지 캐시, 웹 캐시 해제(웹 브라우저 방문할 때 이미지나 CSS 같은 파일을 로컬 저장해둔 것을 해제시킴)
2. 스왑(swap) 사용
   - OS는 물리 메모리에 쌓여있는 페이지들 중에서 오랫동안 사용되지 않은 것부터 찾아내서 디스크의 스왑 공간으로 복사하고 물리 메모리에서는 해당 페이지를 해제해버림
   - 그러면 물리 메모리가 확보가 되겠죠? 이렇게 확보된 공간은 메모리가 당장 필요한 다른 프로세스에게 할당됨
   - 스왑 아웃되었던 기존 페이지에 다시 접근해야한다면 OS는 스왑 공간을 인지해서 다시 불러옴
   - 용량 한계를 돌파하는 것은 좋다만, 애초에 디스크는 RAM보다 훨씬 느리므로 스왑 인/아웃이 자주 발생한다? 성능이 엄청 저하됨   
3. 프로세스 종료
   - 1, 2번의 캐시 정리와 스왑 사용으로도 메모리 부족 문제가 해결되지 않는다면 OS는 비장의 카드인 "강제 종료" 이것이 OOM Killer의 역할임
   - 메모리 고갈 감지를 해서 가용 메모리도 0이고 스왑 공간도 0이면 OOM을 선언함
   - 이제 뭐부터 숙청시킬지 골라야하겠죠? 현재 실행중인 프로세스를 평가해서 oom_score라는 점수를 매김
   - oom_score는 메모리량, 실행 시간, 중요도(nice값)을 바탕으로 종합 평가함
   - 그래서 SIGKILL이라는 사형 선고를 보내면 강제 종료가 됨

#### "캐시 정리로 가벼운 것부터 정리, 스왑으로 안 쓰는 것은 잠시 넣어두고, 프로세스 종료를 통해 최후에는 가장 큰 짐을 버리게 됨"

## OOM Killer의 고질적인 문제

일반적인 OOM Killer는 메모리가 완전히 고갈된 후에 움직이기 때문에 너무 늦게 동작한다.

모바일 앱에서 이렇게 되면 앱이 버벅거리고, 화면이 뻑나가고, 사용자는 이탈한다.

그래서 안드로이드는 미리미리 메모리를 관리해야하기 때문에 LMK를 사용한다.

### 나쁜 예시
```kotlin
class BadActivity : AppCompatActivity() {
    private var timer: Timer? = null
    private var handler = Handler(Looper.getMainLooper())
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 타이머가 Activity를 참조하고 있어서 메모리 누수 발생!
        timer = Timer()
        timer?.schedule(object : TimerTask() {
            override fun run() {
                handler.post {
                    // Activity가 종료되어도 이 코드가 실행될 수 있음
                    updateUI()
                }
            }
        }, 0, 1000)
    }
    
    private fun updateUI() {
        // UI 업데이트
    }
    
    // onDestroy에서 정리하지 않으면 메모리 누수!
}
```

### 좋은 예시
```kotlin
class GoodActivity : AppCompatActivity() {
    private var timer: Timer? = null
    private var handler = Handler(Looper.getMainLooper())
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        timer = Timer()
        timer?.schedule(object : TimerTask() {
            override fun run() {
                // Activity가 여전히 유효한지 확인
                if (!isDestroyed && !isFinishing) {
                    handler.post {
                        updateUI()
                    }
                }
            }
        }, 0, 1000)
    }
    
    override fun onDestroy() {
        super.onDestroy()
        
        // 🔧 메모리 누수 방지를 위한 정리 작업
        timer?.cancel()
        timer = null
        handler.removeCallbacksAndMessages(null)
    }
    
    private fun updateUI() {
        // UI 업데이트
    }
}
```

#### 실제로 Fragment에 연결되어있는 binding을 해제하기 위해서 필자는 onDestroy에서 _binding을 null로 선언하면서 메모리 누수를 방지한다.

## 프로세스 우선순위

OS는 여러 프로세스들이 동시에 실행되는 멀티태스킹 환경에서 각 프로세스에 자원을 얼마나 할당할지 결정해야한다.

리눅스에서의 우선순위는 위에서 언급한 nice값을 기준으로 잡는데, 범위는 -20 ~ +19까지고 우선순위와 값은 반비례한다.

## 그래서 LMK는 OOM Killer랑 뭐가 다른가
<img width="500" height="1204" alt="image" src="https://github.com/user-attachments/assets/4bf60b19-5e36-4371-acdf-a5c91f4acf00" />

번외로 Low Memory Killer Daemon은 다음 이미지와 같은 순서로 oom_adj_score라는 점수로 지정한다.

사실은 LMK는 일반적인 리눅스 OOM Killer와 매우 유사하지만 단지 모바일 환경이라는 특성을 고려해서 작동하는 것뿐이지 비슷하다.

사진처럼 안드로이드 프레임워크의 생명주기와 긴밀히 연동되어있고 Foreground Service인지, Service인지, Background Service인지 따라서 중요도도 갈린다.

1. lowmemkiller 드라이버가 주기적으로 시스템의 가용 물리 메모리량을 체크함
2. 가용 메모리가 임계치 이하로 떨어지면 LMK 활성화됨
3. oom-_adj_score를 기반으로 앱을 종료시킴

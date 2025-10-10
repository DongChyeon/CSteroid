---
date: 2025-09-26
user: Daemon
topic: "MVVM< MVI 패턴"
---

## MVW (Model-View-Whatever) 패턴

### 핵심 구성요소와 역할

**Model 계층**

- Repository 패턴과 함께 사용되어 데이터 소스를 추상화
- Local Database (Room), Remote API (Retrofit), SharedPreferences 등 다양한 데이터 소스 관리
- Domain Layer의 UseCase/Interactor를 통해 비즈니스 로직 분리

**View 계층 (Activity/Fragment/Composable)**

- UI 렌더링과 사용자 입력 처리
- ViewModel의 상태를 관찰(observe)하여 UI 업데이트
- Data Binding 또는 View Binding 활용
- Jetpack Compose에서는 State hoisting 원칙 적용

**ViewModel 계층**

- UI 관련 데이터 홀더 및 상태 관리
- LiveData, StateFlow, SharedFlow를 통한 반응형 프로그래밍
- Configuration Change에도 생존 (ViewModelScope)
- SavedStateHandle을 통한 프로세스 종료 복구

### MVVM의 데이터 흐름 메커니즘

```kotlin
*// ViewModel에서의 단방향 데이터 흐름*
class ProductViewModel(
    private val repository: ProductRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    *// StateFlow를 사용한 상태 관리*
    private val _uiState = MutableStateFlow(ProductUiState())
    val uiState: StateFlow<ProductUiState> = _uiState.asStateFlow()
    
    *// 사용자 액션 처리*
    fun onAddToCart(productId: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            
            repository.addToCart(productId)
                .flowOn(Dispatchers.IO)
                .catch { error ->
                    _uiState.update { 
                        it.copy(
                            isLoading = false,
                            error = error.message
                        )
                    }
                }
                .collect { result ->
                    _uiState.update {
                        it.copy(
                            isLoading = false,
                            cartItems = result
                        )
                    }
                }
        }
    }
}
```

### MVVM의 고급 구현 패턴

**1. Multi-Module 환경에서의 MVVM**

- Feature 모듈별 ViewModel 분리
- Shared ViewModel을 통한 모듈 간 통신
- Navigation Component와의 통합

**2. StateFlow vs LiveData 선택 기준**

- StateFlow: 초기값 필수, 순수 Kotlin, Cold Stream
- LiveData: Android 전용, Lifecycle-aware, 초기값 옵션
- SharedFlow: Hot Stream, 이벤트 처리에 적합

## MVI (Model-View-Intent) 패턴 심화

### MVI의 핵심 철학

**단일 상태 원칙 (Single State Principle)**

```kotlin
data class ScreenState(
    val isLoading: Boolean = false,
    val products: List<Product> = emptyList(),
    val error: String? = null,
    val isRefreshing: Boolean = false,
    val selectedFilter: FilterType = FilterType.ALL,
    val cartItemCount: Int = 0
)
```

**Intent (사용자 의도) 모델링**

```kotlin
sealed interface UserIntent {
    object LoadProducts : UserIntent
    data class SelectProduct(val id: String) : UserIntent
    data class ApplyFilter(val filter: FilterType) : UserIntent
    object RefreshProducts : UserIntent
    data class AddToCart(val productId: String) : UserIntent
}
```

### MVI의 상태 변환 메커니즘

```kotlin
class ProductMviViewModel(
    private val repository: ProductRepository
) : ViewModel() {
    
    private val _state = MutableStateFlow(ScreenState())
    val state: StateFlow<ScreenState> = _state.asStateFlow()
    
    private val _intent = Channel<UserIntent>(Channel.UNLIMITED)
    
    init {
        handleIntents()
    }
    
    private fun handleIntents() {
        viewModelScope.launch {
            _intent.consumeAsFlow().collect { intent ->
                when (intent) {
                    is UserIntent.LoadProducts -> loadProducts()
                    is UserIntent.SelectProduct -> selectProduct(intent.id)
                    is UserIntent.ApplyFilter -> applyFilter(intent.filter)
                    is UserIntent.RefreshProducts -> refreshProducts()
                    is UserIntent.AddToCart -> addToCart(intent.productId)
                }
            }
        }
    }
    
    private suspend fun loadProducts() {
        _state.update { it.copy(isLoading = true) }
        
        repository.getProducts()
            .fold(
                onSuccess = { products ->
                    _state.update { 
                        it.copy(
                            isLoading = false,
                            products = products,
                            error = null
                        )
                    }
                },
                onFailure = { throwable ->
                    _state.update {
                        it.copy(
                            isLoading = false,
                            error = throwable.message
                        )
                    }
                }
            )
    }
    
    fun processIntent(intent: UserIntent) {
        viewModelScope.launch {
            _intent.send(intent)
        }
    }
}
```

### MVI의 Side Effect 처리

```kotlin
*// Side Effect를 위한 별도 채널*
sealed interface SideEffect {
    data class ShowToast(val message: String) : SideEffect
    data class NavigateToDetail(val productId: String) : SideEffect
    object NavigateBack : SideEffect
}

class EnhancedMviViewModel : ViewModel() {
    private val _sideEffect = Channel<SideEffect>()
    val sideEffect = _sideEffect.receiveAsFlow()
    
    private fun handleAddToCartSuccess() {
        viewModelScope.launch {
            _sideEffect.send(SideEffect.ShowToast("장바구니에 추가되었습니다"))
        }
    }
}
```

## MVVM vs MVI 심층 비교

### 상태 관리 철학의 차이

**MVVM**: 여러 개의 관찰 가능한 속성

- 각 UI 요소가 개별 LiveData/StateFlow 관찰
- 부분적 상태 업데이트 가능
- 상태 조합의 복잡성 증가 가능

**MVI**: 단일 불변 상태

- 전체 화면 상태를 하나의 객체로 관리
- 상태 예측 가능성 향상
- 디버깅과 테스팅 용이

### 실제 구현 시 고려사항

**MVVM 선택 시나리오**:

- 간단한 CRUD 앱
- 빠른 프로토타이핑
- 기존 레거시 코드베이스
- 작은 팀 또는 주니어 개발자가 많은 경우

**MVI 선택 시나리오**:

- 복잡한 비즈니스 로직
- 상태 일관성이 중요한 금융/의료 앱
- 테스트 커버리지가 중요한 프로젝트
- 대규모 팀 협업

### 성능 최적화 관점

**MVVM 최적화**:

```kotlin
*// distinctUntilChanged로 불필요한 recomposition 방지*
val products = repository.getProducts()
    .distinctUntilChanged()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = emptyList()
    )
```

**MVI 최적화**:

```kotlin
*// State diffing으로 성능 향상*
@Immutable
data class OptimizedState(
    val list: ImmutableList<Product>,
    val metadata: StateMetadata
)
```

### 테스팅 전략

**MVVM 테스트**:

```kotlin
@Test
fun `제품 로딩 시 상태 변경 확인`() = runTest {
    *// Given*
    val products = listOf(Product("1", "상품1"))
    coEvery { repository.getProducts() } returns flowOf(products)
    
    *// When*
    viewModel.loadProducts()
    
    *// Then*
    assertEquals(products, viewModel.products.first())
    assertFalse(viewModel.isLoading.first())
}
```

**MVI 테스트**:

```kotlin
@Test
fun `Intent 처리 및 State 변환 확인`() = runTest {
    *// Given*
    val initialState = ScreenState()
    
    *// When*
    viewModel.processIntent(UserIntent.LoadProducts)
    
    *// Then*
    viewModel.state.test {
        assertEquals(initialState.copy(isLoading = true), awaitItem())
        assertEquals(initialState.copy(isLoading = false, products = products), awaitItem())
    }
}
```

## 실무 적용 팁

1. **마이그레이션 전략**: MVVM에서 MVI로 점진적 전환 시, Feature 단위로 진행
2. **하이브리드 접근**: 복잡한 화면은 MVI, 간단한 화면은 MVVM
3. **Compose와의 통합**: MVI가 Compose의 단방향 데이터 흐름과 더 자연스럽게 맞음
4. **도구 활용**:
    - Orbit MVI 라이브러리
    - Mobius 프레임워크
    - Redux-Kotlin

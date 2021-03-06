# Go 동시성 프로그래밍

2019, 지은이 캐서린 콕스 부데이, 옮긴이 이상식

[1. 동시성 소개](#1장-동시성-소개)  
[2. 코드 모델링: 순차적인 프로스세 간의 통신](#2장-코드-모델링:-순차적인-프로스세-간의-통신)

## 1장 동시성 소개

[동시성](https://ko.wikipedia.org/wiki/%EB%B3%91%ED%96%89%EC%84%B1)은 여러 프로세스가 concurrent 동시적으로 수행하는 것을 의미한다.

### _동시성이 어려운 이유_

1. race condition

   - 작업들이 올바른 순서로 실행돼야하지만 프로그램이 그렇게 작성되지 않아서 순서 유지가 보장 되지 않을 때 발생한다.
   - 한 동시 작업이 어떤 변수를 읽으려고 시도하는 동안 다른 동시 작업이 어떤 시점에 값을 쓰는 data race인 경우가 많다.
   - Sleep 사용은 해결책이 아니다. data race 가능성을 낮추지만 발생하고 오랜 시간 Sleep 하면 비효율성이 발생한다.

2. atomic

   - [원자성](https://ko.wikipedia.org/wiki/%EC%9B%90%EC%9E%90%EC%84%B1)은 동작하는 컨텍스트 내에서 나누어지거나 중단되지 않는것이다.
   - 컨텍스트 context 에 따라 원자적이었던 연산이 원자적이지 않을 수 있다. 연산의 원자성은 현재 정의된 범위에 따라 달라진다.
   - 불가분 indivisible, 중단불가 uninterruptible은 사용자가 정의한 컨텍스트 내부에서 원자적인 요소가 통째로 발생해 해당 요소 외에 어떤 것도 동시에 이뤄지지 않는 것이다.
   - 원자적 연산은 조합한다고 해서 반드시 더 큰 원자적 연산이 되는 건 아니다.

3. 메모리 접근 동기화

   - 임계영역은 공유 리소스에 독점적으로 점근해야하는 영역이다.
   - 임계영역을 발견하면 메모리에 대한 접근을 동기화하기 위한 포인트를 추가해야 한다.

4. deadlock

   - 데드락은 동시 실행중인 프로세스들이 다른 프로세스를 기다리면서 발생한다.

   ```Go
   type value struct {
     mu sync.Mutex
     value int
   }

   var wg sync.WaitGroup

   func sum(a,b *value){
     defer wg.Done()
     a.mu.Lock()
     defer a.mu.Unlock()

     time.Sleep(1*time.Second)
     b.mu.Lock()
     defer b.mu.Unlock()

    fmt.Println(a+b)
   }

   a,b := 1, 2
   wg.Add(2)
   go sum(&a, &b) // a 리소스가 lock
   go sum(&b, &a) // b 리소스가 lock
   // a와 b가 lock 상태이므로 데드락 발생
   wg.Wait()
   ```

   - [코프먼 조건(데드락 발생 조건)](https://ko.wikipedia.org/wiki/%EA%B5%90%EC%B0%A9_%EC%83%81%ED%83%9C#%EA%B5%90%EC%B0%A9_%EC%83%81%ED%83%9C%EC%9D%98_%EC%A1%B0%EA%B1%B4)  
     이 조건들 중 하나라도 만족하지 못하면 데드락을 막을 수 있다.
     - 상호배제
     - 대기조건
     - 비선점
     - 순환대기

5. livelock

   프로그램들이 활동적 동시에 연산을 수행하고 있지만 실제로 프로그램 상태 진행을 하지 못하는 상태이다. 또한 CPU 사용률을 보면 사용 중이기 때문에 알아차리기 힘들다.

   [ZepehWAVE, 데드락(Deadlock)과 라이브락(Livelock)](https://zepeh.tistory.com/196)

6. [기아 상태](https://ko.wikipedia.org/wiki/%EA%B8%B0%EC%95%84_%EC%83%81%ED%83%9C)

   동시 프로세스가 작업을 수행하는데 필요한 모든 리소스를 얻을 수 없는 모든 상황을 의미한다.

---

## 2장 코드 모델링: 순차적인 프로스세 간의 통신

### _동시성 병행성 차이_

동시성은 코드 속성이고 병렬 처리는 실행중인 프로그램 속성이다. 프로그램은 병렬적으로 실행되는 것처럼 보이지만 CPU 컨텍스트가 전환되면서 작업이 병렬적으로 실행되는 것처럼 보일 뿐이다.

- 병렬로 실행되기를 바라며 동시성 코드를 작성한다.
- 동시성 코드가 실제로 병렬로 실행되는지 여부는 모른다.
- 병렬 처리 여부는 시간이나 컨텍스트로 결정된다. 여기서 컨텍스트는 연산들이 병렬적으로 실행되는 범위이다. 50pg 예시 확인 필요

### _CSP란_

상호 작용하는 순차적 프로세스들, Communicating Sequential Processes  
[Hoare의 CSP 논문 PDF](https://www.cs.cmu.edu/~crary/819-f09/Hoare78.pdf)

### _동시성을 지원하는 언어 장점_

다른 프로그래밍 언어는 OS 스레드와 메모리 접근 동기화가 추상화 수준에서 끝나지만 Golang은 고루틴과 채널로 이를 현실화? 한다. 고루틴은 스레드, 채널은 뮤텍스와 비교할 수 있다. 고루틴은 병렬성을 고민하지 않게 하고 동시성을 지원한다.

### _동시성, 병렬성 분리 장점_

Go 런타임이 고루틴 스케줄링 관리를 해서 고루틴이 I/O를 기다리면서 멈춰있는지를 검사하고 멈춰있지 않은 OS 스레드로 지능적으로 재할당한다. 이런 방법은 코드 성능을 향상시킨다.

### _동시성에 대한 Golang 철학_

Go는 sync 패키지에서 lock, mutex 같은 전통적인 동시성 코드 작성을 지원한다. 하지만 "통신을 통해 메모리를 공유하고 메모리 공유를 통해 통신하지 말라" 라고 Go 개발 팀은 주장한다.

#### _동시성 결정 트리_

1. 데이터 소유권을 이전하려고 하는가?

   채널은 한 번에 한 동시 컨텍스트만 데이터 소유권을 가져야한다. 이를 위해 채널 타입을 인코등함으로써 메모리 소유권 개념을 전달할 수 있게 도와준다. 이에 따른 장점은 첫째, 적은 비용으로 메모리 내부 큐 구현을 위한 버퍼링된 채널을 생성한다. 둘째, 동시성 코드를 다른 동시성 코드와 함께 구성할 수 있다.

2. 구조체 내부 상태를 보호하고자 하는가?

   구조체 내부 상태를 보존하고자 하는 상황은 기본 메모리 접근 동기화 요소를 사용하기를 추천한다.

3. 여러 부분의 논리를 조정해야 하는가?

   채널은 Go의 대기열로 제공되고 select 문은 안전하게 전달해주는 기능을 하고 복잡성을 쉽게 제어한다.

4. 성능상의 임계영역인가?

   채널 동작시 메모리 접근 동기화를 사용하기 때문에 채널이 더 느릴 수 있다. 특정 영역이 다른 부분보다 느린 병목지점으로 되어 있다면 기존 메모리 접근 동기화 요소를 사용한다.

### Go의 철학은 단순한 목표를 가지고 채널을 최대한 활용하고 고루틴 사용에 제한을 두지 말라는데 있다

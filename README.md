# 멀티 플레이 위치 & 회전 동기화
### 위치보정 이슈 처리
멀티플레이 환경을 고려하며 개발하며 `RootMotion`과 회전을 같이 사용하는 과정에서 캐릭터의 움직임이 버벅이는 보정 현상이 발생했습니다.  
정상적인 범주의 핑(20ms)에서도 이런 현상이 발생했기 때문에 유저의 플레이 경험과 반응성 측면에서 개선하고자 했습니다.

**프로젝트에 정의한 회전 타입**
| Type                          | Description                                                |
| ----------------------------- | ---------------------------------------------------------- |
| ERotateTaskRule::InputVector  | 입력한 방향으로 회전합니다.                                 |
| ERotateTaskRule::Camera       | 카메라 방향으로 회전합니다.                                 |
| ERotateTaskRule::TrackCamera  | 지속적으로 카메라 방향으로 회전합니다.                      |

### 발생 원인
<hr>

`Character Movement Component(CMC)`는 기본적으로 `Client-Prediction` 방식으로 동작합니다.

1. 클라이언트는 이동을 수행하고 `FSavedMove_Character` 객체에 기록하여 서버에 전송합니다.
2. 서버에서는 받은 정보를 이용해 시뮬레이션 하고 클라이언트의 이동이 유효한지 체크합니다.        
3. 유효하지 않다면 클라이언트에 보정을 요청합니다.    

위 프로세스에서 디버깅 과정을 통해 서버의 시뮬레이션 결과가 클라이언트와 달랐기 때문에 발생하여 발생한 문제임을 파악했습니다.


### 무엇이 문제였나?
<hr>

<details>
<summary> 코드 접기 / 펼치기 </summary>
<img src="https://github.com/BUOMACC/ProjectV_SourceCode/blob/main/Images/CMC_01.png" width="100%" height="100%"/>
</details>

근본적인 문제는 작성한 임시 회전 로직에 있었습니다.

`Server` / `Client`의 프레임이 각각 다르고 지연시간이 커지는 경우 오차가 누적되어 잘못된 움직임으로 판단되어 보정이 발생하게 됩니다.  
즉, 위 코드는 `Client` / `Server`의 환경을 고려하지 않은 채로 각각 따로 수행되면서 동기화가 어긋나게 되었습니다.


### 해결 방법
<hr>

생각해 볼 수 있었던 방식은 두 가지가 있었습니다.

1. 클라이언트의 움직임을 특정 상황에만 수용하도록 한다.    
> - 특정 상황에서만 활성화한다고 하더라도, 그 사이에 잘못된 이동이 올 수 있다.    
> - 임의의 회전에 대해서 여전히 회전 불일치가 발생할 여지가 있어 이를 맞춰야 한다.  
>   CMC 내부적으로 클라이언트의 회전 값을 그대로 수용하진 않게 되어있어 따로 처리해 주어야 함을 의미    
> - 클라이언트의 환경(주로 인터넷 환경)에 따라 다른 플레이어에게는 순간이동하는 것처럼 보일 수 있다.    
2. 서버 측에 클라이언트의 회전을 동일하게 적용하여 시뮬레이션 결과를 맞춰본다.    
> - 서버에서 검증하는 방식이기 때문에 잘못된 이동을 확실하게 처리할 수 있다.    
> - CMC에서 제공하는 프로세스만 잘 따라준다면 좋은 반응성과 치트 방지가 모두 가능하다.    

위와 같은 장단점을 고려했을 때 `(2)` 번의 방식으로 처리하는 것이 치트 및 조작감 측면에서 적합하다고 판단했습니다.

```cpp
if (bAllowPhysicsRotationDuringAnimRootMotion || !HasAnimRootMotion())
{
    PhysicsRotation(DeltaSeconds);
}
```
마침 언리얼에서 제공하는 클라이언트 예측을 위한 회전 코드가 있었기에 이를 이용해서 활용하였습니다.

### 첫 시도
<hr>

<details>
<summary> 코드 접기 / 펼치기 </summary>
<img src="https://github.com/BUOMACC/ProjectV_SourceCode/blob/main/Images/CMC_02.png" width="75%" height="75%"/>
<img src="https://github.com/BUOMACC/ProjectV_SourceCode/blob/main/Images/CMC_03.png" width="100%" height="100%"/>
<img src="https://github.com/BUOMACC/ProjectV_SourceCode/blob/main/Images/CMC_04.png" width="85%" height="85%"/>
</details>

`회전 방향(DesiredTargetRotation)`과 `회전량(RotateSpeedPerSec)`, `Flag`를 서버와 클라이언트에서 각자 설정하고,  
회전 로직을 담당하는 `PhysicsRotation()` 내부에서 설정된 값을 활용해 회전을 처리해 주는 방식으로 처리했습니다.

### 첫 시도 결과
<hr>

처음에 비해 낮은 지연시간에서 보정되는 현상이 줄었으나 지연시간이 높은 경우 여전히 발생하고 있었습니다.  

### 첫 시도 실패 원인 분석 
<hr>

왜 이런 문제가 재발했는지 알아보기 위해 로그를 사용하여 분석했습니다.  

<img src="https://github.com/BUOMACC/ProjectV_SourceCode/blob/main/Images/CMC_05.png" width="75%" height="75%"/>

보정이 일어나는 환경으로 로그를 확인해 본 결과, 마지막 한 번의 연산으로 보정 현상이 발생했음을 알 수 있었습니다.  
CMC에서 확장한 회전 로직을 통한 연산값 `Yaw`는 `Server`와 `Client`가 일치했기 때문에 로직 자체는 정상적으로 동작함을 확인할 수 있었으며  
Flag 값을 `Server` / `Client` 환경에서 각각 설정하여 `Server`의 Flag 값이 늦게 비활성화되어 발생한 문제라고 가설을 세웠습니다.  


### 두 번째 시도
<hr>

두 번째 시도에서는 Flag 값을 `Client`의 값으로 동기화 시키도록 하는 방식으로 처리하였습니다.
<details>
<summary> 코드 접기 / 펼치기 </summary>
<img src="https://github.com/BUOMACC/ProjectV_SourceCode/blob/main/Images/CMC_06.png" width="100%" height="100%"/>
<img src="https://github.com/BUOMACC/ProjectV_SourceCode/blob/main/Images/CMC_07.png" width="75%" height="75%"/>
</details>

CMC에서 `Client` -> `Server`로 상태 값을 전달하기 위해 사용되는 `FSavedMove`와 `FNetworkPrediction_Client`를 상속받고,  
`FLAG_Custom`을 활용해서 동기화된 Flag는 `bWantsToRootMotionRotation` 변수에 저장하여 회전을 사용하고자 할 때 설정해 주었습니다.


### 두 번째 시도 결과 및 문제
<hr>

결과적으로 높은 핑에서도 보정 현상이 발생하지 않고 `RootMotion`의 회전이 정상적으로 동작했습니다.    
하지만 여전히 두 가지 문제가 남아있었습니다.

1. Listen Server 환경에서 클라이언트의 동기화가 틀어지는 문제
> 원인을 파악하는 데 시간은 걸렸으나, 해결은 비교적으로 단순했습니다.    
  Listen Server의 경우 부드럽게 회전하는것처럼 보이기 위해 최종 값이 아닌 보간 값을 사용하기 때문이었는데     
  `ListenServerNetworkSimulatedSmoothRotationTime` 값을 조정하는 것으로 해결했습니다.    
2. 지속적으로 회전하는 타입(`ERotateTaskRule::TrackCamera`)에서는 동기화가 틀어지는 문제


### 두 번째 시도 원인 분석
<hr>

지금까지 작업한 내용은 회전을 사용하고자 할 때 목표 회전 값과 회전량을 `Client` / `Server` 각각에서 최초로 한번 설정하고 이후에는 변경할 일이 없기 때문에 문제가 없었지만,      

지속적으로 회전하고자 하는 환경(`ERotateTaskRule::TrackCamera`)에서는 이 두 개의 값이 지속적으로 변경되기 때문에 `Client`의 값을 `Server`에 동기화해주어야 했습니다.        

<details>
<summary> 코드 접기 / 펼치기 </summary>
<img src="https://github.com/BUOMACC/ProjectV_SourceCode/blob/main/Images/CMC_08.png" width="75%" height="75%"/>
<img src="https://github.com/BUOMACC/ProjectV_SourceCode/blob/main/Images/CMC_09.png" width="100%" height="100%"/>
</details>

이를 해결하기 위해 `CMC`에 추가 데이터를 전달하기 위한 용도로 제공되는 `FCharacterNetworkMoveData` 구조체를 사용하도록 했습니다.    
이후 서버에서 이동을 처리하기 직전 `MoveAutonomous( )`에서 서버로 전달된 데이터를 기반으로 `CMC` 정보를 업데이트하여 동기화를 맞췄습니다.

### 최종 결과
<hr>

Flag 및 목표 회전이 `Client`에 의해 동기화되고 서버에서 이를 검증하기 때문에 모든 동작이 정상적으로 잘 동작했습니다.    
높은 지연시간에도 `RootMotion`을 재생하는 도중 회전할 방향을 실시간으로 설정할 수 있으며, `Client`의 위치 치팅을 방지함과 동시에 유저경험 역시 크게 증가했습니다.

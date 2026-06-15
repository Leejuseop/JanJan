# DB 최소 변경 음주 카운트 설계

이 문서는 기존 Firebase DB 구조를 최대한 유지하면서 YOLO 기반 음주 카운트를 붙일 때 DB가 어떻게 바뀌는지 설명한다.

이번 설계에서는 새 카운트 전용 컬렉션을 크게 늘리지 않는다.

- `drinkCounters`는 사용하지 않는다.
- `drinkRecords`도 사용하지 않는다.
- 기존 `sessions`, `participants`, `glassMappings`를 중심으로 카운트를 저장한다.

## 핵심 결론

사용자가 소주 또는 맥주를 마시면 아래 3곳만 같이 업데이트한다.

```text
sessions/{sessionId}
sessions/{sessionId}/participants/{userId}
sessions/{sessionId}/glassMappings/{mappingId}
```

즉, 새 기록 컬렉션을 만들지 않고 기존 문서에 누적값을 추가하는 방식이다.

## 현재 DB에서 그대로 사용할 것

현재 DB에 이미 있는 구조는 그대로 사용한다.

```text
sessions/{sessionId}
```

세션 전체 정보를 저장하는 문서다.

이미 있는 주요 필드:

```text
sessionId
storeId
storeName
tableId
tableNumber
inviteCode
status
participantCount
totalSojuDrinkCount
totalBeerDrinkCount
```

```text
sessions/{sessionId}/participants/{userId}
```

세션에 참여한 사용자 정보를 저장하는 문서다.

이미 있는 주요 필드:

```text
userId
userName
glassColor
glassMappingType
joinedAt
```

```text
sessions/{sessionId}/glassMappings/{mappingId}
```

사용자의 색상 화면과 실제 술잔 매핑 정보를 저장하는 문서다.

이미 있는 주요 필드:

```text
userId
drinkType
screenColorHex
glassId
mappingSource
drinkCount
createdAt
```

## 새로 추가할 필드

새 컬렉션을 만드는 대신 기존 `participants` 문서에 카운트 필드를 추가한다.

```text
sessions/{sessionId}/participants/{userId}
  sojuDrinkCount: 0
  beerDrinkCount: 0
  lastDrinkType: null
  lastDrinkAt: null
  updatedAt: Timestamp
```

각 필드의 의미:

| 필드 | 의미 |
| --- | --- |
| `sojuDrinkCount` | 해당 사용자가 이 세션에서 마신 소주잔 수 |
| `beerDrinkCount` | 해당 사용자가 이 세션에서 마신 맥주잔 수 |
| `lastDrinkType` | 마지막으로 카운트된 주종. `soju` 또는 `beer` |
| `lastDrinkAt` | 마지막으로 카운트된 시간 |
| `updatedAt` | participant 문서가 마지막으로 갱신된 시간 |

기존 `sessions` 문서에는 이미 있는 필드를 그대로 사용한다.

```text
sessions/{sessionId}
  totalSojuDrinkCount
  totalBeerDrinkCount
```

필요하면 아래 필드만 추가할 수 있다.

```text
sessions/{sessionId}
  lastDrinkAt: Timestamp
  updatedAt: Timestamp
```

기존 `glassMappings` 문서도 이미 있는 `drinkCount`를 그대로 사용한다.

필요하면 아래 필드만 추가할 수 있다.

```text
sessions/{sessionId}/glassMappings/{mappingId}
  lastDrinkAt: Timestamp
  updatedAt: Timestamp
```

## 색상 화면과 실제 술잔이 매핑될 때

현재 `glassMappings`에는 색상만 배정된 대기 상태 문서가 있다.

예시:

```text
sessions/{sessionId}/glassMappings/{mappingId}
  userId: "{userId}"
  drinkType: "soju"
  screenColorHex: "#14b8a6"
  glassId: "glass_pending_color_14b8a6"
  mappingSource: "colorPending"
  drinkCount: 0
```

YOLO가 색상 화면과 술잔이 5초 이상 가까운 것을 확인하면 이 문서를 수정한다.

변경 후:

```text
sessions/{sessionId}/glassMappings/{mappingId}
  userId: "{userId}"
  drinkType: "soju"
  screenColorHex: "#14b8a6"
  glassId: "camera_table_1_soju_glass_1"
  physicalGlassId: "camera_table_1_soju_glass_1"
  mappingSource: "colorProximity"
  drinkCount: 0
  mappedAt: Timestamp
  updatedAt: Timestamp
```

같은 사용자의 participant 문서도 같이 수정한다.

```text
sessions/{sessionId}/participants/{userId}
  glassMappingType: "colorProximity"
  physicalGlassId: "camera_table_1_soju_glass_1"
  mappedScreenColorHex: "#14b8a6"
  glassMappedAt: Timestamp
  updatedAt: Timestamp
```

## 소주 한 잔이 카운트될 때

조건:

```text
매핑된 소주잔 + 소주병이 3초 이상 가까이 있음
```

이때 수정되는 DB는 3곳이다.

### 1. sessions 전체 소주 카운트 증가

```text
sessions/{sessionId}
  totalSojuDrinkCount += 1
  lastDrinkAt = Timestamp
  updatedAt = Timestamp
```

### 2. participants 사용자 소주 카운트 증가

```text
sessions/{sessionId}/participants/{userId}
  sojuDrinkCount += 1
  lastDrinkType = "soju"
  lastDrinkAt = Timestamp
  updatedAt = Timestamp
```

### 3. glassMappings 해당 술잔 카운트 증가

```text
sessions/{sessionId}/glassMappings/{mappingId}
  drinkCount += 1
  lastDrinkAt = Timestamp
  updatedAt = Timestamp
```

## 맥주 한 잔이 카운트될 때

조건:

```text
매핑된 맥주잔 + 맥주병이 3초 이상 가까이 있음
```

이때 수정되는 DB도 3곳이다.

### 1. sessions 전체 맥주 카운트 증가

```text
sessions/{sessionId}
  totalBeerDrinkCount += 1
  lastDrinkAt = Timestamp
  updatedAt = Timestamp
```

### 2. participants 사용자 맥주 카운트 증가

```text
sessions/{sessionId}/participants/{userId}
  beerDrinkCount += 1
  lastDrinkType = "beer"
  lastDrinkAt = Timestamp
  updatedAt = Timestamp
```

### 3. glassMappings 해당 술잔 카운트 증가

```text
sessions/{sessionId}/glassMappings/{mappingId}
  drinkCount += 1
  lastDrinkAt = Timestamp
  updatedAt = Timestamp
```

## YOLO 입력용 임시 컬렉션

카운트 저장 구조는 기존 DB를 활용하지만, YOLO 결과를 Cloud Functions에 전달하려면 입력용 문서는 필요하다.

```text
sessions/{sessionId}/cvFrames/{frameId}
```

이 문서는 YOLO가 매 프레임 또는 일정 간격으로 올리는 탐지 결과다.

예시:

```text
sessions/{sessionId}/cvFrames/{frameId}
  cameraId: "camera_table_1"
  frameWidth: 1280
  frameHeight: 720
  capturedAt: Timestamp
  detections: [
    {
      trackId: "screen_1",
      objectType: "phone_screen",
      centerX: 310,
      centerY: 330,
      confidence: 0.96,
      screenColorHex: "#14b8a6"
    },
    {
      trackId: "soju_glass_1",
      objectType: "soju_glass",
      centerX: 355,
      centerY: 338,
      confidence: 0.91
    },
    {
      trackId: "soju_bottle_1",
      objectType: "green_soju_bottle",
      centerX: 370,
      centerY: 345,
      confidence: 0.90
    }
  ]
```

`cvFrames`는 카운트 결과 저장소가 아니라 YOLO 입력용이다.

## 3초와 5초를 기억하기 위한 상태 문서

5초 매핑, 3초 카운트를 하려면 "언제부터 가까웠는지"를 기억해야 한다.

이를 위해 아래 상태 문서를 사용할 수 있다.

```text
sessions/{sessionId}/cvPairStates/{pairId}
```

색상 화면과 술잔 상태 예시:

```text
pairType: "colorGlass"
screenTrackId: "screen_1"
glassTrackId: "soju_glass_1"
screenColorHex: "#14b8a6"
physicalGlassId: "camera_table_1_soju_glass_1"
drinkType: "soju"
firstSeenAt: Timestamp
lastSeenAt: Timestamp
durationMs: 5200
thresholdMs: 5000
mappedAt: Timestamp
isActive: true
```

술잔과 술병 상태 예시:

```text
pairType: "pour"
glassTrackId: "soju_glass_1"
bottleTrackId: "soju_bottle_1"
physicalGlassId: "camera_table_1_soju_glass_1"
drinkType: "soju"
firstSeenAt: Timestamp
lastSeenAt: Timestamp
durationMs: 3100
thresholdMs: 3000
countedAt: Timestamp
isActive: true
```

`cvPairStates`도 최종 통계 DB가 아니라 Cloud Functions가 시간 누적을 계산하기 위한 내부 상태 문서다.

## 만들지 않을 DB

이번 최소 변경안에서는 아래 컬렉션을 만들지 않는다.

```text
sessions/{sessionId}/drinkCounters
sessions/{sessionId}/drinkRecords
```

이유:

- 사용자별 카운트는 `participants`에 저장한다.
- 세션 전체 카운트는 `sessions`에 저장한다.
- 술잔별 카운트는 `glassMappings`에 저장한다.
- 따라서 당장 카운트 표시와 세션 요약에는 `drinkCounters`, `drinkRecords`가 없어도 된다.

## 삭제할 DB

이번 설계에서 기존 DB를 삭제하지 않는다.

삭제하지 않는 이유:

- 기존 `sessions`는 주문, 참여자, 테이블 상태와 연결되어 있다.
- 기존 `participants`는 사용자 세션 참여 정보다.
- 기존 `glassMappings`는 색상 화면 배정 정보를 이미 갖고 있다.
- 기존 `orders`, `users`, `stores`, `tables`도 그대로 사용한다.

다만 운영 중 너무 많이 쌓일 수 있는 아래 문서는 나중에 정리 대상이 될 수 있다.

```text
sessions/{sessionId}/cvFrames/{oldFrameId}
sessions/{sessionId}/cvPairStates/{oldPairId}
```

이 둘은 디버깅과 상태 추적용이라 세션 종료 후 일정 시간이 지나면 삭제해도 된다.

## 최종 구조 요약

최소 변경 후 핵심 구조는 다음과 같다.

```text
sessions/{sessionId}
  totalSojuDrinkCount
  totalBeerDrinkCount
  lastDrinkAt
  updatedAt

sessions/{sessionId}/participants/{userId}
  userId
  userName
  glassColor
  glassMappingType
  sojuDrinkCount
  beerDrinkCount
  lastDrinkType
  lastDrinkAt
  updatedAt

sessions/{sessionId}/glassMappings/{mappingId}
  userId
  drinkType
  screenColorHex
  glassId
  physicalGlassId
  mappingSource
  drinkCount
  mappedAt
  lastDrinkAt
  updatedAt

sessions/{sessionId}/cvFrames/{frameId}
  YOLO 탐지 결과 입력

sessions/{sessionId}/cvPairStates/{pairId}
  5초 매핑, 3초 카운트 시간 누적 상태
```

## 한 줄 요약

기존 DB를 최대한 활용하면, 카운트 결과는 `participants`, `sessions`, `glassMappings` 세 곳에만 저장하고, `cvFrames`와 `cvPairStates`는 YOLO 처리용 임시 입력/상태 문서로만 사용하면 된다.

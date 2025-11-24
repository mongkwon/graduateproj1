# WYE 프로젝트 설계 문서 (Architecture Design Document)

## 목차
1. [프로젝트 개요](#1-프로젝트-개요)
2. [전체 시스템 아키텍처](#2-전체-시스템-아키텍처)
3. [모듈 구성 및 상호작용](#3-모듈-구성-및-상호작용)
4. [클래스 다이어그램](#4-클래스-다이어그램)
5. [데이터 플로우 다이어그램](#5-데이터-플로우-다이어그램)
6. [E-R 다이어그램](#6-e-r-다이어그램)
7. [핵심 알고리즘](#7-핵심-알고리즘)
8. [API 연동](#8-api-연동)

---

## 1. 프로젝트 개요

### 1.1 서비스 명
**WYE** (Where You meet Everyone)

### 1.2 목적
여러 사람이 만날 때 모든 참여자의 위치를 고려하여 최적의 약속 장소를 추천하는 웹 애플리케이션

### 1.3 핵심 기능
- 카카오 소셜 로그인
- 친구 관리 (카카오 친구 불러오기, 수동 추가)
- 약속 생성 및 참여자 관리
- **Haversine 공식 기반** 거리 계산 (직선거리)
- 두 가지 추천 알고리즘:
  - **최적거리**: Geometric Median (거리 합 최소화)
  - **공평거리**: Smallest Enclosing Circle (가장 먼 사람의 거리 최소화)
- 카카오맵 연동 시각화

### 1.4 기술 스택
- **Frontend**: React + TypeScript, Tailwind CSS
- **Backend**: Supabase (PostgreSQL + Edge Functions)
- **인증**: Kakao OAuth 2.0
- **지도/장소 검색**: Kakao Maps API, Kakao Local API
- **거리 계산**: Haversine 공식 (직선거리)

---

## 2. 전체 시스템 아키텍처

```mermaid
graph TB
    subgraph Client["Client Layer (React)"]
        WYE[WYEApp.tsx<br/>Main App]
        Friends[FriendsListPage<br/>Page]
        Places[FairDistance<br/>Places]
        
        subgraph Components["Component Layer"]
            UI[KakaoMap, KakaoAddressSearch<br/>Tabs, Cards, Buttons, Dialogs]
        end
        
        WYE --> UI
        Friends --> UI
        Places --> UI
    end
    
    subgraph Utils["Utility Layer (Utils)"]
        Storage[meetupStorage<br/>약속 관리]
        Profile[userProfile<br/>프로필 관리]
        FriendsMgr[friendsManager<br/>친구 관리]
        
        Algo[fairDistanceCalculator<br/>거리 알고리즘]
        Token[kakaoTokenManager<br/>토큰 관리]
        
        Supabase[Supabase Client]
        
        Storage --> Supabase
        Profile --> Supabase
        FriendsMgr --> Supabase
    end
    
    subgraph Backend["Backend Layer (Supabase)"]
        subgraph DB["PostgreSQL Database"]
            UserProfiles[(user_profiles)]
            FriendsDB[(friends)]
            Meetups[(meetups)]
            Participants[(meetup_participants)]
        end
        
        subgraph EdgeFn["Edge Functions (Deno)"]
            KakaoAuth[kakao-auth<br/>카카오 OAuth]
            KakaoFriends[kakao-friends<br/>카카오 친구 조회]
            Server[server<br/>KV Store]
        end
    end
    
    subgraph External["External APIs"]
        OAuth[Kakao OAuth API<br/>인증]
        Maps[Kakao Maps API<br/>지도/검색]
        Local[Kakao Local API<br/>장소 검색]
    end
    
    Client -->|API Calls| Utils
    Utils -->|Database Queries| Backend
    Backend -->|External API Calls| External
    
    style Client fill:#e1f5ff
    style Utils fill:#fff4e1
    style Backend fill:#f0e1ff
    style External fill:#e1ffe1
```

---

## 3. 모듈 구성 및 상호작용

### 3.1 Presentation Layer (Components)

#### 3.1.1 WYEApp.tsx
**역할**: 메인 애플리케이션 컨테이너, 전체 상태 관리
**주요 기능**:
- 카카오 로그인/로그아웃
- 사용자 프로필 관리
- 약속 생성 및 목록 표시
- 참여자 관리
- 장소 추천 트리거

**상호작용**:
```
WYEApp ──> userProfile.tsx (프로필 조회/저장)
       ──> meetupStorage.tsx (약속 저장/조회)
       ──> friendsManager.tsx (친구 목록)
       ──> kakaoTokenManager.tsx (토큰 관리)
       ──> fairDistanceCalculator.tsx (알고리즘 실행)
```

#### 3.1.2 FriendsListPage.tsx
**역할**: 친구 관리 UI
**주요 기능**:
- 친구 목록 표시
- 카카오 친구 가져오기
- 수동 친구 추가/삭제

**상호작용**:
```
FriendsListPage ──> friendsManager.tsx (CRUD)
                ──> kakao-friends Edge Function (카카오 API)
```

#### 3.1.3 FairDistancePlaces.tsx
**역할**: 추천 장소 목록 표시
**주요 기능**:
- 최적거리/공평거리 장소 목록
- 장소 선택
- 상세 정보 표시

#### 3.1.4 KakaoMap.tsx
**역할**: 지도 시각화
**주요 기능**:
- 참여자 위치 마커 표시
- 추천 장소 마커 표시
- 마커 클릭 이벤트

#### 3.1.5 KakaoAddressSearch.tsx
**역할**: 주소 검색 인터페이스
**주요 기능**:
- 카카오 장소 검색 API 호출
- 검색 결과 표시
- 좌표 변환

---

### 3.2 Business Logic Layer (Utils)

#### 3.2.1 fairDistanceCalculator.tsx
**역할**: 거리 계산 및 최적화 알고리즘
**핵심 함수**:

```typescript
// Geometric Median 계산 (최적거리)
calculateGeometricMedian(participants: ParticipantLocation[]): {lat, lng}

// Smallest Enclosing Circle 계산 (공평거리)
calculateSmallestEnclosingCircle(participants: ParticipantLocation[]): {lat, lng, radius}

// Haversine 직선거리 계산
calculateDistance(lat1, lng1, lat2, lng2): number

// 카카오 주소/장소 검색
getCoordinatesFromAddress(address: string): Promise<{lat, lng}>
searchNearbyPlaces(center: {lat, lng}, keyword: string): Promise<Location[]>

// 정렬 알고리즘
sortByOptimalDistance(places, participants): Location[]
sortByFairDistance(places, participants): Location[]
```

**알고리즘 상세**:
- **Weiszfeld 알고리즘**: 반복적으로 기하학적 중심에 수렴
- **Welzl 알고리즘**: 최소 원 계산 (재귀적)

#### 3.2.2 meetupStorage.tsx
**역할**: 약속 데이터 영속성 관리
**핵심 함수**:
```typescript
saveMeetup(meetupData: MeetupData): Promise<void>
getMeetupsByUser(kakaoId: string): Promise<MeetupData[]>
deleteMeetup(meetupId: string): Promise<void>
```

**데이터 흐름**:
```
WYEApp → saveMeetup() → Supabase.from('meetups').insert()
                      → Supabase.from('meetup_participants').insert()
```

#### 3.2.3 userProfile.tsx
**역할**: 사용자 프로필 관리
**핵심 함수**:
```typescript
getUserProfile(kakaoId: string): Promise<UserProfile | null>
upsertUserProfile(profile: UserProfile): Promise<boolean>
updateUserSettings(kakaoId, settings): Promise<boolean>
```

#### 3.2.4 friendsManager.tsx
**역할**: 친구 관계 관리
**핵심 함수**:
```typescript
getFriends(userKakaoId: string): Promise<Friend[]>
addFriend(userKakaoId, friendName, friendPhone, friendKakaoId): Promise<Friend>
deleteFriend(friendId: string): Promise<void>
getFriendProfile(friendKakaoId: string): Promise<UserProfile | null>
```

#### 3.2.5 kakaoTokenManager.tsx
**역할**: 카카오 OAuth 토큰 생명주기 관리
**핵심 함수**:
```typescript
saveTokenData(data: {accessToken, refreshToken, expiresIn}): void
getTokenData(): KakaoTokenData | null
isAccessTokenValid(): boolean
isRefreshTokenValid(): boolean
getAccessToken(): string | null
refreshAccessToken(): Promise<string | null>
```

**토큰 갱신 로직**:
```
1. getAccessToken() 호출
2. isAccessTokenValid() 체크
3. 만료 임박 → refreshAccessToken() 호출
4. Refresh Token으로 새 Access Token 발급
5. saveTokenData()로 저장
```

---

### 3.3 Data Access Layer (Supabase)

#### 3.3.1 Database Schema

**user_profiles**
```sql
- id (PK, UUID)
- kakao_id (UNIQUE NOT NULL, TEXT)
- name (TEXT NOT NULL)
- email (TEXT)
- profile_image (TEXT)
- default_location (TEXT)
- default_transport (TEXT)
- share_default_location (BOOLEAN DEFAULT true)
- share_default_transport (BOOLEAN DEFAULT true)
- created_at (TIMESTAMP)
- updated_at (TIMESTAMP)
```

**friends**
```sql
- id (PK, UUID)
- user_kakao_id (TEXT)
- friend_kakao_id (TEXT, nullable)
- friend_name (TEXT)
- friend_phone (TEXT, nullable)
- status (TEXT: pending/accepted/blocked)
- created_at (TIMESTAMP)
```

**meetups**
```sql
- id (PK, TEXT)
- title (TEXT)
- date (TEXT)
- time (TEXT)
- purpose (TEXT)
- status (TEXT: planning/confirmed/active)
- time_optimization_type (TEXT: optimal/fair)
- selected_location (TEXT, JSON)
- notification_settings (TEXT, JSON)
- created_by (TEXT, kakao_id)
- created_at (TIMESTAMP)
- updated_at (TIMESTAMP)
```

**meetup_participants**
```sql
- id (PK, UUID)
- meetup_id (FK → meetups.id)
- participant_id (TEXT, kakao_id)
- participant_name (TEXT)
- location (TEXT)
- transport (TEXT)
- created_at (TIMESTAMP)
```

#### 3.3.2 Edge Functions

**kakao-auth**
```typescript
// Authorization Code → Access Token 교환
POST /kakao-auth
Body: { code, redirectUri }
Response: { accessToken, refreshToken, expiresIn, userId }

// Refresh Token → 새 Access Token
POST /kakao-auth
Body: { grantType: "refresh_token", refreshToken }
Response: { accessToken, refreshToken, expiresIn }
```

**kakao-friends**
```typescript
// 카카오 친구 목록 조회
POST /kakao-friends
Body: { accessToken }
Response: { friends: Friend[] }
```

---

## 4. 클래스 다이어그램

```mermaid
classDiagram
    %% Core Types
    class UserProfile {
        +string kakao_id
        +string name
        +string email
        +string? profile_image
        +string? default_location
        +string? default_transport
        +boolean share_default_location
        +boolean share_default_transport
        +Date created_at
        +Date updated_at
    }

    class Friend {
        +string id
        +string user_kakao_id
        +string? friend_kakao_id
        +string friend_name
        +string? friend_phone
        +string status
        +Date created_at
    }

    class MeetupData {
        +string id
        +string title
        +string date
        +string time
        +string purpose
        +MeetupParticipant[] participants
        +MeetupLocation? selectedLocation
        +string status
        +string? timeOptimizationType
        +NotificationSettings? notificationSettings
        +string createdBy
    }

    class MeetupParticipant {
        +string id
        +string name
        +string location
        +string transport
    }

    class MeetupLocation {
        +string name
        +string address
        +number rating
        +string distance
        +string type
        +string? phone
        +string? hours
        +string? description
        +string[] features
    }

    class Location {
        +string name
        +string address
        +number lat
        +number lng
        +string? distance
        +number? rating
        +string? type
        +number? distanceVariance
        +number? avgDistance
        +number? maxDistance
        +number? totalDistance
    }

    class ParticipantLocation {
        +string name
        +string address
        +number lat
        +number lng
        +string? transport
    }

    %% Utility Classes
    class FairDistanceCalculator {
        <<utility>>
        +calculateGeometricMedian(participants) Coordinates
        +calculateSmallestEnclosingCircle(participants) Circle
        +calculateDistance(lat1, lng1, lat2, lng2) number
        +getCoordinatesFromAddress(address) Coordinates
        +searchNearbyPlaces(center, keyword) Location[]
        +sortByOptimalDistance(places, participants) Location[]
        +sortByFairDistance(places, participants) Location[]
    }

    class UserProfileManager {
        <<utility>>
        +getUserProfile(kakaoId) UserProfile
        +upsertUserProfile(profile) boolean
        +updateUserSettings(kakaoId, settings) boolean
    }

    class FriendsManager {
        <<utility>>
        +getFriends(userKakaoId) Friend[]
        +addFriend(userKakaoId, friendName, ...) Friend
        +deleteFriend(friendId) boolean
        +getFriendProfile(friendKakaoId) UserProfile
    }

    class MeetupStorage {
        <<utility>>
        +saveMeetup(meetupData) void
        +getMeetupsByUser(kakaoId) MeetupData[]
        +deleteMeetup(meetupId) void
    }

    class KakaoTokenManager {
        <<utility>>
        +saveTokenData(data) void
        +getTokenData() KakaoTokenData
        +isAccessTokenValid() boolean
        +refreshAccessToken() string
    }

    %% React Components
    class WYEApp {
        <<component>>
        -state: AppState
        -kakaoUser: KakaoUser
        -meetups: MeetupData[]
        -friends: Friend[]
        +handleLogin() void
        +handleLogout() void
        +createMeetup() void
        +searchFairDistancePlaces() void
    }

    class FriendsListPage {
        <<component>>
        -friends: Friend[]
        +loadFriends() void
        +addFriend() void
        +deleteFriend() void
        +syncKakaoFriends() void
    }

    class FairDistancePlaces {
        <<component>>
        -places: Location[]
        -selectedLocation: Location
        +onSelectLocation() void
    }

    class KakaoMap {
        <<component>>
        -mapInstance: any
        -markers: Marker[]
        +renderParticipants() void
        +renderRecommendedPlaces() void
    }

    %% Relationships
    MeetupData "1" *-- "many" MeetupParticipant
    MeetupData "1" o-- "0..1" MeetupLocation
    
    WYEApp --> UserProfileManager: uses
    WYEApp --> FriendsManager: uses
    WYEApp --> MeetupStorage: uses
    WYEApp --> FairDistanceCalculator: uses
    WYEApp --> KakaoTokenManager: uses
    
    FriendsListPage --> FriendsManager: uses
    FairDistancePlaces --> Location: displays
    KakaoMap --> Location: renders
    
    FairDistanceCalculator --> ParticipantLocation: processes
    FairDistanceCalculator --> Location: returns
    
    MeetupStorage --> MeetupData: persists
    UserProfileManager --> UserProfile: manages
    FriendsManager --> Friend: manages
```

---

## 5. 데이터 플로우 다이어그램

### 5.1 로그인 플로우

```mermaid
sequenceDiagram
    actor User
    participant WYE as WYEApp.tsx
    participant EdgeFn as kakao-auth<br/>Edge Function
    participant Kakao as Kakao Server
    participant DB as Supabase DB

    User->>Kakao: 1. 카카오 로그인 버튼 클릭
    Kakao-->>User: 2. Authorization Code 발급
    User->>EdgeFn: 3. Code 전달
    EdgeFn->>Kakao: 4. Token 요청
    Kakao-->>EdgeFn: 5. Access Token + Refresh Token
    EdgeFn-->>WYE: 6. Token 응답
    WYE->>WYE: 7. Token 저장 (localStorage)
    WYE->>Kakao: 8. 사용자 정보 조회
    Kakao-->>WYE: 9. 사용자 정보 (kakao_id, name, email)
    WYE->>DB: 10. 프로필 저장/업데이트
    DB-->>WYE: 11. 저장 완료
    WYE-->>User: 12. 로그인 완료
```

### 5.2 약속 생성 및 장소 추천 플로우

```mermaid
sequenceDiagram
    actor User
    participant WYE as WYEApp.tsx
    participant Calc as fairDistanceCalculator
    participant Kakao as Kakao Maps API
    participant DB as Supabase DB

    User->>WYE: 1. 약속 정보 입력<br/>(제목, 날짜, 목적)
    User->>WYE: 2. 참여자 추가<br/>(이름, 위치, 교통수단)
    User->>WYE: 3. "최적거리" 또는 "공평거리" 클릭
    
    WYE->>Kakao: 4. 각 참여자 주소 → 좌표 변환
    Kakao-->>WYE: 5. 좌표 응답
    
    WYE->>Calc: 6. 알고리즘 실행 (참여자 좌표 전달)
    
    Note over Calc: 7. Geometric Median or<br/>Smallest Enclosing Circle 계산<br/>(Haversine 공식 사용)
    
    Calc-->>WYE: 8. 중심 좌표 응답
    
    WYE->>Kakao: 9. 중심 좌표 기준 주변 장소 검색
    Kakao-->>WYE: 10. 장소 목록 응답
    
    WYE->>Calc: 11. 각 장소와 참여자 간 거리 계산
    
    Note over Calc: 12. Haversine 공식으로<br/>직선거리 계산
    
    Calc->>Calc: 13. 정렬 알고리즘 실행<br/>(최적거리: totalDistance 오름차순)<br/>(공평거리: maxDistance 오름차순)
    
    Calc-->>WYE: 14. 정렬된 장소 목록
    WYE-->>User: 15. 추천 장소 표시
    
    User->>WYE: 16. 장소 선택
    WYE->>DB: 17. 약속 저장 (meetups, meetup_participants)
    DB-->>WYE: 18. 저장 완료
    WYE-->>User: 19. 약속 생성 완료
```

### 5.3 친구 관리 플로우

```mermaid
sequenceDiagram
    actor User
    participant Friends as FriendsListPage
    participant Mgr as friendsManager
    participant DB as Supabase DB
    participant EdgeFn as kakao-friends<br/>Edge Function
    participant Kakao as Kakao API

    User->>Friends: 1. 친구 목록 페이지 이동
    Friends->>Mgr: 2. getFriends() 호출
    Mgr->>DB: 3. SELECT query
    DB-->>Mgr: 4. 친구 목록 응답
    Mgr-->>Friends: 5. 친구 목록 반환
    Friends-->>User: 6. 친구 목록 표시
    
    alt 수동 친구 추가
        User->>Friends: 7. 친구 추가 (이름, 전화번호)
        Friends->>Mgr: 8. addFriend() 호출
        Mgr->>DB: 9. INSERT query
        DB-->>Mgr: 10. 성공 응답
        Mgr-->>Friends: 11. 추가 완료
        Friends-->>User: 12. 목록 갱신
    end
    
    alt 카카오 친구 가져오기
        User->>Friends: 13. 카카오 친구 가져오기
        Friends->>EdgeFn: 14. 카카오 친구 조회 요청
        EdgeFn->>Kakao: 15. 친구 목록 API 호출
        Kakao-->>EdgeFn: 16. 친구 목록 응답
        EdgeFn-->>Friends: 17. 친구 데이터 반환
        Friends->>Mgr: 18. 친구 일괄 추가
        Mgr->>DB: 19. BATCH INSERT
        DB-->>Mgr: 20. 성공 응답
        Mgr-->>Friends: 21. 추가 완료
        Friends-->>User: 22. 목록 갱신
    end
```

---

## 6. E-R 다이어그램

```mermaid
erDiagram
    user_profiles ||--o{ meetups : creates
    user_profiles ||--o{ friends : has
    meetups ||--o{ meetup_participants : contains
    
    user_profiles {
        uuid id PK
        varchar kakao_id UK "NOT NULL"
        varchar name "NOT NULL"
        varchar email
        text profile_image
        text default_location
        varchar default_transport
        boolean share_default_location "DEFAULT true"
        boolean share_default_transport "DEFAULT true"
        timestamp created_at
        timestamp updated_at
    }
    
    meetups {
        text id PK
        text title
        text date
        text time
        text purpose
        text status "planning/confirmed/active"
        text time_optimization_type "optimal/fair"
        text selected_location "JSON"
        text notification_settings "JSON"
        text created_by FK
        timestamp created_at
        timestamp updated_at
    }
    
    meetup_participants {
        uuid id PK
        text meetup_id FK
        text participant_id
        text participant_name
        text location
        text transport
        timestamp created_at
    }
    
    friends {
        uuid id PK
        varchar user_kakao_id FK
        varchar friend_kakao_id "nullable"
        varchar friend_name
        varchar friend_phone "nullable"
        varchar status "pending/accepted/blocked"
        timestamp created_at
        timestamp updated_at
    }
```

**관계 설명**:

1. **user_profiles ─(1:N)─> meetups**
   - 한 사용자는 여러 약속을 생성할 수 있음
   - `meetups.created_by` → `user_profiles.kakao_id`

2. **meetups ─(1:N)─> meetup_participants**
   - 한 약속에는 여러 참여자가 포함됨
   - `meetup_participants.meetup_id` → `meetups.id`
   - CASCADE DELETE: 약속 삭제 시 참여자도 삭제

3. **user_profiles ─(1:N)─> friends**
   - 한 사용자는 여러 친구를 가질 수 있음
   - `friends.user_kakao_id` → `user_profiles.kakao_id`

4. **friends.friend_kakao_id → user_profiles.kakao_id** (논리적 관계)
   - 친구가 WYE 사용자인 경우 프로필 참조
   - NULL 가능 (비사용자 친구)

---

## 7. 핵심 알고리즘

### 7.1 최적거리 알고리즘 (Geometric Median)

**목적**: 모든 참여자까지의 거리 합을 최소화하는 지점 찾기

**알고리즘**: Weiszfeld Algorithm
```
Input: participants[] with {lat, lng}
Output: {lat, lng} of optimal point

1. Initialize: 
   current = geometric center of all participants

2. Iterate (max 100 times):
   numeratorLat = 0, numeratorLng = 0, denominator = 0
   
   For each participant p:
     distance = haversineDistance(current, p)
     weight = 1 / distance
     numeratorLat += p.lat * weight
     numeratorLng += p.lng * weight
     denominator += weight
   
   newLat = numeratorLat / denominator
   newLng = numeratorLng / denominator
   
   If change < tolerance:
     CONVERGED → return {newLat, newLng}
   
   current = {newLat, newLng}

3. Return current point
```

**장소 정렬 (최적거리/공평거리 공통)**:
```
For each place:
  totalDistance = 0
  distances = []
  
  For each participant:
    distance = calculateDistance(participant.lat, participant.lng, place.lat, place.lng)
    distances.push(distance)
    totalDistance += distance
  
  place.totalDistance = totalDistance
  place.avgDistance = totalDistance / participants.length
  place.maxDistance = max(distances)
  place.minDistance = min(distances)
  place.distanceVariance = stdDeviation(distances)

최적거리 정렬:
  Sort by totalDistance (ascending)

공평거리 정렬:
  Sort by maxDistance (ascending)
  If tied → sort by distanceVariance (ascending)
```

---

### 7.2 공평거리 알고리즘 (Smallest Enclosing Circle)

**목적**: 가장 먼 참여자의 거리를 최소화하는 지점 찾기

**알고리즘**: Welzl's Algorithm (재귀적 최소 원 계산)
```
Input: participants[]
Output: {lat, lng, radius} of smallest circle

Function welzl(P, R, n):
  If n == 0 or |R| == 3:
    return trivialCircle(R)
  
  p = P[n-1]
  circle = welzl(P, R, n-1)
  
  If p is inside circle:
    return circle
  
  Return welzl(P, R ∪ {p}, n-1)

Function trivialCircle(points):
  If |points| == 0:
    return {center: {0,0}, radius: 0}
  If |points| == 1:
    return {center: points[0], radius: 0}
  If |points| == 2:
    return circle from diameter
  If |points| == 3:
    return circumcircle of triangle
```

**장소 정렬**: 7.3의 공통 정렬 알고리즘 참조

---

### 7.3 Haversine 공식

**목적**: 두 지점 간의 직선 거리 계산 (구면 거리)

**공식**:
```
R = 6371 (지구 반지름, km)

dLat = (lat2 - lat1) × π / 180
dLng = (lng2 - lng1) × π / 180

a = sin²(dLat/2) + cos(lat1) × cos(lat2) × sin²(dLng/2)
c = 2 × atan2(√a, √(1-a))

distance = R × c
```

**특징**:
- 지구를 완전한 구로 가정
- 오차: 0.5% 이내 (실제 지구는 타원체)
- 계산 속도: 매우 빠름 (삼각함수만 사용)
- 실제 이동 거리보다 짧음 (직선거리)

**장소 정렬 (최적거리/공평거리 공통)**:
```
For each place:
  totalDistance = 0
  distances = []
  
  For each participant:
    distance = calculateDistance(participant.lat, participant.lng, place.lat, place.lng)
    distances.push(distance)
    totalDistance += distance
  
  place.totalDistance = totalDistance
  place.avgDistance = totalDistance / participants.length
  place.maxDistance = max(distances)
  place.minDistance = min(distances)
  place.distanceVariance = stdDeviation(distances)

최적거리 정렬:
  Sort by totalDistance (ascending)

공평거리 정렬:
  Sort by maxDistance (ascending)
  If tied → sort by distanceVariance (ascending)
```

---

## 8. API 연동

### 8.1 Kakao OAuth API

**인증 플로우**:
```
1. Authorization Request
   GET https://kauth.kakao.com/oauth/authorize
   ?client_id={REST_API_KEY}
   &redirect_uri={REDIRECT_URI}
   &response_type=code

2. Token Request (via Edge Function)
   POST https://kauth.kakao.com/oauth/token
   Body: {
     grant_type: "authorization_code",
     client_id: {REST_API_KEY},
     client_secret: {SECRET},
     redirect_uri: {REDIRECT_URI},
     code: {AUTH_CODE}
   }
   Response: {
     access_token,
     refresh_token,
     expires_in,
     refresh_token_expires_in
   }

3. Refresh Token
   POST https://kauth.kakao.com/oauth/token
   Body: {
     grant_type: "refresh_token",
     client_id: {REST_API_KEY},
     refresh_token: {REFRESH_TOKEN}
   }
```

### 8.2 Kakao Maps & Local API

**장소 검색** (keywordSearch):
```javascript
kakao.maps.services.Places.keywordSearch(
  keyword,
  callback,
  {
    location: new kakao.maps.LatLng(lat, lng),
    radius: 3000,  // 검색 반경 (미터)
    size: 15       // 결과 개수
  }
)
```

**주소 → 좌표 변환** (Geocoder):
```javascript
kakao.maps.services.Geocoder.addressSearch(
  address,
  (result, status) => {
    if (status === kakao.maps.services.Status.OK) {
      return { lat: result[0].y, lng: result[0].x }
    }
  }
)
```

---

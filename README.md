# WYE (Where You @) - 시퀀스 다이어그램

## 1. 카카오 로그인 흐름

```mermaid
sequenceDiagram
    actor User
    participant WYEApp
    participant KakaoSDK
    participant SupabaseEdge as Supabase Edge Function
    participant KakaoAPI as Kakao API
    participant TokenManager as kakaoTokenManager
    participant UserProfile as userProfile
    participant LocalStorage

    User->>WYEApp: 앱 접속
    WYEApp->>LocalStorage: getItem('kakao_token_data')
    LocalStorage-->>WYEApp: tokenData
    
    alt 유효한 토큰 존재
        WYEApp->>TokenManager: isAccessTokenValid()
        TokenManager->>TokenManager: 만료시간 체크 (5분 여유)
        TokenManager-->>WYEApp: true
        WYEApp->>TokenManager: getAccessToken()
        TokenManager-->>WYEApp: accessToken
        WYEApp->>SupabaseEdge: getUserInfo(accessToken)
        SupabaseEdge->>KakaoAPI: GET /v2/user/me
        KakaoAPI-->>SupabaseEdge: userInfo
        SupabaseEdge-->>WYEApp: { id, nickname, profileImage }
        WYEApp->>WYEApp: setCurrentUser(userInfo)
        WYEApp->>WYEApp: setCurrentScreen("home")
    else 토큰 만료
        User->>WYEApp: 로그인 버튼 클릭
        WYEApp->>KakaoSDK: Kakao.Auth.authorize()
        KakaoSDK-->>User: 카카오 로그인 페이지 리디렉션
        User->>KakaoAPI: 로그인 및 동의
        KakaoAPI-->>WYEApp: 리디렉션 (code)
        WYEApp->>WYEApp: URLSearchParams.get('code')
        WYEApp->>SupabaseEdge: POST /kakao-auth { code }
        SupabaseEdge->>KakaoAPI: POST /oauth/token
        KakaoAPI-->>SupabaseEdge: { access_token, refresh_token, expires_in }
        SupabaseEdge->>KakaoAPI: GET /v2/user/me
        KakaoAPI-->>SupabaseEdge: userInfo
        SupabaseEdge-->>WYEApp: { success, accessToken, refreshToken, userInfo }
        WYEApp->>TokenManager: saveTokenData({ accessToken, refreshToken, expiresIn, refreshTokenExpiresIn })
        TokenManager->>LocalStorage: setItem('kakao_token_data', JSON.stringify(tokenData))
        WYEApp->>WYEApp: setCurrentUser(userInfo)
        WYEApp->>WYEApp: setCurrentScreen("home")
    end
```

## 2. 약속 생성 흐름

```mermaid
sequenceDiagram
    actor User
    participant WYEApp
    participant FriendsManager as friendsManager
    participant MeetupStorage as meetupStorage
    participant NotificationHelper as notificationHelpers
    participant LocalStorage
    participant SupabaseKV as Supabase KV Store

    User->>WYEApp: "새 약속 만들기" 클릭
    WYEApp->>WYEApp: setCurrentScreen("create")
    WYEApp->>WYEApp: 폼 초기화
    
    User->>WYEApp: 약속 정보 입력
    Note over User,WYEApp: title, date, time, purpose
    
    User->>WYEApp: "친구 추가" 클릭
    WYEApp->>FriendsManager: loadFriends(currentUser.id)
    FriendsManager->>SupabaseKV: kv.get(`friends_${userId}`)
    SupabaseKV-->>FriendsManager: friendsList
    FriendsManager-->>WYEApp: Friend[]
    WYEApp->>WYEApp: setShowFriendsDialog(true)
    
    User->>WYEApp: 친구 선택
    WYEApp->>WYEApp: participants.push({ id, name, location, transport })
    
    User->>WYEApp: 출발지 검색 (KakaoAddressSearch)
    WYEApp->>KakaoMap: 카카오맵 API 호출
    KakaoMap-->>WYEApp: 주소 결과
    User->>WYEApp: 주소 선택
    WYEApp->>WYEApp: participant.location = address
    
    User->>WYEApp: 교통수단 선택
    WYEApp->>WYEApp: participant.transport = "car" | "transit"
    
    User->>WYEApp: "약속 만들기" 버튼 클릭
    WYEApp->>WYEApp: 입력 검증
    
    WYEApp->>WYEApp: 새 Meetup 객체 생성
    Note over WYEApp: { id, title, date, time,<br/>purpose, participants,<br/>status: "planning",<br/>createdBy, createdAt }
    
    WYEApp->>MeetupStorage: saveMeetup(meetup)
    MeetupStorage->>SupabaseKV: kv.set(`meetup_${meetupId}`, meetup)
    SupabaseKV-->>MeetupStorage: success
    MeetupStorage->>SupabaseKV: kv.get(`user_meetups_${userId}`)
    SupabaseKV-->>MeetupStorage: meetupIds[]
    MeetupStorage->>MeetupStorage: meetupIds.push(newMeetupId)
    MeetupStorage->>SupabaseKV: kv.set(`user_meetups_${userId}`, meetupIds)
    
    loop 각 참여자
        WYEApp->>MeetupStorage: addMeetupToUser(participantId, meetupId)
        MeetupStorage->>SupabaseKV: kv.get(`user_meetups_${participantId}`)
        SupabaseKV-->>MeetupStorage: meetupIds[]
        MeetupStorage->>SupabaseKV: kv.set(`user_meetups_${participantId}`, updatedIds)
    end
    
    WYEApp->>NotificationHelper: createMeetupNotification(meetup, participants)
    loop 각 참여자
        NotificationHelper->>SupabaseKV: kv.get(`notifications_${participantId}`)
        SupabaseKV-->>NotificationHelper: notifications[]
        NotificationHelper->>NotificationHelper: notifications.unshift(newNotification)
        NotificationHelper->>SupabaseKV: kv.set(`notifications_${participantId}`, notifications)
    end
    
    WYEApp->>WYEApp: setMeetups([...meetups, newMeetup])
    WYEApp->>WYEApp: setCurrentScreen("home")
    WYEApp->>User: "약속이 생성되었습니다!" 토스트
```

## 3. 공평 거리 장소 추천 흐름

```mermaid
sequenceDiagram
    actor User
    participant WYEApp
    participant FairCalculator as fairDistanceCalculator
    participant KakaoPlaces as Kakao Places API
    participant FairDistancePlaces

    User->>WYEApp: 약속 상세 > "장소 추천" 클릭
    WYEApp->>WYEApp: setShowRecommendations(true)
    
    WYEApp->>FairCalculator: calculateGeometricMedian(participants)
    Note over FairCalculator: Weiszfeld 알고리즘 사용
    
    loop 최대 100회 반복 (수렴까지)
        FairCalculator->>FairCalculator: 현재 추정점에서<br/>각 참여자까지 거리 계산
        FairCalculator->>FairCalculator: calculateDistance(lat1, lng1, lat2, lng2)
        Note over FairCalculator: Haversine 공식:<br/>R = 6371km,<br/>dLat, dLng 계산,<br/>a = sin²(dLat/2) + cos(lat1)×cos(lat2)×sin²(dLng/2),<br/>c = 2×atan2(√a, √(1-a)),<br/>distance = R × c
        FairCalculator->>FairCalculator: 가중 평균으로 새 추정점 계산
        FairCalculator->>FairCalculator: 이동 거리 < 0.001km면 종료
    end
    
    FairCalculator-->>WYEApp: { lat, lng } (중심점)
    
    WYEApp->>KakaoPlaces: 키워드 검색 (중심점, purpose)
    Note over WYEApp,KakaoPlaces: Places API:<br/>new kakao.maps.services.Places()<br/>.keywordSearch(purpose, callback, options)
    
    KakaoPlaces-->>WYEApp: searchResults[]
    
    WYEApp->>FairCalculator: calculateFairDistancePlaces(searchResults, participants)
    
    loop 각 장소
        loop 각 참여자
            FairCalculator->>FairCalculator: distance = calculateDistance(<br/>place.lat, place.lng,<br/>participant.lat, participant.lng)
            FairCalculator->>FairCalculator: distances.push(distance)
        end
        
        FairCalculator->>FairCalculator: avgDistance = mean(distances)
        FairCalculator->>FairCalculator: variance = √(Σ(d - avg)² / n)
        Note over FairCalculator: 거리 편차(표준편차) 계산
        
        FairCalculator->>FairCalculator: place.distanceVariance = variance
        FairCalculator->>FairCalculator: place.avgDistance = avgDistance
        FairCalculator->>FairCalculator: place.maxDistance = max(distances)
        FairCalculator->>FairCalculator: place.minDistance = min(distances)
    end
    
    FairCalculator->>FairCalculator: places.sort((a, b) => <br/>a.distanceVariance - b.distanceVariance)
    Note over FairCalculator: 편차가 작을수록 공평
    
    FairCalculator-->>WYEApp: sortedPlaces[]
    
    WYEApp->>FairDistancePlaces: 렌더링 (places, onSelectLocation)
    FairDistancePlaces->>User: 공평한 장소 목록 표시
    
    User->>FairDistancePlaces: 장소 선택
    FairDistancePlaces->>WYEApp: onSelectLocation(selectedPlace)
    WYEApp->>WYEApp: setSelectedLocation(place)
    
    User->>WYEApp: "장소 확정" 버튼
    WYEApp->>WYEApp: currentMeetup.selectedLocation = selectedLocation
    WYEApp->>WYEApp: currentMeetup.status = "confirmed"
    WYEApp->>MeetupStorage: updateMeetup(meetup)
    MeetupStorage->>SupabaseKV: kv.set(`meetup_${id}`, meetup)
    WYEApp->>User: "장소가 확정되었습니다!" 토스트
```

## 4. 경로 표시 및 이동시간 계산 흐름

```mermaid
sequenceDiagram
    actor User
    participant WYEApp
    participant KakaoMap
    participant KakaoMobility as Kakao Mobility API
    participant ODsayAPI as ODsay API
    participant FairCalculator as fairDistanceCalculator

    User->>WYEApp: 확정된 약속 상세 열기
    WYEApp->>WYEApp: selectedLocation 확인
    
    WYEApp->>FairCalculator: calculateParticipantDetails(participants, selectedLocation)
    
    loop 각 참여자
        alt 실시간 위치 있음
            FairCalculator->>FairCalculator: startLat = realtimeLocation.latitude
            FairCalculator->>FairCalculator: startLng = realtimeLocation.longitude
        else 저장된 출발지만 있음
            Note over FairCalculator: 주소는 좌표 변환 필요<br/>(경로 표시 시)
        end
        
        FairCalculator->>FairCalculator: distance = calculateDistance(<br/>startLat, startLng,<br/>selectedLocation.lat, selectedLocation.lng)
        
        alt transport === "car"
            FairCalculator->>FairCalculator: avgSpeed = 40 km/h
        else transport === "transit"
            FairCalculator->>FairCalculator: avgSpeed = 25 km/h
        end
        
        FairCalculator->>FairCalculator: time = (distance / avgSpeed) × 60
        Note over FairCalculator: 시간(분) = (거리 / 속도) × 60
        
        FairCalculator->>FairCalculator: details.push({ name, distance, time, transport })
    end
    
    FairCalculator-->>WYEApp: participantDetails[]
    
    WYEApp->>KakaoMap: 지도 렌더링 (participantLocations, meetingPlace)
    
    loop 각 참여자 (실시간 위치 있는 경우)
        KakaoMap->>KakaoMap: drawRoute(participant)
        
        alt transport === "car"
            KakaoMap->>KakaoMobility: GET /v1/directions
            Note over KakaoMap,KakaoMobility: Headers: { Authorization: "KakaoAK {REST_API_KEY}" }<br/>Query: origin={lng},{lat}&destination={lng},{lat}&priority=RECOMMEND
            
            KakaoMobility-->>KakaoMap: { routes: [{ sections: [{ roads: [{ vertexes }] }] }] }
            
            KakaoMap->>KakaoMap: path = vertexes를 LatLng로 변환
            Note over KakaoMap: vertexes는 [lng, lat, lng, lat, ...]<br/>형태이므로 2개씩 묶어서 변환
            
            KakaoMap->>KakaoMap: polyline = new kakao.maps.Polyline({<br/>path, strokeWeight: 4,<br/>strokeColor: "#3b82f6",<br/>strokeOpacity: 0.8 })
            
            KakaoMap->>KakaoMap: polyline.setMap(map)
            KakaoMap->>KakaoMap: polylinesRef.current.push(polyline)
            
        else transport === "transit"
            KakaoMap->>ODsayAPI: GET /v1/api/searchPubTransPathT
            Note over KakaoMap,ODsayAPI: Query: SX={lng}&SY={lat}&EX={lng}&EY={lat}&apiKey={API_KEY}
            
            alt API 성공
                ODsayAPI-->>KakaoMap: { result: { path: [{ subPath }] } }
                
                loop 각 subPath
                    alt subPath.trafficType === 1 (지하철)
                        KakaoMap->>KakaoMap: passStopList.lane에서 좌표 추출
                    else subPath.trafficType === 2 (버스)
                        KakaoMap->>KakaoMap: passStopList.lane에서 좌표 추출
                    else subPath.trafficType === 3 (도보)
                        KakaoMap->>KakaoMap: 시작점-끝점 직선
                    end
                    
                    KakaoMap->>KakaoMap: path.push(new kakao.maps.LatLng(y, x))
                end
                
                KakaoMap->>KakaoMap: polyline = new kakao.maps.Polyline({<br/>path, strokeWeight: 4,<br/>strokeColor: "#ef4444",<br/>strokeOpacity: 0.8 })
                
            else API 실패 (인증 에러)
                Note over KakaoMap,ODsayAPI: [ApiKeyAuthFailed] 발생 시
                KakaoMap->>KakaoMap: 직선 경로로 fallback
                KakaoMap->>KakaoMap: path = [start, end]
                KakaoMap->>KakaoMap: polyline 생성 (strokeStyle: "dashed")
            end
            
            KakaoMap->>KakaoMap: polyline.setMap(map)
            KakaoMap->>KakaoMap: polylinesRef.current.push(polyline)
        end
        
        KakaoMap->>KakaoMap: 출발지 커스텀 오버레이 생성
        Note over KakaoMap: 참여자 이름 박스 + 화살표 포인터<br/>색상: car = 파란색, transit = 빨간색
        
        KakaoMap->>KakaoMap: overlay.setMap(map)
        KakaoMap->>KakaoMap: participantMarkersRef.current.push(overlay)
    end
    
    KakaoMap->>KakaoMap: 약속장소 마커 생성
    Note over KakaoMap: "📍 약속 장소" 빨간색 오버레이
    
    KakaoMap->>User: 경로 및 마커 표시된 지도
```

## 5. 실시간 위치 공유 흐름

```mermaid
sequenceDiagram
    actor User
    participant WYEApp
    participant Browser as Navigator.geolocation
    participant SupabaseKV
    participant RealtimeLocationMap
    participant KakaoGeocoder as Kakao Geocoder API

    User->>WYEApp: 약속 상세 페이지 진입
    WYEApp->>WYEApp: isWithinOneHourOfMeetup() 체크
    Note over WYEApp: 약속 시간 1시간 이내인지 확인
    
    alt 1시간 이내
        WYEApp->>User: "내 위치 공유" 버튼 활성화
        
        User->>WYEApp: "공유 시작" 클릭
        WYEApp->>WYEApp: toggleLocationSharing()
        
        WYEApp->>Browser: navigator.geolocation.getCurrentPosition()
        Browser-->>WYEApp: { coords: { latitude, longitude } }
        
        WYEApp->>WYEApp: realtimeLocation = {<br/>userId: currentUser.id,<br/>latitude,<br/>longitude,<br/>timestamp: Date.now() }
        
        WYEApp->>SupabaseKV: kv.set(`realtime_location_${meetupId}_${userId}`, location)
        SupabaseKV-->>WYEApp: success
        
        WYEApp->>WYEApp: setIsLocationSharingEnabled(true)
        WYEApp->>WYEApp: setActiveMeetupId(meetupId)
        
        WYEApp->>Browser: setInterval(updateLocation, 10000)
        Note over WYEApp,Browser: 10초마다 위치 업데이트
        
        WYEApp->>WYEApp: setLocationUpdateInterval(intervalId)
        
        loop 10초마다
            WYEApp->>Browser: navigator.geolocation.getCurrentPosition()
            Browser-->>WYEApp: { coords: { latitude, longitude } }
            
            WYEApp->>WYEApp: 위치 변화 확인 (50m 이상 이동 시에만 업데이트)
            
            alt 위치 변화 있음
                WYEApp->>SupabaseKV: kv.set(`realtime_location_${meetupId}_${userId}`, newLocation)
                WYEApp->>KakaoGeocoder: coord2Address(lng, lat)
                KakaoGeocoder-->>WYEApp: address
                WYEApp->>WYEApp: setLocationAddresses({ [userId]: address })
            end
        end
        
    else 1시간 이후
        WYEApp->>User: "위치 공유는 약속 1시간 전부터 가능합니다" 메시지
    end
    
    Note over WYEApp: 다른 참여자들의 위치 조회
    
    WYEApp->>SupabaseKV: kv.getByPrefix(`realtime_location_${meetupId}_`)
    SupabaseKV-->>WYEApp: allLocations[]
    
    WYEApp->>WYEApp: setRealtimeLocations(allLocations)
    
    loop 각 위치
        WYEApp->>WYEApp: isLocationActive(timestamp)
        Note over WYEApp: 현재 시간 - timestamp < 5분이면 활성
        
        WYEApp->>KakaoGeocoder: coord2Address(lng, lat)
        KakaoGeocoder-->>WYEApp: address
        WYEApp->>WYEApp: locationAddresses[userId] = address
    end
    
    WYEApp->>RealtimeLocationMap: 렌더링 (locations, participants, meetingPlace)
    
    RealtimeLocationMap->>RealtimeLocationMap: 지도 초기화
    Note over RealtimeLocationMap: 중심: meetingPlace 또는 참여자 위치 평균
    
    loop 각 참여자
        RealtimeLocationMap->>RealtimeLocationMap: 커스텀 오버레이 생성
        Note over RealtimeLocationMap: 활성: 초록색 테두리<br/>비활성: 회색 테두리
        
        RealtimeLocationMap->>RealtimeLocationMap: overlay.setMap(map)
    end
    
    RealtimeLocationMap->>RealtimeLocationMap: meetingPlace 마커 추가
    RealtimeLocationMap->>RealtimeLocationMap: fitMapToBounds() - 모든 마커 포함
    
    RealtimeLocationMap->>User: 실시간 위치 지도 표시
    
    User->>WYEApp: "공유 중단" 클릭
    WYEApp->>WYEApp: clearInterval(locationUpdateInterval)
    WYEApp->>WYEApp: setIsLocationSharingEnabled(false)
    WYEApp->>WYEApp: setActiveMeetupId(null)
    WYEApp->>User: 위치 공유 중단됨
```

## 6. 길찾기 흐름

```mermaid
sequenceDiagram
    actor User
    participant WYEApp
    participant KakaoGeocoder as Kakao Geocoder API
    participant KakaoMap as Kakao Map App
    participant Browser

    User->>WYEApp: 약속 상세 > 참여자 옆 "길찾기" 버튼 클릭
    
    WYEApp->>WYEApp: geocodeAddress(participant.location)
    
    alt 실시간 위치 공유 중
        WYEApp->>WYEApp: startCoords = {<br/>lat: realtimeLocation.latitude,<br/>lng: realtimeLocation.longitude }
        
    else 저장된 주소만 있음
        WYEApp->>KakaoGeocoder: kakao.maps.load()
        KakaoGeocoder-->>WYEApp: SDK 로드 완료
        
        WYEApp->>KakaoGeocoder: new Geocoder()
        WYEApp->>KakaoGeocoder: addressSearch(address, callback)
        
        alt 주소 검색 성공
            KakaoGeocoder-->>WYEApp: { result: [{ x, y }] }
            WYEApp->>WYEApp: startCoords = { lat: parseFloat(y), lng: parseFloat(x) }
            
        else 주소 검색 실패
            Note over WYEApp,KakaoGeocoder: Status: ZERO_RESULT
            
            WYEApp->>KakaoGeocoder: new Places()
            WYEApp->>KakaoGeocoder: keywordSearch(address, callback)
            Note over WYEApp,KakaoGeocoder: Fallback: 키워드로 장소 검색
            
            alt 키워드 검색 성공
                KakaoGeocoder-->>WYEApp: { result: [{ x, y }] }
                WYEApp->>WYEApp: startCoords = { lat: parseFloat(y), lng: parseFloat(x) }
                
            else 키워드 검색 실패
                WYEApp->>User: alert("출발지 주소를 찾을 수 없습니다.")
                WYEApp->>WYEApp: return
            end
        end
    end
    
    WYEApp->>WYEApp: getNavigationUrl(startCoords, endLat, endLng, endName, transport)
    
    alt transport === "car"
        WYEApp->>WYEApp: by = "car"
    else transport === "transit"
        WYEApp->>WYEApp: by = "publictransit"
    end
    
    WYEApp->>WYEApp: navUrl = `http://m.map.kakao.com/scheme/route?<br/>sp=${startCoords.lat},${startCoords.lng}<br/>&ep=${endLat},${endLng}<br/>&by=${by}`
    
    WYEApp->>Browser: window.open(navUrl, '_blank')
    
    alt 모바일 환경 + 카카오맵 앱 설치됨
        Browser->>KakaoMap: 앱 실행
        KakaoMap->>User: 길찾기 화면 (출발지→도착지)
        
    else 웹 환경 또는 앱 미설치
        Browser->>User: 카카오맵 웹 페이지 오픈
        Note over Browser,User: 새 탭에서 길찾기 표시
    end
```

## 7. 약속 수정 흐름

```mermaid
sequenceDiagram
    actor User
    participant WYEApp
    participant MeetupStorage
    participant SupabaseKV
    participant NotificationHelper

    User->>WYEApp: 약속 상세 > "수정" 버튼
    WYEApp->>WYEApp: setIsEditingMeetup(true)
    WYEApp->>WYEApp: 기존 약속 정보 폼에 채우기
    Note over WYEApp: meetupName = currentMeetup.title<br/>meetupDate = currentMeetup.date<br/>meetupTime = currentMeetup.time<br/>meetupPurpose = currentMeetup.purpose<br/>participants = currentMeetup.participants<br/>selectedLocation = currentMeetup.selectedLocation
    
    WYEApp->>WYEApp: setOriginalMeetupPurpose(currentMeetup.purpose)
    
    User->>WYEApp: 정보 수정 (참여자 추가/삭제, 시간 변경 등)
    
    alt 약속 장소 유형(purpose) 변경됨
        User->>WYEApp: purpose 변경
        WYEApp->>WYEApp: setSelectedLocation(null)
        Note over WYEApp: 장소 초기화 - 재추천 필요
    end
    
    User->>WYEApp: "수정 완료" 버튼
    
    WYEApp->>WYEApp: updatedMeetup = {<br/>...currentMeetup,<br/>title: meetupName,<br/>date: meetupDate,<br/>time: meetupTime,<br/>purpose: meetupPurpose,<br/>participants: participants,<br/>selectedLocation: selectedLocation,<br/>updatedAt: new Date().toISOString() }
    
    WYEApp->>MeetupStorage: updateMeetup(updatedMeetup)
    MeetupStorage->>SupabaseKV: kv.set(`meetup_${id}`, updatedMeetup)
    SupabaseKV-->>MeetupStorage: success
    MeetupStorage-->>WYEApp: success
    
    alt 새로운 참여자 추가됨
        loop 각 신규 참여자
            WYEApp->>MeetupStorage: addMeetupToUser(participantId, meetupId)
            MeetupStorage->>SupabaseKV: kv.get(`user_meetups_${participantId}`)
            SupabaseKV-->>MeetupStorage: meetupIds[]
            MeetupStorage->>SupabaseKV: kv.set(`user_meetups_${participantId}`, [...meetupIds, meetupId])
            
            WYEApp->>NotificationHelper: createNotification(participantId, "약속 초대")
            NotificationHelper->>SupabaseKV: kv.get(`notifications_${participantId}`)
            NotificationHelper->>SupabaseKV: kv.set(`notifications_${participantId}`, updatedNotifications)
        end
    end
    
    alt 참여자 제거됨
        loop 각 제거된 참여자
            WYEApp->>MeetupStorage: removeMeetupFromUser(participantId, meetupId)
            MeetupStorage->>SupabaseKV: kv.get(`user_meetups_${participantId}`)
            SupabaseKV-->>MeetupStorage: meetupIds[]
            MeetupStorage->>MeetupStorage: meetupIds.filter(id !== meetupId)
            MeetupStorage->>SupabaseKV: kv.set(`user_meetups_${participantId}`, filteredIds)
        end
    end
    
    WYEApp->>WYEApp: setMeetups(meetups.map(m => m.id === updatedMeetup.id ? updatedMeetup : m))
    WYEApp->>WYEApp: setCurrentMeetup(updatedMeetup)
    WYEApp->>WYEApp: setIsEditingMeetup(false)
    WYEApp->>User: "약속이 수정되었습니다!" 토스트
```

## 8. 데이터 구조 및 저장소 관계

```mermaid
classDiagram
    class WYEApp {
        -User currentUser
        -Meetup[] meetups
        -Participant[] participants
        -Location selectedLocation
        -RealtimeLocation[] realtimeLocations
        -Notification[] notifications
        -string currentScreen
        -boolean isLocationSharingEnabled
        +handleLogin()
        +createMeetup()
        +updateMeetup()
        +deleteMeetup()
        +toggleLocationSharing()
        +geocodeAddress(address)
        +getNavigationUrl()
    }
    
    class User {
        +string id
        +string name
        +string email
        +string profileImage
    }
    
    class Meetup {
        +string id
        +string title
        +string date
        +string time
        +string purpose
        +Participant[] participants
        +Location selectedLocation
        +string status
        +string createdBy
        +string[] items
        +string createdAt
        +string updatedAt
    }
    
    class Participant {
        +string id
        +string name
        +string location
        +string transport
    }
    
    class Location {
        +string name
        +string address
        +number lat
        +number lng
        +number rating
        +string distance
        +string type
        +number distanceVariance
        +number avgDistance
    }
    
    class RealtimeLocation {
        +string userId
        +number latitude
        +number longitude
        +number timestamp
    }
    
    class Notification {
        +string id
        +string type
        +string message
        +string timestamp
        +boolean read
        +string meetupId
        +string meetupTitle
    }
    
    class kakaoTokenManager {
        +saveTokenData(data)
        +getTokenData()
        +isAccessTokenValid()
        +getAccessToken()
        +refreshAccessToken()
        +clearTokenData()
    }
    
    class meetupStorage {
        +saveMeetup(meetup)
        +getMeetup(id)
        +updateMeetup(meetup)
        +deleteMeetup(id)
        +getUserMeetups(userId)
        +addMeetupToUser(userId, meetupId)
    }
    
    class fairDistanceCalculator {
        +calculateDistance(lat1, lng1, lat2, lng2)
        +calculateGeometricMedian(participants)
        +calculateFairDistancePlaces(places, participants)
        +calculateParticipantDetails(participants, selectedLocation)
    }
    
    class friendsManager {
        +loadFriends(userId)
        +saveFriends(userId, friends)
        +addFriend(userId, friend)
        +removeFriend(userId, friendId)
    }
    
    class userProfile {
        +saveProfile(userId, profile)
        +getProfile(userId)
        +updateProfile(userId, updates)
    }
    
    class KakaoMap {
        -any mapInstance
        -any[] polylinesRef
        -any[] participantMarkersRef
        +drawRoute(participant)
        +addMeetingPlaceMarker()
        +fitMapToBounds()
    }
    
    class RealtimeLocationMap {
        -any mapInstance
        -any[] overlaysRef
        +addMeetingPlaceMarker()
        +fitMapToBounds()
    }
    
    class SupabaseKV {
        +get(key)
        +set(key, value)
        +getByPrefix(prefix)
        +mget(keys)
        +mset(entries)
        +del(key)
    }
    
    WYEApp --> User
    WYEApp --> Meetup
    WYEApp --> RealtimeLocation
    WYEApp --> Notification
    Meetup --> Participant
    Meetup --> Location
    
    WYEApp ..> kakaoTokenManager : uses
    WYEApp ..> meetupStorage : uses
    WYEApp ..> fairDistanceCalculator : uses
    WYEApp ..> friendsManager : uses
    WYEApp ..> userProfile : uses
    WYEApp --> KakaoMap : renders
    WYEApp --> RealtimeLocationMap : renders
    
    kakaoTokenManager ..> SupabaseKV : stores in localStorage
    meetupStorage ..> SupabaseKV : uses
    friendsManager ..> SupabaseKV : uses
    userProfile ..> SupabaseKV : uses
```

## 9. 주요 알고리즘 및 계산식

### 9.1 Haversine 거리 계산
```
R = 6371 (지구 반지름 km)
dLat = (lat2 - lat1) × π / 180
dLng = (lng2 - lng1) × π / 180

a = sin²(dLat/2) + cos(lat1 × π/180) × cos(lat2 × π/180) × sin²(dLng/2)
c = 2 × atan2(√a, √(1-a))
distance = R × c
```

### 9.2 Geometric Median (Weiszfeld 알고리즘)
```
초기값: centerLat = 평균위도, centerLng = 평균경도

반복 (최대 100회):
  totalWeight = 0
  newLat = 0
  newLng = 0
  
  각 참여자 p:
    d = calculateDistance(centerLat, centerLng, p.lat, p.lng)
    만약 d > 0:
      weight = 1 / d
      totalWeight += weight
      newLat += p.lat × weight
      newLng += p.lng × weight
  
  newLat /= totalWeight
  newLng /= totalWeight
  
  이동거리 = calculateDistance(centerLat, centerLng, newLat, newLng)
  만약 이동거리 < 0.001: 종료
  
  centerLat = newLat
  centerLng = newLng

반환: { lat: centerLat, lng: centerLng }
```

### 9.3 거리 편차 계산 (공평도)
```
각 장소 place:
  distances = []
  
  각 참여자 p:
    d = calculateDistance(place.lat, place.lng, p.lat, p.lng)
    distances.push(d)
  
  avgDistance = sum(distances) / distances.length
  
  variance = √(Σ(d - avgDistance)² / distances.length)
  
  place.distanceVariance = variance
  place.avgDistance = avgDistance

정렬: places.sort((a, b) => a.distanceVariance - b.distanceVariance)
```

### 9.4 이동 시간 추정
```
만약 transport === "car":
  avgSpeed = 40 km/h
아니면:
  avgSpeed = 25 km/h

time (분) = (distance / avgSpeed) × 60
```

## 10. 외부 API 호출 정리

### 10.1 Kakao Mobility API (자동차 경로)
```
GET https://apis-navi.kakaomobility.com/v1/directions
Headers:
  Authorization: KakaoAK {REST_API_KEY}
Query:
  origin={startLng},{startLat}
  destination={endLng},{endLat}
  priority=RECOMMEND

Response:
  routes[0].sections[0].roads[].vertexes (경로 좌표 배열)
```

### 10.2 ODsay API (대중교통 경로)
```
GET https://api.odsay.com/v1/api/searchPubTransPathT
Query:
  SX={startLng}
  SY={startLat}
  EX={endLng}
  EY={endLat}
  apiKey={API_KEY} (URL encoded)

Response:
  result.path[0].subPath[] (구간별 경로 정보)
  - trafficType: 1(지하철), 2(버스), 3(도보)
  - passStopList.lane[] (경유 정류장 좌표)
```

### 10.3 Kakao Places API (장소 검색)
```
JavaScript SDK:
  places = new kakao.maps.services.Places()
  places.keywordSearch(keyword, callback, {
    location: new kakao.maps.LatLng(centerLat, centerLng),
    radius: 3000
  })

Response:
  결과 배열 [{
    place_name,
    address_name,
    x (경도),
    y (위도),
    phone,
    category_name
  }]
```

### 10.4 Kakao Geocoder API (주소→좌표)
```
JavaScript SDK:
  geocoder = new kakao.maps.services.Geocoder()
  geocoder.addressSearch(address, callback)

Fallback (키워드 검색):
  places = new kakao.maps.services.Places()
  places.keywordSearch(address, callback)

Response:
  result[0] { x (경도), y (위도) }
```

### 10.5 Kakao Coord2Address API (좌표→주소)
```
JavaScript SDK:
  geocoder = new kakao.maps.services.Geocoder()
  geocoder.coord2Address(lng, lat, callback)

Response:
  address.address_name (지번 주소)
  또는 road_address.address_name (도로명 주소)
```

## 11. 저장소 키 구조 (Supabase KV)

```
토큰:
  - kakao_token_data (localStorage)

사용자:
  - user_profile_{userId}
  - user_meetups_{userId} (약속 ID 배열)
  - friends_{userId} (친구 목록)
  - crews_{userId} (크루 목록)

약속:
  - meetup_{meetupId}
  - realtime_location_{meetupId}_{userId}

알림:
  - notifications_{userId}

설정:
  - lateCheckedMeetups (Set<meetupId>) (localStorage)
  - unreadMeetups (Set<meetupId>) (localStorage)
```

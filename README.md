

# 서비스 시나리오

기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다.
2. 고객이 선택한 메뉴에 대해 결제한다.
3. 주문이 되면 주문 내역이 입점상점주인에게 주문정보가 전달된다
4. 상점주는 주문을 수락하거나 거절할 수 있다
5. 상점주는 요리시작때와 완료 시점에 시스템에 상태를 입력한다
6. 고객은 아직 요리가 시작되지 않은 주문은 취소할 수 있다
7. 요리가 완료되면 고객의 지역 인근의 라이더들에 의해 배송건 조회가 가능하다
8. 라이더가 해당 요리를 Pick한 후, 앱을 통해 통보한다.
9. 고객이 주문상태를 중간중간 조회한다
10. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다
11. 고객이 요리를 배달 받으면 배송확인 버튼을 탭하여, 모든 거래가 완료된다
12. (추가사항)고객이 상점의 배달된 요리를 평가점수를와 댓글을 작성한다.
13. (추가사항)상점주는 고객이 평가한 요리에 대한 댓글을 작성한다.

비기능적 요구사항

1. 장애격리
    1. 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
    2. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
2. 성능
    1. 고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다 CQRS
    2. 배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다 Event driven

<img width="825" alt="스크린샷 2022-12-06 오전 9 24 47" src="https://user-images.githubusercontent.com/118098096/205776104-3a63fe08-e5e0-4943-8a6a-7e14e0df7545.png">

# 체크포인트
1. Saga (Pub/Sub)
2. CQRS
3. Compensation/Correlation
4. Request/Response
5  Circuit Breaker
6. Gateway/Ingress

# Saga(Pub/Sub)
<img width="483" alt="스크린샷 2022-12-06 오전 10 22 47" src="https://user-images.githubusercontent.com/118098096/205785237-f54eb5b3-b126-4b15-af97-2aefc0dbc78b.png">
OrderAccepted, OrderRejected, CookStarted, CookFinished, Commented, Picked, Deliver event들이  notify policy와 pub/sub 관계를 이룬다.

# CQRS

# Compensation/Correaltion

# Request/Response

# Circuit Breaker

# Gateway/Ingress



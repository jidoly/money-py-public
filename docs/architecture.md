# 아키텍처

## 데이터 흐름

```mermaid
flowchart LR
    KIS[KIS WebSocket<br/>실시간 호가/체결] --> COL[수집기<br/>collector]
    MOCK[Mock 생성기<br/>개발/테스트] -.대체.-> COL
    COL --> BUF[링버퍼<br/>deque maxlen]
    BUF --> FE[피처 엔진<br/>Polars · 8 features]
    FE --> ML[LightGBM<br/>predict_proba]
    ML --> ST[거미줄 전략<br/>grid + EV 게이트]
    ST --> EX[실행 엔진<br/>주문 상태머신]
    EX --> BRK[KIS REST<br/>지정가 주문]
    PAPER[페이퍼 브로커] -.대체.-> EX
    EX --> RISK[리스크 매니저<br/>한도 · 킬스위치]

    BUF --> API[FastAPI 서빙]
    EX --> API
    RISK --> API
    API <-->|WebSocket + 폴링| UI[React 어드민<br/>대시보드]
```

## 연구 파이프라인 (오프라인)

```mermaid
flowchart LR
    REC[야간 자동 녹화<br/>parquet] --> SPLIT[purged split<br/>+ embargo]
    SPLIT --> TRAIN[트리플 배리어 라벨링<br/>LightGBM 학습]
    SPLIT --> BT[백테스터<br/>이벤트 리플레이]
    TRAIN --> BT
    BT --> WF[walk-forward<br/>out-of-sample 검증]
    BT --> EDGE[edge 분석<br/>확신도·시간대·심볼]
    WF --> REPORT[자동 리포트]
    EDGE --> REPORT
```

## 설계 원칙

- **단일 이벤트 루프** — asyncio 하나로 수집·추론·실행을 논블로킹 처리. 링버퍼(deque)로
  락 없는 고속 적재.
- **프로세스 내 메모리 접근** — 수집→피처→추론→실행이 같은 프로세스 메모리를 공유해
  네트워크/직렬화 오버헤드 제거.
- **부품 교체 가능** — 수집기(KIS↔mock)·브로커(실거래↔페이퍼)를 팩토리로 주입.
  같은 의사결정 코어를 라이브와 백테스트가 공유.
- **안전 우선** — 실거래는 다중 플래그 없이 차단. 비밀키는 격리·마스킹. 거래소 유량 제한 준수.

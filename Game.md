# 모래성 게임 (Sand Castle) — 전체 설계 문서

> Claude Code를 위한 구현 스펙 문서입니다. 이 문서를 기반으로 전체 게임을 구현하세요.

---

## 1. 프로젝트 개요

### 컨셉
넓은 해변을 배경으로 플레이어들이 각자의 구역에 모래성을 짓고 서로 구경하는 **창작 샌드박스 게임**. 공격 요소 없음. 경쟁은 오직 예술점수로만.

### 플랫폼
- **앱인토스 (Apps in Toss)** 미니앱으로 출시
- WebView 방식: `npx create-ait-app` 기반
- 모바일 우선 (터치 인터페이스)

### 기술 스택
```
렌더링:      Canvas 2D + 청크 기반 카메라 시스템
조각 시스템: 픽셀 마스크 (Float32Array per block)
물리엔진:    Matter.js
DB:          Supabase (PostgreSQL)
실시간:      Supabase Realtime (blocks 테이블 구독)
인증:        토스 로그인 → Supabase users 테이블 매핑
결제:        앱인토스 인앱결제 SDK
리더보드:    앱인토스 리더보드 API
푸시 알림:   앱인토스 푸시 API
프레임워크:  @apps-in-toss/web-framework
```

---

## 2. 맵 구조

세 구역이 횡스크롤로 이어진 하나의 넓은 맵. 플레이어는 어느 구역이든 자유롭게 이동 가능.

```
[ 백사장 ] ──────── [ 해안 ] ──────── [ 성 구역 (좌우로 동적 확장) ]
  모래 수급            물 접촉              구역 선점 & 건축
```

### 2-1. 백사장 (Desert Beach)
- 밝고 건조한 모래밭 배경
- **삽**을 들고 클릭하면 모래 블럭 1개 → 인벤토리 추가
- 무한 수급 가능
- 시각적으로 클릭 위치에 삽질 애니메이션 재생
- 삽 업그레이드 시 클릭당 수급량 증가 (2개, 3개, 5개 단계)

### 2-2. 해안 (Shoreline)
- 얕은 바닷물이 찰랑이는 구역
- **밀물/썰물 사이클** 적용 → 썰물 때 접근 가능 면적 증가
- 인벤토리에서 모래 블럭 선택 후 물 영역에 드래그 → **젖은 모래**로 변환
- **물 양동이**: 해안에서 클릭하면 물 채움 → 인벤토리에 보관 → 성 구역에서 블럭에 사용하면 해당 블럭이 젖은 모래로 변환
- 물 양동이는 기본 1개 지급, 추가 구매 가능 (유료)

### 2-3. 성 구역 (Castle Zone)
- 전체 맵에서 가장 넓은 구역. 좌우로 동적 확장
- 슬롯 단위로 구역 선점 및 소유권 관리
- **기본 구역 폭**: 1슬롯 무료 제공
- **구역 확장**: 좌우 각 1슬롯씩 추가 가능 (유료 인앱결제)
- 확장 시 맵 전체 길이 증가
- 밀물 때 하단부 블럭이 물에 잠길 수 있음 → 높이 전략 중요

---

## 3. 인벤토리 시스템

맵 이동과 무관하게 항상 유지되는 글로벌 인벤토리.

### 기본 구성
| 아이템 | 기본 제공 | 설명 |
|---|---|---|
| 삽 | ✅ | 백사장에서 모래 블럭 수급 |
| 나뭇가지 | ✅ | 모래성 조각 및 음각 |
| 물 양동이 | ✅ 1개 | 해안에서 물 채워 성에 붓기 |
| 모래 블럭 (마른) | 수급량만큼 | 기본 건축 재료 |
| 모래 블럭 (젖은) | 변환량만큼 | 안정성 강화 재료 |
| 특수 블럭 | 이벤트/유료 | 조개, 나무판자, 돌 등 |

### 인벤토리 슬롯
- 기본 20슬롯
- 확장권 구매 시 +10슬롯씩 추가 (유료)

---

## 4. 블럭 시스템

### 블럭 종류 및 물리 특성
| 블럭 | 획득 방법 | 질량 | 점착력 | 특이사항 |
|---|---|---|---|---|
| 마른 모래 | 백사장 수급 | 낮음 | 낮음 | 높이 쌓으면 무너지기 쉬움 |
| 젖은 모래 | 해안 변환 / 양동이 | 중간 | 높음 | 구조 안정성 우수 |
| 조개 | 계절 이벤트 | 낮음 | 낮음 | 장식용, 예술점수 +보너스 |
| 나무판자 | 유료 | 중간 | 중간 | 수평 브릿지 구조 가능 |
| 돌 | 유료 | 높음 | 높음 | 기단부에 최적, 매우 안정적 |
| 시즌 블럭 | 시즌 한정 | 블럭별 상이 | 블럭별 상이 | 크리스마스 눈, 여름 산호 등 |

### 픽셀 마스크 구조
각 블럭은 내부적으로 픽셀 마스크를 보유:
```typescript
interface Block {
  id: string;
  type: BlockType;
  x: number;         // 맵 좌표
  y: number;
  plotId: string;    // 소속 구역
  pixelMask: Float32Array;  // 픽셀별 상태 (1.0 = 원본, 0.5 = 음각, 0.0 = 제거)
  isWet: boolean;
  health: number;    // 침식 내구도
}
```

---

## 5. 나뭇가지 조각 시스템

### 5-1. 겉면 깎기 모드 (Surface Carving)
- 블럭 표면에 나뭇가지 드래그 → 해당 픽셀 영역 제거 (`pixelMask = 0.0`)
- 블럭 실루엣을 자유롭게 다듬기 가능 (둥근 탑, 뾰족한 첨탑 등)

### 5-2. 음각 새기기 모드 (Engraving)
- 블럭 내부를 파고들어 문양 새기기
- 나뭇가지가 지나간 픽셀: `pixelMask = 0.5` → 렌더링 시 원래 색보다 **30~40% 어둡게** 처리
- 음영 효과로 2D임에도 입체감 있는 조각 느낌 연출
- 블럭을 완전히 관통(pixelMask = 0.0)하면 구멍(투명) 처리

### 5-3. 렌더링 규칙
```
pixelMask 값 → 렌더링 색상
  1.0         →  원본 블럭 색상 (100%)
  0.5         →  원본 색상 * 0.65 (음각 음영)
  0.0         →  투명 (완전 제거)
```

### 5-4. 데이터 저장
- 블럭 단위로 pixelMask를 직렬화 → Supabase `blocks` 테이블에 저장
- 변경 시 Supabase Realtime으로 구독자에게 즉시 브로드캐스트

---

## 6. 물리엔진 (Matter.js)

### 기본 규칙
- 모든 블럭은 Matter.js Body로 등록
- 블럭 종류별 질량(`mass`)과 마찰(`friction`) 값 차등 적용
- 무게중심이 지지점에서 벗어나면 기울어지다 붕괴

### 블럭별 Matter.js 파라미터
```javascript
const BLOCK_PHYSICS = {
  dry_sand:    { mass: 1.0, friction: 0.4, restitution: 0.1 },
  wet_sand:    { mass: 2.5, friction: 0.9, restitution: 0.05 },
  wood_plank:  { mass: 1.5, friction: 0.6, restitution: 0.2 },
  stone:       { mass: 5.0, friction: 0.8, restitution: 0.0 },
  shell:       { mass: 0.5, friction: 0.3, restitution: 0.3 },
};
```

### 안정성 지표
- 현재 성 전체의 구조적 안정성을 0~100%로 실시간 계산
- 무게중심, 지지 블럭 수, 젖은 모래 비율 등을 종합
- UI 상단에 게이지 형태로 표시
- 음각 조각으로 내부를 과도하게 파내면 안정성 하락
- 양동이로 물을 부으면 즉시 안정성 상승

### Realtime 동기화 방식
- 클라이언트에서 물리 시뮬레이션 실행
- 블럭이 안착한 최종 좌표를 DB에 저장
- 구경 중인 유저는 Supabase Realtime으로 블럭 변경 이벤트를 수신해 화면에 반영
- 물리 연출은 각 클라이언트 로컬에서 재생, 최종 상태로 주기적 보정(reconciliation)

---

## 7. 자연 현상 시스템

### 7-1. 밀물/썰물 사이클
- 실제 시간 기반, 하루 4사이클 (6시간마다)
- 성 구역 하단부 블럭이 물에 잠기면:
  - 마른 모래 블럭 → 젖은 모래로 변환 또는 유실 (확률 기반)
  - 젖은 모래 블럭 → 안전
- **밀물 10분 전** 앱인토스 푸시 알림 발송
- UI에 현재 조수 상태 및 다음 밀물/썰물까지 남은 시간 표시

### 7-2. 바람
- 시간대별 방향/세기 랜덤 변동 (Perlin Noise 기반 권장)
- 블럭 배치 시 물리 방향에 영향 (바람 방향으로 힘 추가)
- 바람이 강할 때 불안정한 블럭(안정성 낮은 것)이 밀릴 수 있음
- UI 상단에 바람 방향 인디케이터(화살표) + 세기(약/중/강) 표시

### 7-3. 모래 유실 (침식)
- 마지막 접속 후 **7일 경과** 시 침식 시작
- 하루에 블럭 최대 N개(설정값) 랜덤 유실
- 접속하면 침식 즉시 중단
- **방파제 아이템** (유료, 기간제) 설치 시 미접속 중에도 침식 완전 방지

---

## 8. Supabase DB 스키마

```sql
-- 유저
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  toss_user_id TEXT UNIQUE NOT NULL,  -- 토스 로그인 식별자
  nickname TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 구역 (슬롯)
CREATE TABLE plots (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID REFERENCES users(id),
  position_x INT NOT NULL,           -- 맵 상의 슬롯 X 위치
  width INT DEFAULT 1,               -- 슬롯 폭 (확장 시 증가)
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_visited_at TIMESTAMPTZ DEFAULT NOW()
);

-- 공동 편집 권한
CREATE TABLE plot_collaborators (
  plot_id UUID REFERENCES plots(id),
  user_id UUID REFERENCES users(id),
  PRIMARY KEY (plot_id, user_id)
);

-- 블럭
CREATE TABLE blocks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plot_id UUID REFERENCES plots(id),
  x FLOAT NOT NULL,
  y FLOAT NOT NULL,
  type TEXT NOT NULL,                -- 'dry_sand' | 'wet_sand' | 'stone' | ...
  is_wet BOOLEAN DEFAULT FALSE,
  pixel_mask BYTEA,                  -- Float32Array 직렬화
  health INT DEFAULT 100,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 예술점수
CREATE TABLE art_scores (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plot_id UUID REFERENCES plots(id),
  voter_id UUID REFERENCES users(id),
  score INT CHECK (score BETWEEN 1 AND 5),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (plot_id, voter_id)         -- 1인 1회 투표
);

-- 방명록
CREATE TABLE guestbook (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plot_id UUID REFERENCES plots(id),
  author_id UUID REFERENCES users(id),
  message TEXT CHECK (char_length(message) <= 50),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 인벤토리
CREATE TABLE inventories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) UNIQUE,
  slots JSONB DEFAULT '[]',          -- 슬롯 배열
  max_slots INT DEFAULT 20
);
```

### Realtime 구독 채널
```typescript
// 특정 구역의 블럭 변경만 구독 (plot 단위로 좁혀서 트래픽 최소화)
supabase
  .channel(`plot:${plotId}`)
  .on('postgres_changes', {
    event: '*',
    schema: 'public',
    table: 'blocks',
    filter: `plot_id=eq.${plotId}`
  }, handleBlockChange)
  .subscribe();
```

---

## 9. 소셜 기능

### 9-1. 예술점수
- 타인의 성 구경 중 별점(1~5) 부여 가능
- 하루 10회 투표 제한 (user 단위)
- 총 평균 예술점수 기준 **앱인토스 리더보드 API**에 등록
- 내 성의 점수 변화 시 푸시 알림 옵션 제공

### 9-2. 방명록
- 타인의 구역 방문 시 최대 50자 메시지 남기기
- 성주에게 방명록 알림 발송 (앱인토스 푸시)
- 성주는 방명록 삭제 권한 보유

### 9-3. 성 투어 모드
- 예술점수 상위 10개 성을 자동 순회하는 슬라이드쇼
- 각 성에서 5~10초 머문 후 자동 이동
- 메인 화면에서 "둘러보기" 버튼으로 진입
- 신규 유저 온보딩 역할 겸임

### 9-4. 협동 구역
- 구역 소유자가 최대 3명까지 공동 편집 권한 부여
- 공동 작업 중 Supabase Realtime으로 서로의 작업 즉시 반영
- 협동 구역 표시 아이콘 (리더보드 및 지도에 표시)

---

## 10. 건축 도전 과제

| 과제 ID | 과제명 | 달성 조건 | 보상 |
|---|---|---|---|
| `first_dig` | 첫 삽 | 모래 블럭 최초 수급 | 칭호: 견습 미장이 |
| `tower` | 탑 쌓기 | 높이 20블럭 이상 | 특수 블럭 해금 |
| `arch` | 아치 건축가 | 아치형 구조 감지 | 칭호: 건축가 |
| `sculptor` | 조각가 | 음각 문양 100픽셀 이상 | 칭호: 조각가 |
| `survivor` | 생존자 | 밀물 3회 버티기 | 방파제 아이템 1개 |
| `collab` | 협업의 달인 | 3인 공동 작업 완성 | 칭호: 협업의 달인 |
| `high_art` | 예술가 | 예술점수 평균 4.5 이상 | 시즌 한정 블럭 팩 |

- 달성 시 **앱인토스 프로모션 API** (`grantPromotionRewardForGame`) 호출 → 토스포인트 지급
- 토스포인트 지급은 게임 카테고리 미니앱 전용 API 사용
- 프로모션 예산은 비즈 월렛에 사전 충전 필요

---

## 11. 유료 아이템 (인앱결제)

| 아이템 ID | 아이템명 | 설명 | 가격 (예시) |
|---|---|---|---|
| `shovel_lv2` | 삽 Lv.2 | 클릭당 모래 2개 수급 | 990원 |
| `shovel_lv3` | 삽 Lv.3 | 클릭당 모래 3개 수급 | 1,990원 |
| `shovel_lv5` | 삽 Lv.5 | 클릭당 모래 5개 수급 | 3,990원 |
| `bucket_extra` | 물 양동이 추가 | 양동이 슬롯 +1 | 990원 |
| `plot_expand` | 구역 확장권 | 좌우 슬롯 +1 | 2,990원 |
| `wood_plank_pack` | 나무판자 팩 | 나무판자 50개 | 1,990원 |
| `stone_pack` | 돌 블럭 팩 | 돌 블럭 30개 | 2,990원 |
| `breakwater_7d` | 방파제 7일 | 7일간 침식 방지 | 990원 |
| `breakwater_30d` | 방파제 30일 | 30일간 침식 방지 | 2,990원 |
| `inventory_expand` | 인벤토리 확장 | 슬롯 +10 | 990원 |
| `season_pack` | 시즌 블럭 팩 | 한정 이벤트 블럭 세트 | 4,990원 |

- 모든 결제는 **앱인토스 인앱결제 SDK** 사용
- 가격 범위: 400원 ~ 1,400,000원 (정책 준수)
- 디지털 상품이므로 인앱결제 방식만 사용 (토스페이 불가)

---

## 12. 앱인토스 정책 준수 사항

- **로그인**: 토스 로그인만 사용
- **결제**: 디지털 상품 → 인앱결제만 사용
- **광고**: 앱인토스 전면형/보상형/배너만 허용
- **외부 링크**: 원칙적 금지 (필수 법적 고지 등 예외만 허용)
- **자사 앱 설치 유도 금지**
- **다크 모드 미지원**: 라이트 모드 기준으로만 구현
- **앱 번들 용량**: 100MB 이하 유지 (Matter.js + Canvas 최적화 필수)
- **생성형 AI 사용 시**: 사전 고지 및 AI 결과물 표시 의무
- **게임 등급심의**: 출시 전 게임물관리위원회 또는 자체등급분류사업자 심의 필수
- **사행성 요소 없음**: 확률형 아이템 없음, 모든 유료 아이템은 확정 지급

---

## 13. 프로젝트 구조 (권장)

```
/src
  /canvas
    renderer.ts         # Canvas 2D 렌더링 엔진
    camera.ts           # 청크 기반 카메라 시스템
    chunkManager.ts     # 맵 청크 관리
  /physics
    engine.ts           # Matter.js 초기화 및 관리
    blockBodies.ts      # 블럭별 물리 바디 생성
    stability.ts        # 안정성 지표 계산
  /carving
    pixelMask.ts        # Float32Array 픽셀 마스크 관리
    carvingTool.ts      # 나뭇가지 도구 로직
  /systems
    weather.ts          # 밀물/썰물, 바람 시스템
    erosion.ts          # 침식 시스템
    inventory.ts        # 인벤토리 관리
  /supabase
    client.ts           # Supabase 클라이언트 초기화
    realtime.ts         # Realtime 구독 관리
    blocks.ts           # 블럭 CRUD
    plots.ts            # 구역 관리
    scores.ts           # 예술점수
  /ait
    auth.ts             # 토스 로그인 연동
    payment.ts          # 인앱결제
    leaderboard.ts      # 리더보드 API
    promotion.ts        # 프로모션 (토스포인트 지급)
    push.ts             # 푸시 알림
  /ui
    hud.ts              # 인게임 HUD (안정성 게이지, 바람 인디케이터)
    inventory.ts        # 인벤토리 UI
    leaderboard.ts      # 리더보드 UI
    guestbook.ts        # 방명록 UI
    tourMode.ts         # 성 투어 모드
  /constants
    blocks.ts           # 블럭 물리 파라미터
    maps.ts             # 맵 구역 정의
```

---

## 14. 구현 우선순위

### Phase 1 — 핵심 루프
1. Canvas 렌더링 + 청크 카메라
2. Matter.js 물리엔진 연동
3. 맵 3구역 기본 구현 (백사장, 해안, 성 구역)
4. 삽으로 모래 수급 → 인벤토리 → 성 구역에 블럭 배치
5. Supabase 블럭 저장/불러오기
6. 토스 로그인 연동

### Phase 2 — 조각 & 물리
7. 나뭇가지 픽셀 마스크 조각 시스템
8. 젖은 모래 변환 (해안 담그기 + 양동이)
9. 안정성 지표
10. 밀물/썰물 시스템

### Phase 3 — 소셜 & 실시간
11. Supabase Realtime 블럭 동기화
12. 예술점수 + 리더보드
13. 방명록
14. 성 투어 모드
15. 협동 구역

### Phase 4 — 수익화 & 운영
16. 인앱결제 (구역 확장, 특수 블럭 등)
17. 앱인토스 프로모션 (도전 과제 달성 시 토스포인트)
18. 푸시 알림 (밀물 경보, 방명록 알림)
19. 바람 시스템
20. 침식 시스템 + 방파제 아이템
21. 계절 이벤트 블럭

---

*이 문서는 전체 게임 설계 기준입니다. 구현 중 정책 변경이나 기술적 제약이 발견되면 이 문서를 업데이트하세요.*

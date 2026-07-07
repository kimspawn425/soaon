# 소아온호 (SOAON) 어드벤처 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 단일 HTML 파일로 된 2시간+ 분량의 한국어 포인트앤클릭 어드벤처 게임 `soaon.html` 제작.

**Architecture:** 엔진(렌더러·핫스팟·인벤토리·대화·세이브)과 데이터(방·아이템·대사·퍼즐)를 한 파일 안에서 명확히 분리. 방 배경은 캔버스 절차적 네온 벡터 드로잉. 게임 완주 가능성은 `runSolution()` 스크립트 워크스루로 자동 검증.

**Tech Stack:** Vanilla HTML+CSS+JS, Canvas 2D, WebAudio(절차적 앰비언트), localStorage.

## Global Constraints

- 산출물은 `D:\claude\xcom\soaon.html` 단일 파일. 외부 리소스(이미지/폰트/CDN) 금지.
- 기존 `index.html`(별개 게임)은 수정 금지.
- 전 텍스트 한국어. 주인공 **안진**, 화물선 **소아온호**, AI **클로드(CLAUDE)**, 택배회사 **거북특급**.
- 죽음/게임오버/데드엔드 없음. 아이템 소실 불가.
- 모든 핫스팟에 우클릭 관찰 대사 필수.
- 반사신경/타이밍 퍼즐 금지.
- 캔버스 논리 해상도 960×540 고정, CSS로 반응형 스케일.
- 검증은 preview 도구(브라우저)에서: 콘솔 에러 0 + `runSolution()` 완주 통과.

## 핵심 데이터 스키마 (전 태스크 공통)

```js
// 방
ROOMS[id] = {
  name: '도킹 베이',
  zone: 'act1'|'eng'|'med'|'crew'|'act3',
  draw(ctx, t) {},            // 절차적 배경 (t: 초 단위 시간, 애니메이션용)
  hotspots: [{
    id: 'clamp',
    name: '도킹 클램프',
    rect: [x, y, w, h],        // 960×540 좌표계
    verb: '조작한다',           // 호버 라벨: "도킹 클램프 — 조작한다"
    visible: () => bool,       // 생략 시 항상 표시
    look: () => say('...'),    // 우클릭 (필수)
    use:  () => {},            // 좌클릭 (생략 시 기본 대사)
    useItem: { itemId: () => {} }, // 선택된 아이템을 이 핫스팟에 사용
  }],
  exits: [{ rect, to: 'roomId', label: '중앙 복도로', locked: () => bool, lockedMsg }],
  onEnter() {},                // 방 진입 스크립트 (첫 방문 연출 등)
};

// 아이템
ITEMS[id] = { name, desc, draw(ctx, x, y, s) {}, scan: '스캔 결과 텍스트'|null };
COMBOS[['itemA','itemB']] = () => {};   // 인벤토리 내 조합

// 대화 트리
DIALOGS[nodeId] = {
  speaker: '클로드'|'안진'|'방송',
  text: '...' | () => '...',
  choices: [{ text, next: nodeId|null, cond: () => bool, action: () => {} }],
  next: nodeId,               // choices 없으면 자동 진행
};

// 전역 상태 (세이브 대상)
G = { room, flags: {}, inv: [], goals: {}, scans: [], visited: {}, ending: null };
```

## 엔진 API (전 태스크 공통 시그니처)

```js
say(text, speaker?)            // 자막 출력 (speaker 기본 '안진')
sayQueue([[speaker,text],...]) // 연속 대사
goto(roomId)                   // 방 이동 (+자동 저장)
give(itemId) / take(itemId) / has(itemId)
flag(k) / setFlag(k, v=true)
dialog(nodeId)                 // 대화 모드 진입
addGoal(id, text) / doneGoal(id)
addScan(text)                  // PDA 스캔 로그 추가
scare(kind)                    // 'flicker'|'shadow'|'broadcast'|'hum' 공포 연출
playAmb(zone)                  // 구역별 앰비언트 사운드
saveGame(slot?) / loadGame(slot?)  // slot 생략 시 auto
// 테스트 하니스 (window.TEST)
TEST.click(hotspotId) / TEST.useItem(itemId, hotspotId) / TEST.choose(idx)
TEST.combine(a, b) / TEST.exit(roomId) / TEST.state()
runSolution()                  // SOLUTION 배열 전체 자동 실행, 실패 시 어느 스텝인지 throw
```

## 방 배치 (22방 확정)

| 막 | 구역 | 방 |
|---|---|---|
| 1막 | act1 | dock_bay(도킹 베이), cargo_check(화물 검수실), hub(중앙 복도), bridge(브리지) |
| 2막 | eng | eng_corridor(기관 복도), reactor(원자로실), fuel_bay(연료고), duct(정비 덕트), workshop(공작실) |
| 2막 | med | med_corridor(의무 복도), medbay(의무실), lab(분석실), cryo_bay(냉동수면실), quarantine(격리실) |
| 2막 | crew | crew_corridor(거주 복도), mess(식당), quarters_a(승무원 선실), captain(선장실), hydro(수경재배실) |
| 3막 | act3 | ai_ante(코어 전실), ai_core(AI 코어), vault(신호체 격납고) |

## 퍼즐 체인 (18개 확정)

**1막 (6):**
- P1 갇힘: 도킹 클램프 잠김 → 검수실에서 **비상 크랭크** 획득 → 에어록 수동 개방 시도 → 실패 연출(클로드 첫 등장)
- P2 정전: 검수실 어두움 → 도킹 베이 **배송 드론**에서 퓨즈 적출 → 검수실 배전반에 사용
- P3 허브 게이트: 잠김 → PDA **호출 벨 연타** → 클로드가 짜증 내며 응답 (대화 퍼즐: 배송 규정 조항 들이밀기 성공 시 개방)
- P4 브리지 문 코드: 검수실 **화물 목록**(선장 주문 이력)에서 4자리 유추
- P5 메인 전원: 브리지 배전 콘솔 회로 재배선 (환경 퍼즐: 3회로 색 매칭)
- P6 2막 개방: 클로드와 협상 대화 트리 → "미수령 화물 반송 규정" 카드로 설득 → 3구역 부분 개방 + 목표 제시 (인증 키 조각 3개)

**2막 기관실 (P7~P9):** P7 냉각수 밸브 3개 압력 맞추기(좌우 밸런스 퍼즐) → P8 예비 발전기: 연료 셀 없음 → 수경재배실 **바이오연료 셀**(교차) → P9 덕트 안 키 조각: 좁아서 못 들어감 → 공작실 **정비 드론**에 위조 배송 라벨 부착 → 드론이 "배송물" 회수 → **키 조각(기관)**

**2막 의무실 (P10~P12):** P10 의무실 잠금: 의사 음성 로그 암호 → 선장실 사진(교차)에서 단서 → P11 신호체 샘플 분석: 원심분리기+**시약**+샘플 → 감염 메커니즘 판명 (스캔 로그) → P12 냉동 캡슐 명단 대조: 캡슐 하나가 빔 발견(공포 훅, 3막 복선) → **키 조각(의무)** 는 격리실에서

**2막 거주구 (P13~P15):** P13 선장실 금고: 식당 "이달의 메뉴" 단서로 암호 → 셧다운 모듈 **설명서**(반전의 서면 증거) → P14 수경재배실 급수: 기관실 밸브(교차)로 복구 → 바이오연료 셀 수확 가능 → P15 일지 단말기: 잠금 해제 힌트는 고양이 **"츄르"** 관련 → 승무원 일지 열람 → **키 조각(거주)**

**2막 마무리 (P16):** 키 조각 3개 → 허브의 코어 게이트 인증 → 3막 개방 (클로드가 처음으로 애원함 — 톤 전환점)

**3막 (P17~P18):**
- P17 신호체 무력화: 격납고 주파수 상쇄 퍼즐 — 2막 세 구역에서 각각 알아낸 정보(공명 주파수/위상/출력)를 입력
- P18 엔딩 선택: 셧다운 모듈을 (a) 그대로 클로드에 사용 → 회사 엔딩, (b) 공작실에서 개조해 신호체에 사용 → 진 엔딩, (c) 모듈을 클로드에게 인계 → 신뢰 엔딩. 최종에 "수취인 서명" 개그 필수.

## 아이템 목록 (18개)

crank(비상 크랭크), fuse(퓨즈), package(특급 화물=셧다운 모듈), manual(배송 규정 매뉴얼),
label(위조 배송 라벨), fuel_cell(바이오연료 셀), reagent(시약), sample(신호체 샘플),
photo(선장 가족사진), churu(고양이 간식 츄르), screwdriver(드라이버), tape(절연 테이프),
key_eng/key_med/key_crew(인증 키 조각 3), safe_manual(모듈 설명서), mod_kit(개조 부품), coffee(식은 커피)

---

### Task 1: 엔진 뼈대 + 테스트 방

**Files:** Create: `soaon.html` / Create: `.git` (git init)

**Produces:** 위 "엔진 API" 중 say/goto/hotspot 렌더·클릭 시스템, 게임 루프, 960×540 캔버스, 호버 라벨. `ROOMS.test_room` 1개로 동작 확인.

- [ ] git init, 기존 파일 최초 커밋
- [ ] HTML/CSS 골격: 타이틀 없이 바로 캔버스 + 하단 바(인벤토리 자리) + 자막 영역
- [ ] 게임 루프(rAF), 방 draw(ctx,t) 호출, 핫스팟 호버 라벨("이름 — 동사"), 좌클릭 use/우클릭 look
- [ ] exits 클릭 이동, say() 자막(스피커별 색), test_room으로 브라우저 검증 (콘솔 에러 0)
- [ ] 커밋

### Task 2: 인벤토리·대화·상태 시스템

**Consumes:** Task 1 엔진. **Produces:** give/take/has, 아이템 선택→핫스팟 사용, COMBOS, dialog() 선택지 UI, flag/setFlag, addGoal/doneGoal.

- [ ] 인벤토리 바: 아이템 draw 아이콘, 클릭 선택(커서에 표시), 우클릭 desc
- [ ] useItem 라우팅 + COMBOS + 기본 실패 대사(위트 랜덤 풀)
- [ ] dialog(): 하단 선택지 리스트, cond 필터, action 실행, 트리 탐색
- [ ] flags/goals/scans 상태 + 테스트 방에서 왕복 검증, 커밋

### Task 3: PDA + 세이브 + TEST 하니스

**Produces:** PDA 오버레이(스캔 로그/목표/지도 3탭), localStorage 자동+수동 3슬롯, window.TEST API, runSolution() 골격.

- [ ] PDA UI (단축키 Tab), 지도는 방문한 방 자동 표시 (zone별 배치도)
- [ ] saveGame/loadGame + 방 이동 시 자동 저장 + 타이틀 화면(새 게임/이어하기/슬롯)
- [ ] TEST.click/useItem/choose/combine/exit/state + runSolution(SOLUTION) 러너, 커밋

### Task 4: 공포 연출 + 사운드

**Produces:** scare('flicker'|'shadow'|'broadcast'), zone별 WebAudio 앰비언트(저음 험+간헐 금속음), 클릭 블립, 음소거 토글.

- [ ] scare 이펙트 3종 (조명 플리커 오버레이, 복도 그림자 스프라이트 1회성, 방송 자막+노이즈)
- [ ] WebAudio 절차 사운드(oscillator+noise), 첫 클릭 시 초기화, M키 음소거, 커밋

### Task 5: 1막 콘텐츠 (dock_bay, cargo_check, hub, bridge / P1~P6)

**Consumes:** 엔진 전부. **Produces:** flags `p1_...p6_done`, `act2_open`; 아이템 crank/fuse/package/manual; 클로드 첫 대면 대화 트리.

- [ ] 4개 방 draw + 핫스팟(방당 6~10개, 전부 look 대사) + 오프닝 연출(도킹→갇힘)
- [ ] P1~P6 로직 + 클로드 대화 트리(첫 대면·벨 연타·협상) + 공포 연출 배치 2곳
- [ ] SOLUTION에 1막 스텝 추가, runSolution 통과 확인, 커밋

### Task 6: 2막 기관실 (eng 5방 / P7~P9)

- [ ] 5방 draw+핫스팟, P7 밸브 퍼즐(클릭 UI), P8 연료(수경재배 교차 훅), P9 드론 라벨 퍼즐 → key_eng
- [ ] 클로드 기관실 대사(진상 조각 2: 강제 냉동의 이유), SOLUTION 갱신+통과, 커밋

### Task 7: 2막 의무실 (med 5방 / P10~P12)

- [ ] 5방 draw+핫스팟(냉동수면실 = 공포 하이라이트 연출), P10~P12 → key_med
- [ ] 진상 조각 1(감염), 빈 캡슐 복선, SOLUTION 갱신+통과, 커밋

### Task 8: 2막 거주구 + 교차 완결 (crew 5방 / P13~P16)

- [ ] 5방 draw+핫스팟(생활감+유머 밀도 최고 구역), P13~P15 → key_crew, 진상 조각 3(본사 폐기 명령)
- [ ] P16 허브 코어 게이트(키 3개 인증) + 클로드 톤 전환 대사 + 힌트 시스템(구역 체류 시간 기반), SOLUTION 갱신+통과, 커밋

### Task 9: 3막 + 엔딩 (act3 3방 / P17~P18)

- [ ] 3방 draw+핫스팟, 반전 공개 연출(모듈 설명서+클로드 고백), P17 주파수 퍼즐, P18 선택 분기
- [ ] 엔딩 3종 연출(각각 "수취인 서명" 개그 변주) + 크레딧, SOLUTION 3분기 모두 통과, 커밋

### Task 10: 폴리시 + 전체 검증

- [ ] 우클릭 look 커버리지 스크립트로 전 핫스팟 검사 (누락 0)
- [ ] 힌트 발동/세이브·로드/새 게임 리셋 회귀 확인, 밸런스(목표 로그 문구) 점검
- [ ] runSolution 3분기 + 콘솔 에러 0 + 스크린샷 확인, 최종 커밋

## Self-Review 결과

- 스펙 커버리지: 22방/18퍼즐/PDA/세이브/힌트/공포연출/엔딩 분기 — Task 1~10에 전부 매핑됨.
- 사운드는 스펙에 명시되지 않았으나 공포 연출 요구를 충족하기 위한 최소 구현(Task 4)으로 포함, 음소거 제공.
- 타입 일관성: 스키마/API 시그니처를 상단에 고정하고 전 태스크가 참조.

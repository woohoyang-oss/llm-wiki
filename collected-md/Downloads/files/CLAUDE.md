# MITO UNIVERSE — Welcome to the Other Side
## DPR IAN 세계관 RPG 팬사이트

> "Welcome to the Other Side" — 초록색 문을 열고 들어가면
> MITO가 서 있다. 한쪽 눈이 비어있는 타락 천사.
> 이 사이트에 오면 "아, DPR IAN이구나"를 바로 알 수 있어야 한다.

---

## 핵심 원칙

**이 사이트는 텍스트 RPG가 아니다.** 방문자가 오면:
1. 초록색 문(The Other Side의 입구)이 보인다
2. 문을 열면 MITO가 서 있다 — 한쪽 눈이 빈 타락 천사
3. 배경 음악이 흐른다 — 몽환적이고 불안한
4. "여기가 뭐지? 더 돌아다니고 싶다"는 호기심이 생긴다
5. DPR IAN의 팬이면 "이건 MITO 세계관이네!"를 즉시 인식

---

## 기술 스택
- **Framework**: Next.js 14+ (App Router)
- **Styling**: Tailwind CSS + CSS Variables
- **Animation**: Framer Motion + GSAP (복잡한 시퀀스)
- **3D/Visual**: Three.js 또는 Spline (캐릭터, 문)
- **Audio**: Howler.js (BGM + 효과음)
- **State**: Zustand (persist로 진행도 저장)
- **Deploy**: Vercel
- **Font**: Cinzel (display), Noto Sans KR (body), JetBrains Mono (UI)

---

## 비주얼 에셋 필요 목록

### 🚪 The Green Door (초록색 문)
Dear Insanity MV에서 IAN이 "Welcome to the Other Side"를 말하며 들어가는 문.
- **외형**: 낡은 나무 문, 초록색 페인트가 벗겨진, 틈 사이로 초록빛이 새어나옴
- **구현**: Three.js 또는 Spline으로 3D 문 제작 → 터치하면 문이 열리는 애니메이션
- **대안**: SVG/CSS로 2D 문 + 그림자/빛 효과 + parallax
- **빛**: 문 틈에서 에메랄드 초록 빛이 맥박치듯 나옴
- **사운드**: 문 열릴 때 — 낮은 현악기 울림 + 바람 소리 + "Welcome to the other side" 속삭임

### 👁️ MITO 캐릭터 비주얼
DPR IAN의 alter ego. 시각적으로 가장 중요한 요소.
- **핵심 특징**:
  - 왼쪽 눈이 비어있음 (blank canvas) — 눈 반지를 검지에 착용
  - 흑백(monochrome) 장면에서 주로 등장
  - 쉰 목소리, 천둥 소리에 둘러싸임
  - 검은 깃털 코트 (타락 천사)
  - 가면을 쓴 상태 = 세라핌 (겸손) → 가면 벗으면 = 루시퍼 (타락)
- **구현 옵션 (우선순위 순)**:
  1. **Spline 3D 캐릭터** — 실루엣 형태, 한쪽 눈 빛나는 반인간 천사
  2. **일러스트레이션** — 팬아티스트와 협업하여 제작
  3. **SVG 실루엣** — 미니멀한 검은 실루엣 + 글로우 효과로 한쪽 눈 표현
  4. **CSS Art** — 최소한으로 실루엣 + 눈 글로우만

### 🎭 Mr. Insanity 비주얼
MITO의 반대편 — IAN의 조증(manic high) 페르소나.
- **핵심 특징**:
  - MITO와 반대로 **컬러풀하고 장난스러움**
  - 어디로 갈지 모르는 예측불가
  - The Mask(짐 캐리 영화) 같은 에너지
- **구현**: MITO보다 밝은 컬러, 왜곡된 형태, 보라/녹색 글로우

### 🌈 앨범별 컬러 시스템
IAN이 직접 밝힌 3색 시리즈:
- **MITO EP (2021)**: ⬛ BLACK — 어둠, 공포, 양극성의 저점
- **MIITO Album (2022)**: 🟥 RED — 사랑과 분노의 전환점
- **Dear Insanity EP (2023)**: 🟪 PURPLE/GREEN — 조증, The Other Side
- **(미래)**: ⬜ WHITE — 빛으로의 귀환 (IAN이 예고한 세 번째 색)

---

## 사운드 디자인

### BGM 전략
**저작권 이슈 때문에 DPR IAN의 실제 곡은 직접 재생 불가.**
대신:

1. **앰비언트 사운드스케이프 자체 제작**
   - Nexus: 깊은 공간 울림 + 먼 천둥 + 낮은 드론
   - Mirror Realm: 깨진 유리 사운드 + 역재생 피아노 + 에코
   - Fallen Paradise: 불타는 현악기 + 심장 박동 + 바람
   - The Other Side: 몽환적 신스 + 속삭임 + 시계 소리
   - 참고: Freesound.org, 직접 제작, AI 생성 앰비언트

2. **Spotify/Apple Music 연동**
   - 각 영역에서 "이 영역의 사운드트랙 듣기" 버튼
   - Spotify embed player 또는 링크

3. **효과음 (필수)**
   ```
   - 문 열리는 소리 (intro)
   - 영역 이동 시 — whoosh + 잔향
   - 로어 발견 시 — 유리 종 소리 (crystalline chime)
   - NPC 대화 시작 — 낮은 현악기 한 음
   - NPC 대화 타이핑 — 부드러운 키 소리
   - 맵 열기/닫기 — 접히는 종이 소리
   - MITO 등장 — 천둥 + 역재생 보이스
   ```

4. **Howler.js 구현 예시**
   ```typescript
   // lib/audio.ts
   import { Howl, Howler } from 'howler'

   export const ambience = {
     nexus: new Howl({ src: ['/audio/nexus-ambient.mp3'], loop: true, volume: 0.3 }),
     mirror: new Howl({ src: ['/audio/mirror-ambient.mp3'], loop: true, volume: 0.3 }),
     fallen: new Howl({ src: ['/audio/fallen-ambient.mp3'], loop: true, volume: 0.3 }),
     otherside: new Howl({ src: ['/audio/otherside-ambient.mp3'], loop: true, volume: 0.3 }),
   }

   export const sfx = {
     doorOpen: new Howl({ src: ['/audio/sfx/door-open.mp3'], volume: 0.5 }),
     loreDiscover: new Howl({ src: ['/audio/sfx/chime.mp3'], volume: 0.4 }),
     transition: new Howl({ src: ['/audio/sfx/whoosh.mp3'], volume: 0.3 }),
     thunder: new Howl({ src: ['/audio/sfx/thunder.mp3'], volume: 0.6 }),
     npcStart: new Howl({ src: ['/audio/sfx/string-note.mp3'], volume: 0.3 }),
   }

   export function crossfadeAmbience(from: string, to: string) {
     const current = ambience[from]
     const next = ambience[to]
     if (current) current.fade(0.3, 0, 1500)
     if (next) { next.play(); next.fade(0, 0.3, 1500) }
     setTimeout(() => current?.stop(), 1600)
   }
   ```

---

## 유저 플로우 (시각적)

### 1. 인트로 시퀀스 (/)
```
[검은 화면]
        ↓ 1초
[DPR IAN 로고 페이드인 — 아주 작게, 위쪽]
        ↓ 2초
[중앙에 초록색 문이 서서히 나타남]
[문 틈에서 초록빛이 맥박치듯 새어나옴]
[낮은 앰비언트 드론 + 바람 소리]
        ↓
[문 아래에 텍스트 타이핑: "Welcome to the other side."]
        ↓
[사용자가 문을 탭/클릭]
        ↓
[문이 열리며 초록빛이 화면을 채움]
[효과음: 문 열림 + 현악기 울림]
        ↓ 화면 전환
[The Nexus 영역 진입 — MITO 실루엣이 멀리 서있음]
```

### 2. 영역 탐험 (/game/[zoneId])
```
[상단: 영역 이름 + 아이콘 + 로어 진행도]
[중앙: 영역 배경 비주얼 — 파티클/색감/캐릭터 실루엣]
[하단: 4개 액션 카드 (로어, 트랙, NPC, 이동)]
[배경: 영역별 앰비언트 BGM + 파티클]

카드 탭 → 바텀시트 올라옴
영역 이동 → 화면 페이드 + whoosh 효과음 + 새 영역 BGM 크로스페이드
```

### 3. 로어 발견
```
[로어 카드 탭]
        ↓
[바텀시트에 로어 목록]
        ↓
[개별 로어 탭]
        ↓
[효과음: 크리스탈 차임]
[로어 텍스트 서서히 나타남]
[발견 카운터 +1]
[localStorage에 저장]
```

### 4. NPC 대화
```
[NPC 카드 탭]
        ↓
[NPC 실루엣/아이콘이 화면에 등장]
[효과음: 낮은 현악기]
        ↓
[대화 시작 — RPG 스타일 대화창]
[타이핑 효과로 한 글자씩 나타남]
[MITO 등장 시: 천둥 효과음 + 화면 살짝 흔들림]
```

---

## 캐릭터별 비주얼 상세

### MITO (양극성의 저점 / 타락 천사)
- **비주얼**: 흑백, 왼쪽 눈 빈 캔버스, 쉰 목소리
- **영역**: Mirror Realm, Fallen Paradise에서 주로 등장
- **등장 효과**: 화면이 잠시 흑백으로 → 천둥 소리 → MITO 실루엣
- **참조**: MV에서 IAN 자신이 연기하되 deranged한 모습
- **인용**: "a living myth, an archangel that takes the form of a human"
- **색상**: 모노크롬(흑백) + 검은 깃털 + 빨간 눈 글로우(타락 후)

### Mr. Insanity (양극성의 고점 / 창조자)
- **비주얼**: 컬러풀, 예측불가, The Mask 에너지
- **영역**: The Nexus, The Other Side에서 등장
- **등장 효과**: 화면 색상이 과포화 → 글리치 → Mr. Insanity 등장
- **인용**: "the colorful, cheeky opposing force, never knowing where he's going"
- **색상**: 보라/초록/네온 — 과포화된 색감

### 소녀 (아발론의 거주자)
- **비주얼**: 따뜻한 톤, 빛 속의 실루엣
- **영역**: Fallen Paradise (Avalon 트랙 관련)
- **인용**: "To the girl, it's paradise, but to MITO, that was kind of like hell"

---

## 세계관 핵심 (리서치 기반)

### The Other Side
- 꿈꾸는 사람의 의식 속에만 존재하는 우주
- DPR = Dream Perfect Regime → 팬덤 = Dreamers → 꿈 속에 존재
- 앤트맨의 양자 영역처럼 우리 바로 밑에 존재하는 우주
- IAN은 기념품(souvenirs)을 가지고 돌아올 수 있는 루프홀 발견
- **오래 머물수록 미쳐감** → 이것이 Dear Insanity EP의 핵심

### MITO 캐릭터 상세
- 이름 출처: 라틴어 계열 언어에서 "myth(신화)"
- **한쪽 눈 = 비주얼 디렉터로서의 시선 + 세계를 다르게 보는 은유**
- 눈 모양 반지를 검지에 착용
- "a living myth, an archangel that takes the form of a human"
- 양극성 저점(manic lows)의 표현 — 천둥에 둘러싸인 외눈 타락 천사

### Mr. Insanity
- 양극성 고점(manic highs)의 표현
- "광기가 질서를 만든다"
- MITO를 포함한 모든 피조물의 창조자
- MITO의 질투를 감지하고 추방 → 비극의 시작

### 순환 구조 (뫼비우스)
- IAN → (The Other Side 탐험 중 미쳐감) → Mr. Insanity가 됨
- Mr. Insanity → (자신을 본뜬 피조물 창조) → MITO 탄생
- MITO → (질투/타락/추방) → 이야기가 결국 IAN의 시작으로 연결
- 시작과 끝이 하나 — Dear Insanity가 MIITO의 프리퀄

### 앨범 색상 시리즈 (IAN이 직접 확인)
1. ⬛ BLACK: Moodswings in This Order (어둠)
2. 🟥 RED: Moodswings in To Order (사랑/분노 전환점)
3. 🟪→⬜ WHITE: 미래 앨범 (빛으로의 귀환) — 역순으로 천사가 복원되는 구조

### 영화/문화 레퍼런스
- **이카루스** (그리스 신화) — 태양에 가까이 → 추락
- **세라핌** (성경) — 6날개 천사, 불타는 자, 가면=겸손
- **루시퍼** — 가면 벗음 = 타락
- **아발론** (아서왕) — 낙원이자 지옥의 이중성
- **캣츠 뮤지컬** — IAN의 어린시절 영감
- **팬텀 오브 오페라** — 가면 모티프
- **더 마스크 (짐 캐리)** — Mr. Insanity의 에너지
- **팀 버튼 시체 신부** — Don't Go Insane MV
- **피터팬/후크** — 잃어버린 어린시절 재발견

---

## 프로젝트 구조

```
mito-universe/
├── app/
│   ├── layout.tsx              # 전역 레이아웃 + 오디오 프로바이더
│   ├── page.tsx                # 인트로: 초록색 문 + 타이핑
│   ├── game/
│   │   ├── layout.tsx          # 게임 HUD + Bottom Nav + 앰비언트 BGM
│   │   └── [zoneId]/
│   │       └── page.tsx        # 영역 뷰
│   └── globals.css
├── components/
│   ├── intro/
│   │   ├── GreenDoor.tsx       # 🚪 초록색 문 (Three.js or CSS)
│   │   ├── TypewriterText.tsx  # 타이핑 애니메이션
│   │   └── IntroSequence.tsx   # 전체 인트로 오케스트레이션
│   ├── characters/
│   │   ├── MitoSilhouette.tsx  # 👁️ MITO 비주얼 (SVG/Spline)
│   │   ├── MrInsanity.tsx      # 🎭 Mr. Insanity 비주얼
│   │   └── CharacterReveal.tsx # 캐릭터 등장 애니메이션
│   ├── game/
│   │   ├── ZoneView.tsx        # 영역 메인 뷰 (배경+캐릭터+카드)
│   │   ├── ActionCards.tsx     # 2x2 카드 그리드
│   │   ├── BottomNav.tsx       # 하단 네비게이션
│   │   ├── TopBar.tsx          # 상단 HUD
│   │   └── ZoneBackground.tsx  # 영역별 배경 비주얼 (파티클+색감)
│   ├── sheets/
│   │   ├── BottomSheet.tsx
│   │   ├── LoreSheet.tsx
│   │   ├── TracksSheet.tsx     # + Spotify 연동 버튼
│   │   ├── NpcSheet.tsx        # + 캐릭터 실루엣 + 대화창
│   │   ├── TravelSheet.tsx
│   │   ├── MapSheet.tsx
│   │   └── TimelineSheet.tsx
│   ├── effects/
│   │   ├── Particles.tsx       # 캔버스 파티클 (모바일 최적화)
│   │   ├── Scanline.tsx        # CRT 스캔라인
│   │   ├── GlitchOverlay.tsx   # 글리치 효과 (MITO 등장시)
│   │   ├── ScreenShake.tsx     # 화면 흔들림 (천둥)
│   │   └── ColorShift.tsx      # 흑백 전환 (MITO) / 과포화 (Mr. Insanity)
│   └── audio/
│       ├── AudioProvider.tsx   # Context로 오디오 상태 관리
│       ├── AudioToggle.tsx     # 음소거 버튼
│       └── useAudio.ts         # 오디오 훅
├── data/
│   ├── zones.ts                # 영역 데이터 (로어/NPC/트랙/연결)
│   └── timeline.ts             # 타임라인
├── stores/
│   └── gameStore.ts            # Zustand: 진행도, 현재 영역, 설정
├── lib/
│   ├── audio.ts                # Howler.js 인스턴스 (BGM + SFX)
│   └── utils.ts
├── public/
│   ├── audio/
│   │   ├── nexus-ambient.mp3
│   │   ├── mirror-ambient.mp3
│   │   ├── fallen-ambient.mp3
│   │   ├── otherside-ambient.mp3
│   │   └── sfx/
│   │       ├── door-open.mp3
│   │       ├── chime.mp3
│   │       ├── whoosh.mp3
│   │       ├── thunder.mp3
│   │       └── string-note.mp3
│   ├── models/                 # 3D 에셋 (Spline/GLB)
│   │   └── green-door.glb
│   ├── images/
│   │   ├── mito-silhouette.svg
│   │   └── mr-insanity-silhouette.svg
│   ├── og-image.png
│   └── manifest.json           # PWA
├── CLAUDE.md
├── package.json
├── tailwind.config.ts
└── next.config.js
```

---

## 구현 우선순위

### Phase 1: 첫인상 (이 단계에서 "DPR IAN이다"를 알 수 있어야 함)
1. 인트로 시퀀스 — 초록색 문 + 타이핑 + 문 열기 애니메이션
2. MITO 실루엣 (SVG) — 한쪽 눈 글로우
3. 앰비언트 사운드스케이프 제작 (최소 4개 영역)
4. 기본 효과음 (문 열림, 영역 이동, 로어 발견)
5. 영역별 배경 색감 + 파티클

### Phase 2: 탐험의 깊이
6. 영역 뷰 + 바텀시트 UI
7. 로어 발견 시스템 (localStorage 저장)
8. NPC 대화 시스템 (타이핑 + 캐릭터 실루엣)
9. 트랙 리스트 + Spotify 연동
10. 유니버스 맵 + 타임라인
11. 영역 이동 트랜지션 (크로스페이드 BGM + 비주얼)

### Phase 3: 몰입 강화
12. Three.js/Spline 3D 초록색 문
13. 캐릭터 등장 효과 (흑백전환, 글리치, 화면 흔들림)
14. PWA (홈화면 설치)
15. 다국어 (한/영/일)
16. 팬 커뮤니티 기능

---

## 개발 명령어

```bash
npx create-next-app@latest mito-universe --typescript --tailwind --app --src-dir=false
cd mito-universe
npm install zustand framer-motion howler gsap
npm install -D @types/howler
npm run dev
```

---

## 프로토타입 참고
`mito-mobile-rpg.jsx` — 모바일 모바일 퍼스트 React 프로토타입.
데이터 구조와 UI 패턴 참고용. 비주얼/사운드는 이 CLAUDE.md의 스펙을 따를 것.

## 세계관 출처
- Grammy.com 인터뷰 (2023): The Other Side = 꿈 속 우주, DPR = Dreamers
- Rolling Stone AU (2022): MITO 캐릭터 상세, 눈 반지, 가면 의미
- Rolling Stone US (2022): 앨범 색상 시리즈 (흑→적→백), Mr. Insanity 기원
- Billboard (2021): MITO = 초능력으로 재정의된 양극성 장애
- Paste Magazine (2023): 영화 레퍼런스, 로자리오, 종교적 맥락
- Consequence (2022): 트랙별 세계관 분석

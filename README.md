https://dondog-pi.vercel.app/

#실제로 동작하게 만든 것 

Next.js 기반 동아리/학생회 회계 대시보드입니다. 실제 결제내역을 Supabase(Postgres)에 저장하고
OpenAI API로 분류하는 파이프라인과, 영수증을 업로드해 OCR로 판독·매칭하고 규칙 기반으로
이상거래를 감지하는 기능을 하나로 합쳤습니다. `/landing`에 별도 마케팅 랜딩 페이지가 있습니다.

장부 다운로드: /api/export가 실제 거래 데이터로 CSV를 만들어 내려줌 (엑셀에서 바로 열림)

대시보드 영수증 업로드: 파일을 실제로 OCR 분석 → Supabase에 저장 → 거래에 연결까지 진행, 새로고침해도 유지됨. "검토하기" 버튼은 실제 감사 페이지(/audit)로 이동

AI 챗봇: 실제 OpenAI API에 연결되어 동작합니다 (`/api/chat`)

AI 리포트: 신뢰도 97% 고정값 → 실제 분류 신뢰도 평균(현재 90%), 전월 대비 증감률 → 실제 월별 데이터 비교, "회칙 위반"과 "공동 승인 필요"가 같은 숫자였던 버그 수정(이제 각각 0건/2건으로 다름), 일정 예산(가짜) → 일정 전후 3일 실거래 지출액, AI 추천 3문장 고정 → 실제 예산 초과/공동승인/카테고리 증감 기반 추천

예산 관리: 카테고리별 예산에 입력창 + 저장 버튼 추가 (이미 있던 백엔드 함수에 연결만 하면 됐음)

#해야할 것

- 다른 그룹 관리
- 요금제
- gmail 로그인

`npm run dev` 실행 시 DB가 비어있으면 자동으로 시드됩니다.

## 결제 데이터 (Supabase)

결제 내역은 **코드가 아니라 Supabase DB**에 저장됩니다. `.env.local`에 `SUPABASE_URL`,
`SUPABASE_ANON_KEY`를 설정하고, `supabase/schema.sql`을 Supabase SQL Editor에서 한 번
실행해야 합니다.

| 경로 | 설명 |
|------|------|
| `supabase/schema.sql` | 전체 테이블 스키마 (payments, receipts, budget_* 등) |
| `data/payments.seed.json` | 결제 원본 데이터 (여기에 추가/수정) |

```bash
# payments.seed.json 수정 후 DB에 반영
npm run db:seed -- --force
```

- `data/payments.seed.json`에 `merchant`, `amount`, `transacted_at`만 넣으면 됩니다
- 잔액은 시드 시 자동 계산됩니다

## OpenAI 결제명 분류

```bash
# .env.local
OPENAI_API_KEY=sk-...

npm run classify
```

- 미분류 결제만 OpenAI(gpt-4o-mini)로 분류합니다
- 결과는 Supabase의 `payment_classifications` 테이블에 저장됩니다
- API 키가 없어도 개발 서버는 정상 동작하며, 미분류 결제는 "기타" 카테고리로 표시됩니다

## 영수증 업로드 · 이상거래 감지

- `/receipts`: 영수증 이미지를 업로드하면 실제 OCR(tesseract.js)로 파싱하고 기존
  거래내역과 매칭하거나, 매칭할 거래가 없으면 새 거래로 자동 등록합니다
- `/audit`: OpenAI로 분류된 결제내역을 승인/이월/공동승인 요청 워크플로우로 검토합니다
- `/audit/overview`: 예산초과·중복결제·일정불일치·고액지출·영수증누락 5가지 규칙 기반 감지 결과를
  전체 거래 테이블로 확인합니다 (`/audit`와 같은 규칙 엔진을 공유)

## 데이터 흐름

```
data/payments.seed.json
        ↓ npm run db:seed
Supabase payments 테이블
        ↓ npm run classify
Supabase payment_classifications 테이블
        ↓ 서버에서 읽기 (lib/get-dashboard-data.ts)
        ↓ 규칙 기반 이상거래 감지 (lib/anomalies/from-payments.ts)
대시보드 / 거래내역 / 감사 / 랜딩 UI
```

## 프로덕션 빌드

```bash
npm run build
npm start
```

## 참고

- 아이콘: lucide-react · 차트: chart.js + react-chartjs-2 · 랜딩 애니메이션: framer-motion
- 영수증 OCR·매칭 로직: `lib/receipts/*`, 이상거래 규칙 엔진: `lib/anomalies/*`
- 색상/라운드/그림자 값은 `tailwind.config.ts`의 `brand.*`, `rounded-card`,
  `rounded-btn`, `shadow-card` 토큰에 정의되어 있어 전체 화면에 일괄 반영됩니다

## 병합 내역 (`supabase` 브랜치 재병합)

`receipt+supabase+frontend` 브랜치에 `origin/supabase`의 새 커밋(실제 OpenAI 챗봇,
장부 CSV 다운로드 등)을 다시 가져오면서 충돌난 파일들을 정리한 기록입니다.

| 파일 | 살린 것 | 지운 것 |
|------|---------|---------|
| `README.md` | 양쪽 설명 다 병합 (Supabase 구조 설명 + 챗봇/CSV 다운로드 설명 + 해야할 것 목록) | `supabase` 브랜치가 새로 만든 소문자 `readme` 중복 파일 (Windows 파일명 충돌 위험) |
| `components/dashboard/ReceiptUploadModal.tsx` | (없음, 삭제 유지) | `supabase` 쪽이 개선한 팝업형 영수증 업로드 모달 전체 — 팝업 대신 `/receipts` 페이지로 이동하는 방식을 쓰기로 이미 결정했었기 때문 |
| `components/layout/Header.tsx` | 우리 쪽 버전 전체 (알림벨, 프로필 아바타, `useMockUser`/`useMockSession`, 검색 드롭다운) | `supabase` 쪽 버전 전체 — 우리가 이전에 프론트엔드 병합 때 이미 흡수한 것보다 더 단순한 구버전이라 가져올 내용 없었음 |
| `components/dashboard/Dashboard.tsx` | 우리 쪽 `isEmptyDashboard` 빈 대시보드 분기, `semester` 계산 | `supabase` 쪽 `handleReceiptUpload`(팝업 모달용 핸들러), 예전부터 안 쓰이던 `saveReceiptAction`/`parseReceipt` 죽은 import |
| `lib/get-dashboard-data.ts` | 둘 다 병합 — `supabase` 쪽 `categoryBudgets` fetch (아래 성능 수정에서 우리 쪽 `receipts` fetch는 다시 제거함) | 없음 (순수 병합) |
| `lib/dashboard-mock-data.ts` | 랜딩페이지 데모 데이터를 새 타입에 맞게 보강 (`totalMoM`, `trafficMoM`, `coApprovalRequired`, `recommendations` 추가) | 존재하지 않는 `AuditAnomaly.confidence` 필드 5곳 |

## 병합 이후 추가로 고친 버그

- **페이지 이동이 안 되던 버그**: `(main)/layout.tsx`에 걸려있던 `export const dynamic = "force-dynamic"`이
  원인이었습니다. Next.js App Router에서 공유 레이아웃에 이 옵션을 걸면, 같은 레이아웃을 쓰는
  페이지 간 클라이언트 사이드 이동(예: 대시보드 → `/receipts`)이 RSC 요청은 성공하지만 실제
  화면 전환이 커밋되지 않는 문제가 있었습니다. 레이아웃에서는 제거하고, 실시간 데이터가 필요한
  개별 `page.tsx` 11곳에 각각 `force-dynamic`을 옮겼습니다. 모든 변경 액션은 이미
  `revalidatePath`를 호출하므로 데이터 신선도는 그대로 유지됩니다.
- **영수증 데이터 중복/과다 조회**: `getDashboardData()`가 (main) 레이아웃을 통해 모든 페이지
  이동마다 `getReceipts()`를 호출해 영수증 이미지 전체(base64, 10MB+)를 매번 다시 받아오고
  있었는데, 실제로 이 값(`DashboardData.receipts`)은 어디서도 쓰이지 않는 죽은 필드였습니다.
  제거해서 페이지 이동/로딩이 빨라지고 Supabase 대역폭 사용량도 줄었습니다. `/receipts` 페이지는
  원래부터 자기 몫으로 `getReceipts()`를 따로 호출하고 있어 영향 없습니다.
- **예산 도넛 차트 비율 불일치**: `buildBudgetSlices()`가 검토 대기(미승인) 거래까지 포함해서
  계산되고 있어, 이미 검토 대기를 제외하고 계산하는 사용률/사용금액/잔액과 기준이 달랐습니다.
  도넛 차트도 동일하게 검토 대기 거래를 제외하도록 수정했습니다. 이상거래를 승인하면
  `router.refresh()`로 자동 재계산되어 비율도 즉시 반영됩니다.

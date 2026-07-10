# 데이터 수집 파이프라인 설계 (PIPELINE.md) — 구현은 이 문서대로 (Sonnet 이관용)

> 상태: 설계 확정안 (2026-07-10, Fable 5). RESEARCH.md §7과 함께 읽을 것.
> **원칙: 서버에는 참가자 코드(pid)만 — 학번·이름 저장 금지** (매칭표는 교사 오프라인 보관). 이 원칙이 IRB 서술을 단순화한다.

## 채택 아키텍처: Google Apps Script Web App + Google Sheets

선정 근거: 비용 0 / 데이터가 곧 스프레드시트라 비개발자 교사가 유지보수 가능 / script.google.com은 학교망에서 안정적 /
논문 부록 서술("정적 웹 도구 + 서버리스 스크립트 → 시트에 시행 단위 long format 자동 축적")이 간결.
탈락: Firebase(보안 규칙 오설정 위험·유지보수 부담), Supabase(무접속 1주 시 프로젝트 일시정지 — 학기 연구에 치명).

```
[학생 브라우저 — GitHub Pages, ?mode=research]
   ① 시행 로그 메모리 적재 → ② 종료 화면 [제출] (명시적 제출 = 동의 행위, 자동 전송 금지)
   ▼ POST text/plain (JSON)
[GAS Web App — 교사 계정 배포, 접근 '모든 사용자']
   ③ tok 검증 → LockService → v 확인
   ├▶ 시트 raw_trials (시행 1건=1행, long format)
   └▶ 시트 sessions   (세션 요약 1행)
[교사] Sheets → CSV → jamovi/SPSS/R
실패 사다리: 재시도 1회 → localStorage 보관+재제출 버튼 → 클립보드 수기 제출(기존 기능)
```

## API 계약

- 엔드포인트: `POST https://script.google.com/macros/s/{배포ID}/exec`
- **요청 헤더: `Content-Type: text/plain;charset=utf-8` 고정** — application/json은 preflight OPTIONS를 유발하고 GAS가 처리 못 함. 본문은 JSON 문자열, 서버에서 `JSON.parse(e.postData.contents)`.
- 요청 본문:

```json
{
  "v": 1,
  "tok": "연구기간_공유토큰",
  "pid": "A03-17",
  "grp": "poke|gami|plain",
  "set": 2,
  "ts": "ISO8601+09:00",
  "trials": [
    { "r": 1, "d": "ee-triage", "axis": "EE", "cond": "H1-cute",
      "monL": 25, "monR": 133, "tierL": 2, "tierR": 2,
      "pick": "L|R", "pole": 0, "rt": 4821 }
  ],
  "summary": { "EE": [1, 2], "SI": [2, 0], "DS": [1, 2] }
}
```

- 응답 (GAS는 항상 HTTP 200 — 본문으로 판정):
  - 성공: `{ "ok": true, "sid": "S0042" }`
  - 실패: `{ "ok": false, "err": "bad-token" | "bad-schema" | "lock-timeout" }`

## 서버 처리 규칙 (doPost 설계)

1. `tok` 불일치 → 저장 없이 `bad-token` (토큰은 연구 기간 고정, 유출 시 재배포).
2. `v` 미지원 → `bad-schema` (스키마 진화 대비).
3. **재제출 허용**: 같은 pid가 다시 제출해도 거절하지 않고 전부 저장, sessions에 회차(n번째) 표기 — 파이프라인은 데이터를 버리지 않는다, 취사선택은 연구자 몫.
4. `LockService.getScriptLock()` 10초 — 동시 제출 시 행 유실 방지.
5. raw_trials 열: sid, pid, grp, set, ts, r, d, axis, cond, monL, monR, tierL, tierR, pick, pole, rt.
   sessions 열: sid, pid, grp, set, ts, 회차, EE_l, EE_r, SI_l, SI_r, DS_l, DS_r, trials수, ua(선택).

## 클라이언트 처리 규칙

- fetch 타임아웃 8초(AbortSignal.timeout). `ok:true` → 접수번호(sid) 표시 + "참여 완료" 화면.
- 실패 → 1회 자동 재시도 → 실패 시 payload를 `localStorage['kachu_pending']`에 보관하고 "다시 제출" 버튼 노출.
- 최후 폴백 = 기존 클립보드 복사(수기 제출) — 이미 구현되어 있음.
- **무결성 규칙: 제출 성공(또는 수기 폴백 안내) 전에는 "참여 완료"를 띄우지 않는다.**
- 엔드포인트 URL·tok은 research-config.js에 상수로 (수업 모드 코드 경로와 분리).

## 교사 운영 절차 (논문 부록 "데이터 수집 절차" 초안 겸용)

1. 교사 구글 계정에서 시트 생성 → 확장 프로그램 > Apps Script → doPost 스크립트 부착 → 웹 앱 배포(액세스: 모든 사용자).
2. 배포 URL과 토큰을 research-config.js에 기입 → GitHub Pages 재배포.
3. 수업에서 참가자 코드 배부(매칭표는 오프라인 보관) → 학생 참여 → 시트에 자동 축적.
4. 수집 종료 후: 시트 사본 → CSV → 분석. 연구 종료 시 원본 시트 보존 기한·파기 계획 명시.

## 검증 체크리스트 (구현 후)

- [ ] text/plain POST가 preflight 없이 성공 (Network 탭에 OPTIONS 없음)
- [ ] 잘못된 tok → 시트에 행이 늘지 않음
- [ ] 동시 제출 5건 → 행 유실 0
- [ ] 네트워크 차단 상태 제출 → localStorage 보관 → 복구 후 재제출 성공
- [ ] raw_trials 행 수 = Σ trials, sessions 행 수 = 제출 수

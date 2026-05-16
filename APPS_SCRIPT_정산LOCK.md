# 정산 동시 쓰기 Lock — Apps Script 가이드 (안전망 3단계)

여러 PC에서 **동시에 정산 데이터를 시트에 저장**할 때 발생하는 race condition을 차단합니다.
12월 주류 누락 사고처럼 "마지막 쓰기만 남고 앞 쓰기가 사라지는" 상황을 막는 핵심 안전망입니다.

> **이 단계는 Apps Script에 직접 코드를 적용해야 효과가 있어요. ERP 코드만 수정해서는 막을 수 없습니다.**
> 적용 안 해도 ERP는 정상 작동하지만, 동시 저장 충돌 위험은 남습니다.

---

## 📋 절차 요약

1. 구글시트 → 확장 프로그램 → Apps Script
2. 기존 `doPost(e)` 함수 **맨 위에** Lock 블록 추가 (아래 1단계)
3. 저장 + 새 배포 / 배포 업데이트
4. 끝

---

## 1️⃣ doPost 맨 위에 Lock 블록 추가

기존 `doPost(e)` 함수의 첫 줄에 아래 코드를 그대로 붙여넣어 주세요.
**기존 처리 로직은 건드리지 말고, Lock으로 감싸기만 합니다.**

### ❌ Before (현재 코드 예시)

```javascript
function doPost(e) {
  const data = JSON.parse(e.postData.contents);
  const action = data.action;
  if(action === 'saveSettlement') return saveSettlementHandler(data);
  if(action === 'saveDrivers') return saveDriversHandler(data);
  // ... 다른 분기들 ...
}
```

### ✅ After (Lock 적용)

```javascript
function doPost(e) {
  // ─── 동시 쓰기 Lock (정산 안전망 3단계) ───
  // 정산 관련 action만 Lock으로 보호. 그 외 action은 우회.
  const data = JSON.parse(e.postData.contents);
  const action = data.action;
  const LOCKED_ACTIONS = [
    'saveSettlement',
    'saveSettlementBackup',
    'deleteSettlement',
    'restoreSettlement'
    // 정산 데이터를 시트에 쓰는 action 이름을 여기에 추가
    // (기존 Apps Script에 있는 정확한 이름으로!)
  ];

  if(LOCKED_ACTIONS.indexOf(action) !== -1) {
    const lock = LockService.getScriptLock();
    let acquired = false;
    try {
      acquired = lock.tryLock(30000); // 30초 대기
      if(!acquired) {
        return ContentService
          .createTextOutput(JSON.stringify({
            success: false,
            locked: true,
            message: '다른 PC가 같은 작업 중입니다. 잠시 후 자동 재시도됩니다.'
          }))
          .setMimeType(ContentService.MimeType.JSON);
      }
      return _doPostInner(action, data);
    } catch(err) {
      return ContentService
        .createTextOutput(JSON.stringify({ success: false, message: err.message }))
        .setMimeType(ContentService.MimeType.JSON);
    } finally {
      if(acquired) try { lock.releaseLock(); } catch(_) {}
    }
  }

  // Lock 불필요한 action은 그대로 실행
  return _doPostInner(action, data);
}

// 기존 doPost 안의 분기 처리를 이 함수로 옮겨주세요
function _doPostInner(action, data) {
  if(action === 'saveSettlement') return saveSettlementHandler(data);
  if(action === 'saveDrivers') return saveDriversHandler(data);
  if(action === 'saveReceiptCheck') return saveReceiptCheckHandler(data);
  if(action === 'logDownload') return logDownload_(data.log);
  // ... 기존 분기 전부 여기로 이동 ...
}
```

---

## 2️⃣ 핵심 포인트 정리

| 항목 | 값 | 이유 |
|------|------|------|
| 대기 시간 | `tryLock(30000)` = 30초 | Apps Script 최대 실행시간 6분의 8% — 충분히 여유 |
| 보호 범위 | 정산 관련 action만 | 인수증/차량 등 다른 시트는 충돌 무관, Lock 부담 X |
| 실패 응답 | `{success:false, locked:true}` | 클라이언트가 친절히 안내 / 자동 재시도 |
| Lock 해제 | `finally` 블록 | 에러 발생해도 반드시 해제 |

---

## 3️⃣ LOCKED_ACTIONS 목록 확인 방법

자기 Apps Script 코드 열어서 `function doPost` 안의 `if(action === '...')` 분기들 중 **정산 시트(시트 이름이 정산/settlement 비슷)에 쓰는 action 이름**을 모두 `LOCKED_ACTIONS` 배열에 추가해 주세요.

예시:
- ✅ `saveSettlement` → 추가
- ✅ `saveSettlementBackup` → 추가
- ✅ `deleteSettlement` → 추가
- ❌ `saveReceiptCheck` (인수증) → 추가 X (다른 시트)
- ❌ `saveDrivers` (차량등록) → 추가 X (다른 시트)
- ❌ `logDownload` (로그) → 추가 X (append만 함, 충돌 무관)

---

## 4️⃣ 저장 + 배포

1. Ctrl+S
2. 우상단 **배포 → 배포 관리 → 편집(연필)** → **새 버전** → **배포**

---

## ✅ 동작 확인

1. PC A에서 정산 시트 백업 버튼 누름 (30초 정도 걸리는 무거운 작업)
2. PC B에서도 동시에 시트 백업 시도
3. PC B는 잠깐 기다린 후 자동 진행 → 두 PC 데이터 모두 안전하게 반영됨

**Lock 적용 전**: PC A의 결과를 PC B가 덮어써서 사라짐 ⚠️
**Lock 적용 후**: PC A 완료 후 PC B가 최신 상태에서 작업 → 모두 반영 ✅

---

## 🛡️ 안전 장치

- Lock 획득 실패해도 `{success:false, locked:true}` 응답 반환 — 데이터 손실 X
- ERP 측 클라이언트는 `locked:true` 받으면 친절한 토스트 + 1회 자동 재시도
- Apps Script 미적용 시에도 ERP는 정상 작동 (Lock 없이 그냥 호출됨)

# 인수증 체크 실시간 공유 — 시트 측 Apps Script 코드 추가 가이드

다른 PC와 인수증 체크를 실시간 공유하려면 **구글 시트의 Apps Script 코드**에 아래 코드를 한 번만 추가해야 합니다. (정산 자료 영향 0)

## 📋 절차

1. 구글 시트 열기
2. 메뉴: **확장 프로그램 → Apps Script**
3. 기존 코드(doGet, doPost 등이 있는 곳)에 아래 두 함수 추가
4. `doGet` 함수 안과 `doPost` 함수 안에 새 case 한 줄씩 추가
5. **저장** + **새 배포** 또는 **기존 배포 업데이트**

---

## 1️⃣ 새 함수 2개 추가 (파일 끝에)

```javascript
// ─── 인수증 체크 시트 동기화 (정산 시트와 별개) ───
function _ensureReceiptSheet() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sh = ss.getSheetByName('인수증체크');
  if(!sh) {
    sh = ss.insertSheet('인수증체크');
    sh.appendRow(['client','date','code','received','scanned','memo','manual','updatedBy','updatedAt']);
    sh.setFrozenRows(1);
  }
  return sh;
}

function saveReceiptCheckHandler(payload) {
  try {
    const sh = _ensureReceiptSheet();
    const records = (payload && Array.isArray(payload.records)) ? payload.records : [];
    sh.clearContents();
    sh.appendRow(['client','date','code','received','scanned','memo','manual','updatedBy','updatedAt']);
    if(records.length > 0) {
      const rows = records.map(r => [
        r.client||'', r.date||'', r.code||'',
        r.received||'', r.scanned||'', r.memo||'',
        r.manual ? 'Y' : '',
        r.updatedBy||'', r.updatedAt||''
      ]);
      sh.getRange(2, 1, rows.length, 9).setValues(rows);
    }
    return ContentService
      .createTextOutput(JSON.stringify({ success: true, count: records.length }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch(e) {
    return ContentService
      .createTextOutput(JSON.stringify({ success: false, message: e.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function getReceiptCheckHandler() {
  try {
    const sh = _ensureReceiptSheet();
    const data = sh.getDataRange().getValues();
    const records = [];
    for(let i = 1; i < data.length; i++) {
      const r = data[i];
      if(!r[0] && !r[2]) continue;
      records.push({
        client: r[0] || '',
        date: String(r[1] || ''),
        code: r[2] || '',
        received: r[3] || '',
        scanned: r[4] || '',
        memo: r[5] || '',
        manual: r[6] === 'Y' || r[6] === true,
        updatedBy: r[7] || '',
        updatedAt: String(r[8] || '')
      });
    }
    return ContentService
      .createTextOutput(JSON.stringify({ success: true, records: records }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch(e) {
    return ContentService
      .createTextOutput(JSON.stringify({ success: false, message: e.message, records: [] }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

---

## 2️⃣ doPost에 case 1줄 추가

기존 `doPost(e)` 함수 안의 switch/if 분기에 추가:

```javascript
// 기존 if/case 분기 안에:
if(action === 'saveReceiptCheck') return saveReceiptCheckHandler(payload);
```

(다른 액션과 비슷한 위치에 — 보통 `if(action === 'saveDispatch') ...` 같은 분기 옆)

---

## 3️⃣ doGet에 case 1줄 추가

기존 `doGet(e)` 함수 안의 분기에 추가:

```javascript
// 기존 if/case 분기 안에:
if(action === 'getReceiptCheck') return getReceiptCheckHandler();
```

(보통 `if(action === 'getAll') ...` 같은 분기 옆)

---

## 4️⃣ 저장 + 배포

1. **저장** (Ctrl+S)
2. 우상단 **배포 → 배포 관리 → 편집(연필)** → **새 버전** → **배포**
3. (또는 처음이면) **배포 → 새 배포**

---

## ✅ 확인

- ERP에서 인수증 체크 → 셀 클릭(O/X) → 3초 후 시트의 `인수증체크` 시트에 자동 반영됨
- 다른 PC에서 ERP 열면 5초 이내 자동 동기화 (autoSync)
- 시트 측 코드 추가 안 해도 ERP는 정상 작동 (조용히 실패)

## 🛡️ 정산 자료 영향 0

- 별도 시트(`인수증체크`)에 저장 — 기존 정산 시트 절대 안 건드림
- ERP 측에서도 settlementData 안 건드림

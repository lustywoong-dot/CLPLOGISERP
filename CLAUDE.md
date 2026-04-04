# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

CLPLOGISERP is a Korean logistics dispatch management system ("CLP 배차 관리 시스템") for managing delivery orders, driver assignments, rate calculations, and settlement for Lock&Lock and KCTC operations. The entire application is a **single `index.html` file (~11,600 lines)** with all CSS, HTML, and JavaScript inline.

## Running the Application

No build tools, package manager, or server required. Open `index.html` in a modern browser. Default login: `admin` / `admin1234`.

Optional: Configure a Google Sheets Apps Script URL in the Settings tab for cloud sync.

## Architecture

### Single-File SPA

Everything lives in `index.html`: styles in `<style>`, markup in the body, and all logic in a single `<script>` block. There is no module system, bundler, or framework — it's vanilla JS with CDN-loaded libraries (xlsx, exceljs, Chart.js).

### Tab-Based Modules (8 tabs)

1. **배차 등록** — Dispatch registration with paste-parsing (Excel/memo/bulk), auto-rate calculation, multi-driver assignment, liquor waypoint stops
2. **배차 이력** — Searchable/filterable dispatch history with inline editing
3. **단가표 조회** — Rate tables by region, vehicle, and delivery type
4. **차량 등록** — Driver and vehicle database
5. **기사 정산** — Driver settlement calculations, SMS templates, Google Sheets backup
6. **통계** — Revenue/cost/margin charts (Chart.js), driver rankings, period grouping
7. **출퇴근부** — Attendance tracking
8. **설정** — Google Sheets API config, user management

### Data Layer

- **IndexedDB** — Primary persistent storage for dispatch data
- **localStorage** — 90-day cache, user accounts (`clp_users`), settings (`clp_sheet_url`)
- **sessionStorage** — Login session (`clp_session`)
- **Google Sheets** — Optional bi-directional cloud sync via Apps Script web app (auto-sync every 5–30s)

### Key Global State

- `dispatchLog` — Main dispatch records array
- `settlementData` — Settlement information
- `driverDB_registered` — Driver/vehicle database
- Rate constants: `RATES`, `LIQUOR_RATES_DP`, `LIQUOR_RATES_YJ`, `LIQUOR_WAYPOINT_FEE`, `LIQUOR_WORK_FEE`, `MJIM_RATES`, `MANUAL_FEE`

### Dispatch Record Schema

Records use fields: `id`, `code`, `clientName`, `date`, `deliveryDate`, `type`, `pickup`, `delivery`, `deliveryAddr`, `deliveryContact`, `deliveryPhone`, `vehicle`, `pallet`, `boxes`, `baseRate`, `manualFee`, `waitFee`, `roundFee`, `total`, `driverPay`, `margin`, `dispatchRoute`, `driverName`, `driverPhone`, `vehicleNo`, `arrivalTime`, `memo`, `stickyMemo`, `extraWorkFee`, `liquorStops`, `createdBy`.

### Access Control

Role-based: `admin`, `dispatch`, `settlement`, `viewer`. Each role has different tab/action permissions.

## Key Patterns to Know

- **All UI is Korean** — labels, alerts, placeholders, and data values are in Korean
- **Rate auto-calculation** — selecting a region + vehicle type auto-fills pricing from the rate constant tables
- **Multi-driver support** — a single order can be split across drivers using sub-codes
- **Liquor deliveries** — special handling with waypoint stops and separate rate tables for Deokpyeong (DP) vs Yeoju (YJ)
- **Inline editing** — dispatch history rows are editable in-place; changes sync locally then push to Google Sheets
- **No tests or linting** — there is no test framework, linter, or CI/CD configured
## CLP ERP 개발 규칙

### 절대 변경 금지
- 📸 백업 저장 버튼 + saveSettlementBackupToSheet 함수
- raw: false (XLSX 읽기) — raw:true 변경 시 금액/비고 유실
- sanitizeDate 핵심 로직 — toISOString() 사용 절대 금지
- autoSync 5초 간격
- dispatchLog.sort 직접 호출 금지
- sortedFiltered = [...filtered] 복사본 방식 유지

### 주의사항
- 날짜는 반드시 getFullYear() 방식 사용
- 코드 교체 시 ROLE_TABS, DEFAULT_USERS 등 인접 상수 삭제 주의
- forEach 블록 교체 시 닫기 코드 확인 필수
- async 함수 변경 시 호출부 await 누락 주의
- 기본 계정: admin/admin1234, dispatch/1234

### 기술 스택
- 단일 파일: index.html (HTML/CSS/JS 전체 포함)
- 데이터: IndexedDB(전체) + localStorage(90일 캐시) + 구글시트(공유)
- 배포: GitHub Pages → git push 하면 자동 배포

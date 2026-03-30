# 데이터 추출 서비스 워크플로우

Compass Engine의 경영성과 데이터 추출 구조와 흐름을 설명하는 문서.

---

## 개요

Compass Engine은 **5개 업종**의 경영성과 데이터를 **XBRL API-first** 아키텍처로 추출한다.

| 업종 | 대상 | 오케스트레이터 | 주요 소스 |
|------|------|---------------|----------|
| 금융지주 | 4개사 | `IntegratedOrchestrator` | XBRL API → DART XML → Factbook |
| 은행 | 4개사 | `IntegratedOrchestrator` | XBRL API → DART XML → Factbook |
| 증권 | 26개사 | `Orchestrator` | XBRL API → DART XML (규제지표) |
| 카드 | 8개사 | `Orchestrator` | XBRL API → DART XML (규제지표) |
| 일반업종 | 774개사 | `Orchestrator` | XBRL API → DART APIs → DART XML |

### 데이터 소스 우선순위

| 순위 | 소스 | 용도 | source 값 |
|------|------|------|-----------|
| 1 | OpenDART XBRL API | IS/BS/CF 핵심 지표 (메인) | `dart` |
| 2 | DART 공시 XML | 규제지표 (NCR, 연체율 등), D&A 주석 | `dart` |
| 3 | Factbook/Databook | 잠정치, API 부재 지표 (BIS, NIM 등) | `factbook` |
| 4 | FSS/CREFIA | 과거 데이터, 카드 이용실적 | `fss` / `crefia` |
| 5 | FSC API | 분기말 종가, 시가총액 | `fsc_api` |

### 이중 구조: 분기말 vs 실시간

| 구분 | 저장 방식 | 용도 | 데이터 출처 |
|------|----------|------|------------|
| **분기말 데이터** | DB 저장 (performance_values) | 분기별 비교, 랭킹, 필터링 | 추출 파이프라인 + 분기말 주가 |
| **실시간 데이터** | 조회 시 계산 (저장 안함) | 대시보드 현재가 표시 | 최근 거래일 주가 + 저장된 EPS/BPS |

---

## 서비스 구조

```
app/services/
├── financial_institutions/
│   ├── holdings/                  # 금융지주 (4개사)
│   │   ├── integrated_orchestrator.rb  # API-first 통합 추출
│   │   ├── orchestrator.rb             # Legacy (Factbook-only)
│   │   ├── xbrl_api_extractor.rb       # XBRL API 추출
│   │   ├── factbook_extractor.rb       # Factbook Excel 추출
│   │   ├── dart_extractor.rb           # DART XML 추출
│   │   ├── persister.rb                # DB 저장
│   │   ├── profitability_calculator.rb # ROE/ROA 계산
│   │   ├── kb.rb, shinhan.rb, hana.rb, woori.rb  # 회사별 설정
│   │   └── base.rb                     # 공통 기반
│   │
│   ├── banks/                     # 은행 (4개사)
│   │   ├── integrated_orchestrator.rb
│   │   ├── xbrl_api_extractor.rb
│   │   ├── factbook_extractor.rb
│   │   ├── dart_extractor.rb
│   │   ├── persister.rb
│   │   ├── profitability_calculator.rb
│   │   ├── productivity_calculator.rb
│   │   └── kb.rb, shinhan.rb, hana.rb, woori.rb, base.rb
│   │
│   ├── securities/                # 증권사 (26개사)
│   │   ├── orchestrator.rb             # Dual-phase 추출
│   │   ├── xbrl_api_extractor.rb
│   │   ├── dart_extractor.rb
│   │   ├── persister.rb
│   │   ├── derived_calculator.rb
│   │   └── market_share_calculator.rb
│   │
│   ├── cards/                     # 카드사 (8개사)
│   │   ├── orchestrator.rb             # Dual-phase + 생산성
│   │   ├── xbrl_api_extractor.rb
│   │   ├── dart_extractor.rb
│   │   ├── crefia_extractor.rb         # 여신금융협회 데이터
│   │   ├── persister.rb
│   │   └── productivity_calculator.rb
│   │
│   └── shared/                    # 공통 모듈 (13 files)
│       ├── xbrl_api_base.rb            # XBRL API 공통 로직
│       ├── q4_calculator.rb            # Q4 분기값 산출 (YTD 차감)
│       ├── orchestrator_helpers.rb     # 직원수, 분기보고서 조회
│       ├── profitability_calculator.rb # ROE/ROA 공통 계산
│       ├── derived_ratio_calculator.rb # 파생 비율 계산
│       ├── source_registry.rb          # 데이터 소스 등록
│       ├── xml_utils.rb, table_utils.rb, value_parser.rb  # 파싱 유틸
│       ├── company_mappings.rb         # 회사명/코드 매핑
│       └── factbook_downloader.rb      # Factbook 자동 다운로드
│
├── general_industry/              # 일반업종 (774개사)
│   ├── orchestrator.rb                 # Multi-source 7단계 파이프라인
│   ├── xbrl_api_extractor.rb           # XBRL API (16개 지표 + CAPEX fallback)
│   ├── bulk_txt_extractor.rb           # TXT 벌크 추출 (과거 백필용)
│   ├── dart_extractor.rb              # DART XML (D&A 주석 전용)
│   ├── persister.rb
│   └── derived_calculator.rb           # 산출 지표 (20개)
│
├── dart/                          # DART 전용 서비스 (26 files)
│   ├── client.rb                       # DART API 클라이언트
│   ├── search_api.rb                   # 공시 검색 API
│   ├── employee_api.rb                 # 직원수 API
│   ├── dividend_api.rb                 # 배당 API
│   ├── shares_api.rb                   # 주식총수 API
│   ├── base/                           # 추출 프레임워크
│   │   ├── extractor.rb, processor.rb
│   │   ├── validator.rb, retrier.rb
│   │   └── logger.rb, run_tracker.rb
│   └── bond/                           # 회사채 추출 (10 files)
│       ├── processor.rb
│       └── extractors/                 # 수요예측, 발행확정 등
│
├── stock/                         # 주가 및 시장 지표
│   ├── fsc_client.rb                   # 금융위원회 주가 API
│   ├── daily_price_updater.rb          # 전일 종가 일괄 업데이트
│   ├── market_value_calculator.rb      # PER/PBR/TSR 계산
│   ├── market_indicator_persister.rb   # 시장 지표 저장
│   └── realtime_indicator_service.rb   # 실시간 지표 조회
│
├── monitoring/
│   └── coverage_calculator.rb          # 커버리지 모니터링
│
└── retirement_pension/
    └── fss_extractor.rb                # 퇴직연금 FSS 데이터

app/jobs/
├── dart_monitor_job.rb                 # DART 공시 모니터링 (매시간)
├── factbook_monitor_job.rb             # Factbook 모니터링 (매일 9시)
├── dart/
│   ├── holdings_extraction_job.rb      # 금융지주 추출
│   ├── banks_extraction_job.rb         # 은행 추출
│   ├── securities_extraction_job.rb    # 증권 추출
│   ├── cards_extraction_job.rb         # 카드 추출
│   └── bond/
│       ├── extract_job.rb              # 회사채 추출
│       ├── auto_monitoring_job.rb      # 자동 모니터링
│       └── retry_job.rb               # 재시도
└── stock/
    └── daily_price_update_job.rb       # 매일 18시 주가 업데이트
```

---

## 업종별 추출 파이프라인

### Pattern 1: API-first with Fallback (금융지주 / 은행)

XBRL API를 주 소스로, DART XML과 Factbook을 보조 소스로 사용.

```
┌──────────────────────────────────────────────────────┐
│          IntegratedOrchestrator                       │
├──────────────────────────────────────────────────────┤
│                                                       │
│  Step 1: XBRL API                                     │
│  ├── IS: 순이자이익, 영업이익, 당기순이익              │
│  └── BS: 총자산, 자본총계, 지배주주자본                │
│                                                       │
│  Step 2: DART XML                                     │
│  ├── 규제지표: NPL, RWA, 상장주식수                   │
│  └── 은행: 원화예수금, 원화대출금                      │
│                                                       │
│  Step 3: Factbook (API 부재 지표만)                    │
│  ├── BIS비율, Tier1, CET1                             │
│  └── NIM, DPS                                         │
│                                                       │
│  Step 4~8: Calculation                                │
│  ├── EPS/BPS → ROE/ROA (ProfitabilityCalculator)     │
│  ├── Stock API → 분기말 주가, 시가총액                 │
│  └── PER/PBR/TSR (MarketValueCalculator)             │
│                                                       │
│  ──────▶ Persister ──────▶ DB                         │
└──────────────────────────────────────────────────────┘
```

```ruby
# 단일 기업 추출
orchestrator = FinancialInstitutions::Holdings::IntegratedOrchestrator.new(
  company_class: FinancialInstitutions::Holdings::Kb,
  year: 2025, quarter: 4
)
result = orchestrator.extract
orchestrator.extract_and_persist!  # 추출 + DB 저장

# 4대 금융지주 일괄 추출
FinancialInstitutions::Holdings::IntegratedOrchestrator.extract_all(
  year: 2025, quarter: 4,
  factbook_paths: { kb: "/path/to/kb.xlsx", shinhan: "/path/to/shinhan.xlsx" }
)
```

### Pattern 2: Dual-phase (증권사 / 카드사)

XBRL API로 재무제표 지표를, DART XML로 규제지표를 별도 추출한 뒤 합산.

```
┌──────────────────────────────────────────────────────┐
│          Securities/Cards Orchestrator                 │
├──────────────────────────────────────────────────────┤
│                                                       │
│  Phase 1: XBRL API → 재무제표 핵심 지표               │
│                                                       │
│  Phase 2: DART XML → 규제지표                         │
│  ├── 증권: 수탁수수료, NCR, 영업용순자본              │
│  └── 카드: 연체율, 조정자기자본비율                    │
│                                                       │
│  Phase 3: DB 저장 (Persister)                         │
│                                                       │
│  Phase 4: 산출 지표 (DerivedCalculator)               │
│  ├── CIR, ROA, ROE, 레버리지비율                      │
│  └── 생산성 지표 (카드: 1인당 영업이익)               │
│                                                       │
│  Phase 5: 시장점유율 (전체 추출 시만)                  │
│                                                       │
└──────────────────────────────────────────────────────┘
```

```ruby
# 단일 기업 추출
orchestrator = FinancialInstitutions::Securities::Orchestrator.new(year: 2025, quarter: 4)
orchestrator.extract_one(dart_code: "00126380")

# 전체 추출 (시장점유율 포함)
orchestrator.extract_all(force: false, on_progress: ->(msg) { puts msg })
```

### Pattern 3: Multi-source Pipeline (일반업종)

7개 데이터 소스를 순차적으로 호출하는 파이프라인.

```
┌──────────────────────────────────────────────────────┐
│          GeneralIndustry::Orchestrator                 │
├──────────────────────────────────────────────────────┤
│                                                       │
│  Step 1: XBRL API → 16개 원천 지표 (+ CAPEX fallback)│
│  Step 2: EmployeeApi → 직원수 (Q2/Q4, fallback)      │
│  Step 3: DividendApi → 배당성향, DPS                  │
│  Step 4: FscClient → 분기말 주가, 시가총액             │
│  Step 5: SharesApi → 유통주식수                       │
│  Step 6: DartExtractor → D&A (감가상각비, XML 주석)   │
│  Step 7: Persister → DB 저장                          │
│  Step 8: DerivedCalculator → 20개 산출 지표            │
│                                                       │
└──────────────────────────────────────────────────────┘
```

```ruby
# 단일 기업 추출
orchestrator = GeneralIndustry::Orchestrator.new(
  company: Company.find_by!(dart_code: "00126380"),
  year: 2024, quarter: 4
)
result = orchestrator.extract(save: true)

# 배치 추출
GeneralIndustry::Orchestrator.extract_batch(year: 2024, quarter: 4, save: true)

# TXT 벌크 추출 (과거 데이터 백필)
GeneralIndustry::Orchestrator.extract_batch_from_txt(year: 2023, quarter: 3)

# 다분기 백필
GeneralIndustry::Orchestrator.backfill(
  start_year: 2021, end_year: 2024,
  start_quarter: 1, end_quarter: 4
)
```

### Pattern 4: Bond Processor (회사채)

DART 공시문서에서 채권 발행/인수 정보를 추출하는 독립 파이프라인.

```
Dart::SearchApi (공시 검색) → Dart::Bond::Processor
  → Extractors (수요예측, 발행확정, 금리, 발행비용)
  → Dart::Bond::Persister → DB (dcm_* 테이블)
```

---

## 공통 모듈

### shared/xbrl_api_base.rb

모든 업종의 XBRL API 추출기가 상속하는 기반 클래스. CFS→OFS fallback, Q/YTD 구분, 단위 변환 로직 제공.

### shared/q4_calculator.rb

Q4 분기값 산출: `Q4 = YTD(연간) - Q1 - Q2 - Q3`. 금융업종 YTD-Only 변환에 사용.

### shared/orchestrator_helpers.rb

공통 헬퍼: 직원수 조회(`fetch_employee_count`), 분기보고서 검색(`find_quarterly_report`), 구조화된 로깅.

### shared/source_registry.rb

데이터 소스 등록 및 추적. 각 지표가 어떤 소스에서 왔는지 metadata에 기록.

---

## 분기말 주가 및 시장 지표

### Stock::FscClient

금융위원회 API를 통해 분기말 종가, 시가총액, 상장주식수를 조회.

```ruby
client = Stock::FscClient.new
client.get_latest_stock_price("105560", "20250930")
# => { close_price: 89700, date: "20250930", listed_shares: 408932378, market_cap: 49860000000000 }
```

### Stock::MarketValueCalculator

EPS/BPS/DPS와 분기말 주가를 조합하여 시장가치 지표를 계산.

```ruby
calculator = Stock::MarketValueCalculator.new
calculator.calculate_all(
  stock_price: 89700, eps: 10489, bps: 122345,
  dps: 3060, start_price: 85700, shares_outstanding: 408932378
)
# => { per: 8.55, pbr: 0.73, tsr: 8.23, dividend_yield: 3.41 }
```

### Stock::DailyPriceUpdater

상장 종목의 전일 종가를 companies 테이블에 저장. 실시간 PER/PBR 계산에 활용.

```ruby
updater = Stock::DailyPriceUpdater.new
updater.update_all_companies
```

---

## 실시간 데이터 조회

### RealtimeIndicatorService

대시보드에서 현재 주가 기반 지표 표시 시 사용. DB에 저장하지 않고 조회 시 계산.

```ruby
service = Stock::RealtimeIndicatorService.new
result = service.get_indicators(stock_code: "105560")
# => { stock_price: 89700, per: 8.55, pbr: 0.73, tsr_qtd: 4.61, tsr_ytd: 42.5, ... }

# 금융지주 일괄 조회
results = service.get_holdings_indicators
```

#### 계산 로직

```
실시간 PER = 최근 거래일 종가 / 저장된 EPS (최근 분기)
실시간 PBR = 최근 거래일 종가 / 저장된 BPS (최근 분기)
TSR(QTD) = (현재가 - 분기초가 + 배당) / 분기초가 × 100
TSR(YTD) = (현재가 - 연초가 + 배당) / 연초가 × 100
```

---

## Persister (DB 영속화)

각 업종별 `Persister` 클래스가 오케스트레이터 결과를 `performance_values` 테이블에 저장.

### 저장 규칙

| 항목 | 규칙 |
|------|------|
| 단위 | 모든 금액은 **원(₩) 단위** |
| basis | `consolidated` (연결, 기본) / `separate` (별도) |
| period_type | `q` (분기), `ytd` (누적), `pit` (시점) |
| source 추적 | `metadata` JSONB에 `{ source: "dart" }` 형태로 기록 |
| 유니크 제약 | `(company_id, indicator_id, year, quarter, period_type, basis)` |

### upsert 전략

기존 값이 있으면 덮어쓰기 (확정치가 잠정치를 대체).

---

## 스케줄링 (Solid Queue)

```yaml
# config/recurring.yml
production:
  dart_monitor:
    class: DartMonitorJob
    queue: default
    schedule: every hour          # DART 신규 공시 감지 → 자동 추출

  factbook_monitor:
    class: FactbookMonitorJob
    queue: default
    schedule: at 9am every day    # Factbook 파일 감지

  daily_stock_price_update:
    class: Stock::DailyPriceUpdateJob
    queue: default
    schedule: at 6pm every day    # 장 마감 후 전일 종가 업데이트
```

### 자동 추출 흐름

```
DartMonitorJob (매시간)
  └── 신규 분기보고서 감지
      ├── Dart::HoldingsExtractionJob
      ├── Dart::BanksExtractionJob
      ├── Dart::SecuritiesExtractionJob
      └── Dart::CardsExtractionJob
```

---

## 데이터 흐름 요약

### 분기말 데이터 저장 (배치)

```
데이터 소스                    추출 계층                     저장
─────────────────          ──────────────────           ─────────────
XBRL API ──────────┐
DART XML ──────────┤       Orchestrator
Factbook ──────────┤       (업종별)        ──▶  Persister ──▶  DB
DART APIs ─────────┤       + Calculator
FSC Stock API ─────┘
```

### 실시간 데이터 조회 (대시보드)

```
Dashboard  ──▶  RealtimeIndicatorService  ──▶  Stock API (최근 종가)
    │                    │
    │                    └──▶  DB (저장된 EPS/BPS)
    │                               │
    └───────────────────────────────┴──▶  실시간 PER/PBR 계산
```

---

**작성일**: 2026-03-30
**버전**: 2.0 (5개 업종 아키텍처, XBRL API-first 전환 반영)

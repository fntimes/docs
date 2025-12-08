# 데이터 추출 서비스 워크플로우

Compass Engine의 경영성과 데이터 추출 구조와 흐름을 설명하는 문서.

---

## 개요

경영성과 데이터 추출은 **이중 구조**로 설계됨:

| 구분 | 저장 방식 | 용도 | 데이터 출처 |
|------|----------|------|------------|
| **분기말 데이터** | DB 저장 (performance_values) | 분기별 비교, 랭킹, 필터링 | Factbook + 분기말 주가 |
| **실시간 데이터** | 조회 시 계산 (저장 안함) | 대시보드 현재가 표시 | 최근 거래일 주가 + 저장된 EPS/BPS |

### 이중 구조의 필요성

**대시보드 화면 예시**:
```
2025년 3분기 KB금융 경영성과
├─ 당기순이익: 4.2조원 (Factbook)
├─ ROE: 11.2% (Factbook)
├─ PER: 8.55배 (실시간) ← "12/6 종가 기준"
└─ PBR: 0.73배 (실시간) ← "12/6 종가 기준"
```

- **Factbook 지표**: 분기말 확정 데이터, DB에 저장
- **PER/PBR 실시간**: 어제 종가 기준, 조회 시 계산
- **PER/PBR 분기말**: 분기말 주가 기준, DB에 저장 (랭킹/필터용)

---

## 서비스 구조

```
app/services/
├── financial_institutions/
│   └── holdings/
│       ├── orchestrator.rb         # 통합 추출 (Factbook + 분기말 주가)
│       ├── factbook_extractor.rb   # Factbook 지표 추출
│       └── value_saver.rb          # DB 저장
│
└── stock/
    ├── fsc_client.rb               # 주가 API 클라이언트
    ├── daily_price_updater.rb      # 전일 종가 일괄 업데이트
    ├── market_value_calculator.rb  # PER/PBR/TSR 계산
    └── realtime_indicator_service.rb  # 실시간 지표 조회

app/jobs/
└── stock/
    └── daily_price_update_job.rb   # 주가 업데이트 스케줄 Job
```

---

## 분기말 데이터 추출

### 1. Orchestrator (통합 추출기)

Factbook 추출과 분기말 주가 조회를 통합 관리.

```ruby
# app/services/financial_institutions/holdings/orchestrator.rb
module FinancialInstitutions
  module Holdings
    class Orchestrator
      def extract(file_path:)
        # 1. Factbook 추출
        factbook_result = extract_from_factbook(file_path)

        # 2. 분기말 주가 조회 + 시장가치 지표 계산
        stock_result = fetch_quarter_end_stock_data(factbook_result)

        # 3. 결과 병합
        merge_results(factbook_result, stock_result)
      end
    end
  end
end
```

#### 추출 흐름

```
┌─────────────────────────────────────────────────────────────────┐
│                        Orchestrator                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     │
│  │  Factbook    │     │   Stock      │     │   Market     │     │
│  │  Extractor   │ ──▶ │   API        │ ──▶ │   Calculator │     │
│  └──────────────┘     └──────────────┘     └──────────────┘     │
│         │                    │                    │              │
│         ▼                    ▼                    ▼              │
│    indicators          stock_price           PER, PBR, TSR      │
│    calculated          market_cap            dividend_yield     │
│                        listed_shares                            │
│                                                                  │
│  ──────────────────────────────────────────────────────────────▶│
│                                                                  │
│                       merge_results()                            │
│                            │                                     │
│                            ▼                                     │
│                   ┌──────────────┐                              │
│                   │   Result     │                              │
│                   │  {           │                              │
│                   │   indicators,│                              │
│                   │   calculated,│                              │
│                   │   stock_data,│                              │
│                   │   market_    │                              │
│                   │   indicators │                              │
│                   │  }           │                              │
│                   └──────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

#### 사용 예시

```ruby
# 단일 기업 추출
orchestrator = FinancialInstitutions::Holdings::Orchestrator.new(
  company_class: FinancialInstitutions::Holdings::Kb,
  year: 2025,
  quarter: 3
)
result = orchestrator.extract(file_path: "/path/to/kb_factbook.xlsx")

# 결과 구조
result[:indicators]        # Factbook 직접 추출 지표
result[:calculated]        # 파생 계산 지표
result[:stock_data]        # 분기말 주가 데이터
result[:market_indicators] # 분기말 시장가치 지표 (PER, PBR, TSR)
result[:metadata]          # 메타데이터

# 4대 금융지주 일괄 추출
results = FinancialInstitutions::Holdings::Orchestrator.extract_all(
  year: 2025,
  quarter: 3,
  file_paths: {
    kb: "/path/to/kb_factbook.xlsx",
    shinhan: "/path/to/shinhan_factbook.xlsx",
    hana: "/path/to/hana_factbook.xlsx",
    woori: "/path/to/woori_factbook.xlsx"
  }
)
```

---

### 2. 분기말 주가 조회

Orchestrator 내부에서 분기말 주가 조회 및 시장가치 지표 계산.

```ruby
def fetch_quarter_end_stock_data(factbook_result)
  # Factbook에서 EPS, BPS, DPS 추출
  eps = find_indicator_value(indicators, calculated, :eps, :eps_ytd)
  bps = find_indicator_value(indicators, calculated, :bps)
  dps = find_indicator_value(indicators, calculated, :dps)

  # 분기말 주가 조회 (예: 2025-09-30)
  quarter_end = quarter_end_date(year, quarter)
  stock_data = @stock_client.get_latest_stock_price(company.stock_code, quarter_end)

  # 전분기말 주가 조회 (TSR 계산용)
  prev_stock_data = fetch_previous_quarter_price(company.stock_code)

  # 시장가치 지표 계산
  market_indicators = @calculator.calculate_all(
    stock_price: stock_data[:close_price],
    eps: eps,
    bps: bps,
    dps: dps,
    start_price: prev_stock_data&.dig(:close_price),
    shares_outstanding: stock_data[:listed_shares]
  )

  # 결과 반환
  {
    stock_price: stock_data[:close_price],
    stock_date: stock_data[:date],
    listed_shares: stock_data[:listed_shares],
    market_cap: stock_data[:market_cap],
    per: market_indicators[:per],
    pbr: market_indicators[:pbr],
    tsr: market_indicators[:tsr],
    dividend_yield: market_indicators[:dividend_yield]
  }
end
```

---

### 3. ValueSaver (DB 저장)

Orchestrator 결과를 performance_values 테이블에 저장.

```ruby
# app/services/financial_institutions/holdings/value_saver.rb
class ValueSaver
  STOCK_INDICATORS = %w[stock_price listed_shares market_cap].freeze
  MARKET_INDICATORS = %w[per pbr tsr dividend_yield].freeze

  def save(result)
    ActiveRecord::Base.transaction do
      # 1. Extraction 레코드 생성
      @extraction = create_extraction(result)

      # 2. Factbook 지표 저장
      save_factbook_indicators(result[:indicators], stats)
      save_factbook_indicators(result[:calculated], stats, calculated: true)

      # 3. 분기말 주가 데이터 저장
      save_stock_data(result[:stock_data], stats)

      # 4. 분기말 시장가치 지표 저장
      save_market_indicators(result[:market_indicators], stats)
    end
  end
end
```

#### 저장 결과

```ruby
# performance_values 테이블에 저장되는 데이터
{
  company_id: 1,              # KB금융
  indicator_id: 45,           # per
  year: 2025,
  quarter: 3,
  period_type: "PIT",         # Point-In-Time (분기말 시점)
  value: 8.55,
  calculated: true,
  metadata: { source_type: "calculated" }
}
```

---

## 전일 종가 업데이트

### DailyPriceUpdater

상장 종목의 전일 종가를 companies 테이블에 저장. PER/PBR 계산에 활용.

```ruby
# app/services/stock/daily_price_updater.rb
class DailyPriceUpdater
  def update_all(delay: 0.1)
    Company.listed.where.not(stock_code: nil).each do |company|
      update_company(company)
      sleep(delay)
    end
  end

  def update_company(company)
    price_data = client.get_latest_stock_price(
      company.stock_code,
      base_date.strftime("%Y%m%d")
    )

    company.update!(
      stock_price: price_data[:close_price],
      stock_price_date: Date.parse(price_data[:date]),
      listed_shares: price_data[:listed_shares]
    )
  end
end
```

#### 사용 예시

```ruby
# 전체 상장 종목 업데이트
updater = Stock::DailyPriceUpdater.new
result = updater.update_all
# => { updated: 4, failed: 0, skipped: 0, base_date: "2025-12-07" }

# 특정 날짜 기준
updater = Stock::DailyPriceUpdater.new(base_date: Date.new(2025, 12, 6))
result = updater.update_all

# 시가총액 조회
Stock::DailyPriceUpdater.market_cap(company)
# => 49_860_000_000_000 (원)
```

### 스케줄링 (Solid Queue)

매일 18:00 장 마감 후 자동 실행.

```yaml
# config/recurring.yml
production:
  daily_stock_price_update:
    class: Stock::DailyPriceUpdateJob
    queue: default
    schedule: at 6pm every day
```

```ruby
# 수동 실행
Stock::DailyPriceUpdateJob.perform_now
Stock::DailyPriceUpdateJob.perform_later(base_date: "2025-12-06")
```

### 저장 위치

```
companies 테이블
├── stock_price        # 전일 종가 (decimal 12,2)
├── stock_price_date   # 기준일
└── listed_shares      # 상장주식수
```

---

## 실시간 데이터 조회

### RealtimeIndicatorService

대시보드에서 현재 주가 기반 지표 표시 시 사용.

```ruby
# app/services/stock/realtime_indicator_service.rb
class RealtimeIndicatorService
  def get_indicators(stock_code:, base_date: Date.today)
    # 1. 최신 주가 조회 (전 거래일)
    stock_data = fetch_latest_price(stock_code, base_date)

    # 2. 최근 분기 Factbook 데이터 조회 (DB에서)
    factbook_data = get_latest_factbook_data(company)

    # 3. 시장가치 지표 계산
    build_result(stock_code, stock_data, factbook_data, base_date)
  end
end
```

#### 사용 예시

```ruby
service = Stock::RealtimeIndicatorService.new

# 단일 기업 조회
result = service.get_indicators(stock_code: "105560")
# => {
#      stock_price: 89700,
#      stock_date: "20251206",
#      per: 8.55,
#      pbr: 0.73,
#      tsr_qtd: 4.61,
#      tsr_ytd: 42.5,
#      factbook_year: 2025,
#      factbook_quarter: 3,
#      ...
#    }

# 4대 금융지주 일괄 조회
results = service.get_holdings_indicators
# => {
#      kb: { per: 8.55, pbr: 0.73, ... },
#      shinhan: { per: 9.12, pbr: 0.68, ... },
#      ...
#    }
```

#### 계산 로직

```
실시간 PER = 최근 거래일 종가 / 저장된 EPS (최근 분기 Factbook)
실시간 PBR = 최근 거래일 종가 / 저장된 BPS (최근 분기 Factbook)
TSR(QTD) = (현재가 - 분기초가 + 배당) / 분기초가 × 100
TSR(YTD) = (현재가 - 연초가 + 배당) / 연초가 × 100
```

---

## Stock API 연동

### FscClient (금융위원회 주가 API)

```ruby
# app/services/stock/fsc_client.rb
class FscClient
  def get_latest_stock_price(stock_code, date)
    # API 호출: 해당 날짜 이전 최근 거래일 종가
    # 반환: { close_price, date, listed_shares, market_cap }
  end
end
```

### MarketValueCalculator (지표 계산기)

```ruby
# app/services/stock/market_value_calculator.rb
class MarketValueCalculator
  def calculate_per(stock_price:, eps:)
    return nil if eps.nil? || eps.zero?
    (stock_price / eps).round(2)
  end

  def calculate_pbr(stock_price:, bps:)
    return nil if bps.nil? || bps.zero?
    (stock_price / bps).round(2)
  end

  def calculate_tsr(start_price:, end_price:, dividend: 0)
    return nil if start_price.nil? || start_price.zero?
    ((end_price - start_price + (dividend || 0)) / start_price * 100).round(2)
  end

  def calculate_all(stock_price:, eps:, bps:, dps:, start_price:, shares_outstanding:)
    {
      per: calculate_per(stock_price: stock_price, eps: eps),
      pbr: calculate_pbr(stock_price: stock_price, bps: bps),
      tsr: calculate_tsr(start_price: start_price, end_price: stock_price, dividend: dps),
      dividend_yield: calculate_dividend_yield(stock_price: stock_price, dps: dps)
    }
  end
end
```

---

## 데이터 흐름 요약

### 분기말 데이터 저장 (배치)

```
Factbook Excel  ──▶  FactbookExtractor  ──▶  Orchestrator  ──▶  ValueSaver  ──▶  DB
                                                 │
Stock API       ─────────────────────────────────┘
(분기말 주가)
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

## 지표 저장 규칙

### period_type 구분

| period_type | 설명 | 예시 |
|-------------|------|------|
| `Q` | 분기 누적 | 분기별 TSR, 분기 당기순이익 |
| `YTD` | 연초~현재 누적 | YTD 당기순이익, ROE |
| `PIT` | 시점 값 | 주가, PER, PBR, 자본총계 |

### 단위 변환

모든 금액 지표는 **원(₩) 단위**로 저장.

```ruby
# Factbook (단위: 십억원)
net_income: 4_200  # 4.2조원

# DB 저장 시 (단위: 원)
Performance::Value.create!(
  value: 4_200 * 1_000_000_000  # => 4,200,000,000,000
)

# 조회 시 표시 단위로 변환
indicator.from_won(value)  # => 4,200 (십억원)
```

---

## 테스트 스크립트

### 분기말 + 실시간 통합 테스트

```bash
bundle exec rails runner tmp/test_final_structure.rb
```

```ruby
# tmp/test_final_structure.rb
# 1. Orchestrator 테스트 (분기말 데이터 추출)
orchestrator = FinancialInstitutions::Holdings::Orchestrator.new(
  company_class: FinancialInstitutions::Holdings::Kb,
  year: 2025,
  quarter: 3
)
result = orchestrator.extract(file_path: "/path/to/factbook.xlsx")

# 2. RealtimeIndicatorService 테스트 (실시간 데이터)
realtime = Stock::RealtimeIndicatorService.new
rt_result = realtime.get_indicators(stock_code: "105560")

# 3. 비교
puts "분기말 PER: #{result[:market_indicators][:per]}"
puts "실시간 PER: #{rt_result[:per]}"
```

---

## 향후 개선 계획

| 항목 | 설명 |
|------|------|
| TSR 계산 고도화 | YTD 배당 합계 정확히 계산 |
| API 캐싱 | 동일 날짜 주가 조회 캐싱 |
| 배치 처리 | 4대 금융지주 일괄 추출 자동화 |
| 모니터링 | 주가 API 장애 시 알림 |

---

**작성일**: 2025-12-08
**버전**: 1.2 (전일 종가 업데이트 서비스 추가)

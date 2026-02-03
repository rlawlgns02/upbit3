# 업비트 AI 자동매매 시스템 - 코드 분석 보고서 v2

> **분석 일시**: 2026-02-03
> **분석 범위**: 전체 소스 코드 (13개 파일)
> **분석자**: Claude AI

---

## 목차
1. [이전 보고서 수정 완료 현황](#1-이전-보고서-수정-완료-현황)
2. [신규 수정 사항](#2-신규-수정-사항)
3. [잔여 이슈 및 개선 권장](#3-잔여-이슈-및-개선-권장)
4. [최종 요약](#4-최종-요약)

---

## 1. 이전 보고서 수정 완료 현황

### 1.1 CODE_REPORT.md 수정 완료 항목 (24개)

| # | 파일 | 이슈 | 상태 |
|---|------|------|------|
| 1 | trading_env.py | NaN 처리 방식 개선 (ffill/bfill) | ✅ 완료 |
| 2 | trading_env.py | 매수 로직 이중 공제 수정 | ✅ 완료 |
| 3 | trading_env.py | 보상 계산 타이밍 수정 | ✅ 완료 |
| 4 | upbit_client.py | API 응답 타입 일관성 | ✅ 완료 |
| 5 | ai_predictor.py | 매수/매도 임계값 균형 | ✅ 완료 |
| 6 | ai_predictor.py | 신뢰도 계산 최적화 (5회) | ✅ 완료 |
| 7 | backtest.py | 빈 데이터 처리 추가 | ✅ 완료 |
| 8 | backtest.py | 승률 계산에 수수료 반영 | ✅ 완료 |
| 9 | backtest.py | 매수 로직 이중 공제 수정 | ✅ 완료 |
| 10 | market_analyzer.py | 0으로 나누기 방지 | ✅ 완료 |
| 11 | signal_generator.py | news_count 파라미터 전달 | ✅ 완료 |
| 12 | webapp.py | CORS 정책 강화 | ✅ 완료 |
| 13 | webapp.py | API 키 검증 강화 | ✅ 완료 |
| 14 | trading_bot.py | 환경 캐싱 구현 | ✅ 완료 |
| 15 | trading_bot.py | 빈 티커 응답 처리 | ✅ 완료 |
| 16 | market_analyzer.py | NaN/Division by zero 보호 강화 | ✅ 완료 |
| 17 | main.py | 백테스트 인덱스 불일치 수정 | ✅ 완료 |
| 18 | main.py | API 실패 시 예외 처리 추가 | ✅ 완료 |
| 19 | webapp.py | Rate Limiting 구현 | ✅ 완료 |
| 20 | webapp.py | WebSocket 연결 수 제한 | ✅ 완료 |
| 21 | webapp.py | 동시성 Lock 추가 | ✅ 완료 |
| 22 | webapp.py | WebSocket 브로드캐스트 에러 처리 | ✅ 완료 |
| 23 | signal_generator.py | 캐시 크기 제한 추가 | ✅ 완료 |
| 24 | recommend.py | API 실패 시 예외 처리 추가 | ✅ 완료 |

---

## 2. 신규 수정 사항

### 2.1 호가 단위 문제 해결 (Critical)

**파일**: `src/api/upbit_client.py`

**문제**: KRW-AUCTION 등 특정 가격대의 코인 주문 시 호가 단위 오류 발생
```
[UPBIT] 지정가 매수 실패 - 상태코드: 400
에러: 'invalid_price_bid' - '주문가격 단위를 잘못 입력하셨습니다.'
```

**원인**: 업비트는 가격대별로 호가 단위가 정해져 있음
- 1,000원 ~ 10,000원: **5원 단위**
- 7,838원은 5원 단위가 아니므로 에러

**수정 내용**: 호가 단위 계산 함수 추가 및 자동 적용
```python
def get_tick_size(price: float) -> float:
    """업비트 호가 단위 계산"""
    if price >= 2000000: return 1000
    elif price >= 1000000: return 500
    elif price >= 500000: return 100
    elif price >= 100000: return 50
    elif price >= 10000: return 10
    elif price >= 1000: return 5
    elif price >= 100: return 1
    elif price >= 10: return 0.1
    elif price >= 1: return 0.01
    elif price >= 0.1: return 0.001
    else: return 0.0001

def adjust_price_to_tick(price: float, is_buy: bool = True) -> float:
    """가격을 호가 단위에 맞게 조정 (매수: 내림, 매도: 올림)"""
    tick_size = get_tick_size(price)
    if is_buy:
        adjusted = (price // tick_size) * tick_size
    else:
        adjusted = math.ceil(price / tick_size) * tick_size
    return adjusted
```

**적용**: `buy_limit_order()`, `sell_limit_order()` 함수에서 자동 호가 단위 조정

**상태**: ✅ 수정 완료

---

### 2.2 베어 except 수정

**파일**: `webapp.py:876`

**문제**: `except:` 사용 (모든 예외 캡처, 안티패턴)

**수정 내용**:
```python
# 수정 전
except:
    pass

# 수정 후
except Exception:
    pass
```

**상태**: ✅ 수정 완료

---

## 3. 잔여 이슈 및 개선 권장

### 3.1 동시성 문제 (중요)

**현황**: Lock이 선언되어 있으나 모든 상태 변경 지점에서 사용되지 않음

**영향받는 코드**:
```python
# webapp.py - Lock 없이 상태 변경
trading_status['trade_count'] += 1  # Race condition 위험
trading_status['trade_history'].append(...)  # 리스트 append는 원자적이지 않음
```

**권장**: 모든 `trading_status`, `auto_trading_status` 수정 시 Lock 사용

**우선순위**: 높음

---

### 3.2 Magic Numbers (중요)

**발견된 하드코딩 값들**:

| 파일 | 값 | 용도 |
|------|-----|------|
| trading_env.py:166 | 60 | 기술 지표 최소 스텝 |
| trading_env.py:193 | 0.95 | 매수 금액 비율 |
| trading_bot.py:124 | 5000 | 최소 거래 금액 |
| webapp.py:164-165 | 10.0 | 목표/손절 비율 |
| webapp.py:325 | 60 | Rate limit 최대 요청 |
| webapp.py:329 | 100 | 최대 WebSocket 연결 |

**권장**: 설정 파일(`config.py`) 분리

**우선순위**: 중간

---

### 3.3 성능 이슈

**동기 API 호출**:
- `upbit_client.py`: 모든 API 호출이 `requests` (동기)
- webapp.py의 async 함수 내에서 블로킹 발생

**권장**: `aiohttp` 사용하여 비동기 처리

**우선순위**: 낮음 (장기 리팩토링)

---

### 3.4 메모리 관리

**RateLimiter cleanup**:
- `cleanup()` 메서드가 서버 종료 시에만 호출됨
- 장시간 운영 시 메모리 증가 가능

**권장**: 주기적 cleanup 호출 또는 TTL 기반 자동 정리

**우선순위**: 낮음

---

### 3.5 에러 처리 개선 필요

**광범위한 Exception 사용**:
```python
# 여러 파일에서 발견
except Exception as e:
    print(f"오류: {e}")
```

**권장**: 구체적인 예외 타입 사용
- `requests.exceptions.Timeout`
- `requests.exceptions.ConnectionError`
- `json.JSONDecodeError`
- `KeyError`, `IndexError`

**우선순위**: 중간

---

## 4. 최종 요약

### 4.1 수정 완료 항목

| 구분 | 수정 항목 수 |
|------|-------------|
| 이전 보고서 수정 | 24개 |
| 신규 수정 (호가 단위) | 1개 |
| 신규 수정 (베어 except) | 1개 |
| **총계** | **26개** |

### 4.2 코드 품질 점수

```
┌─────────────────────────────────────────────────────────────┐
│                      전체 코드 품질 평가                      │
├─────────────────────────────────────────────────────────────┤
│  구조 설계      ████████░░  80%  - 모듈화 양호               │
│  코드 가독성    ███████░░░  70%  - 주석 양호, 일관성 필요     │
│  에러 처리      ██████░░░░  60%  - 개선됨, 추가 작업 필요    │
│  보안성         ██████░░░░  60%  - CORS/인증 개선됨          │
│  테스트 커버리지 ░░░░░░░░░░   0%  - 테스트 없음              │
│  성능 최적화    █████░░░░░  50%  - 캐싱 추가, 비동기 필요    │
├─────────────────────────────────────────────────────────────┤
│  종합 점수:  65/100 (이전: 58/100, +7점 상승)                │
│  등급: B- (교육/연구 목적 적합, 프로덕션 전 추가 개선 권장)    │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 권장 조치사항 (우선순위별)

**즉시 조치 (높음)**:
- [x] 호가 단위 문제 해결
- [x] 베어 except 수정
- [ ] Lock 사용 일관성 확보

**단기 조치 (중간)**:
- [ ] Magic numbers 설정 파일 분리
- [ ] 구체적 예외 타입 사용
- [ ] 입력값 검증 강화

**장기 개선 (낮음)**:
- [ ] 비동기 API 클라이언트 전환
- [ ] 단위 테스트 추가
- [ ] 로깅 시스템 표준화

---

**문서 종료**

*이 보고서는 2026-02-03에 생성되었습니다.*

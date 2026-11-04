# 데이터 정제와 분석 기준

[프로젝트 소개](README.md) · [검토한 코드 스냅샷](https://github.com/ericsw2727/MLO_1st_project_SportsCompass/commit/c82193858d6144a326675aeee22b77fa48643f42)

이 문서는 나침의 데이터 정제·분석 관련 코드를 읽고 정리한 구현 설명입니다. 실제 오류를 재현·해결한 이력이나 이번에 수행한 테스트 결과와는 구분합니다.

## 1. 표기 통일

| 입력 예시 | 정리 결과 | 코드의 처리 |
|---|---|---|
| 서울특별시 | 서울 | 지역명 매핑 |
| swimming | 수영 | 종목 별칭 매핑 |
| 성인수영 초급 월수금 | 수영 | 대상·수준·요일 등 부가 표현 제거 |
| 헬스장 이용 | 표시명 헬스 · 기준 종목 보디빌딩 | 프로그램명 정리와 자격 종목 연결 분리 |
| 초등 풋살 | 표시명 풋살 · 기준 종목 축구 | 상위 종목 규칙 적용 |

동일 문자열 정리와 서비스 목적에 따른 상위 분류를 구분합니다. 기준 종목 연결은 운영 중인 규칙과 자격 데이터의 종목 목록에 따라 결정됩니다.

근거: [normalizers.py](https://github.com/ericsw2727/MLO_1st_project_SportsCompass/blob/c82193858d6144a326675aeee22b77fa48643f42/main/analytics/services/normalizers.py), [ProgramCleaner](https://github.com/ericsw2727/MLO_1st_project_SportsCompass/blob/c82193858d6144a326675aeee22b77fa48643f42/main/analytics/services/program_cleanup.py), [정제 예시 테스트](https://github.com/ericsw2727/MLO_1st_project_SportsCompass/blob/c82193858d6144a326675aeee22b77fa48643f42/main/analytics/tests/test_program_cleanup.py).

## 2. 모호한 종목은 자동 확정하지 않기

종목 매칭은 원문·정규화 일치, 동의어, 상위 종목, 여러 필드의 문맥 순으로 판단합니다. 프로그램명뿐 아니라 기존 종목, 프로그램 유형, 시설 업종·유형을 함께 참조합니다.

| 상황 | 처리 |
|---|---|
| 기준 종목과 명확하게 일치 | exact |
| 상위 종목 또는 충분한 문맥 근거 | similar |
| 필드 충돌·비슷한 후보·약한 근거 | review |
| 식별할 근거가 없음 | unmatched |

문맥 후보의 점수가 8점 이내로 비슷하거나 근거가 약하면 검토 대상으로 남깁니다. 이 점수는 규칙 기반 신뢰 점수이며, 통계적으로 검증한 정확도나 확률이 아닙니다.

근거: [SportMatcher.match](https://github.com/ericsw2727/MLO_1st_project_SportsCompass/blob/c82193858d6144a326675aeee22b77fa48643f42/main/analytics/services/sport_matching.py).

## 3. 기준일과 운영 상태 분리

| 조건 | 운영 상태 |
|---|---|
| 종료·폐강 표현이 확인됨 | ended |
| 기준일이 시작일과 종료일 사이 | active |
| 종료일이 기준일보다 이전 | ended |
| 시작일이 기준일보다 이후 | upcoming |
| 시작일 또는 종료일이 없음 | unknown |

종목이 식별되더라도 종료·예정·날짜 미확인 항목을 현재 운영 중 프로그램과 동일하게 다루지 않습니다. 복수 종목과 미매칭 항목도 분석 대상 선별 시 구분합니다.

수상스키·스키처럼 이름이 겹치는 경우 긴 표현부터 비교하여 중복 식별을 줄입니다. 원본 프로그램을 보존하고 별도 정제 결과에 운영 상태, 분석 가능 여부, 적용 규칙과 근거를 남깁니다.

근거: [ProgramCleaner](https://github.com/ericsw2727/MLO_1st_project_SportsCompass/blob/c82193858d6144a326675aeee22b77fa48643f42/main/analytics/services/program_cleanup.py).

## 4. 신청률의 결측과 집계 방식

`application_rate(capacity, applicants)`는 정원이 없거나 0인 경우, 또는 신청 인원이 없으면 `None`을 반환합니다. 수요 판정에서는 이를 '데이터 부족'으로 표시합니다.

| 지표 | 구현 기준 |
|---|---|
| 개별 신청률 | 신청 인원 ÷ 정원 × 100 |
| 요약 평균 신청률 | 유효한 신청 현황 레코드별 신청률의 산술 평균 |
| 종목별 신청률 | 유효한 신청 현황 레코드의 신청 인원 합계 ÷ 정원 합계 × 100 |

요약 평균과 종목별 비율은 계산 방식이 다르므로 같은 의미로 비교하지 않아야 합니다. 정원이 없다는 이유로 신청률을 0%로 바꾸면 실제 수요가 낮은 항목과 구분할 수 없습니다.

근거: [application_rate·summary·sport_analysis](https://github.com/ericsw2727/MLO_1st_project_SportsCompass/blob/c82193858d6144a326675aeee22b77fa48643f42/main/analytics/services/analytics.py).

## 5. 저장소에 있는 검증 사례

| 검증 항목 | 테스트가 확인하는 내용 |
|---|---|
| 원본 보존 | 정제 후 원래 프로그램명·지역·자격 데이터 유지 |
| 정상 운영 프로그램 | 단일 종목의 정리 결과와 분석 가능 여부 |
| 종료·날짜 누락 | ended·unknown·upcoming 상태 구분 |
| 복수 종목 | 탁구/농구 등을 하나의 종목에 강제 연결하지 않음 |
| 반복 실행 | 정제를 다시 실행해도 결과가 중복되지 않음 |
| 비교 지표 | 분석 가능 프로그램만 집계하고 자격 합계 유지 |
| 분모 0 | 프로그램이 없으면 비교 비율을 비워 안전하게 처리 |

근거: [test_program_cleanup.py](https://github.com/ericsw2727/MLO_1st_project_SportsCompass/blob/c82193858d6144a326675aeee22b77fa48643f42/main/analytics/tests/test_program_cleanup.py). 테스트 코드 존재를 확인했으며 이번 문서 작성 중 실행하지 않았습니다.

## 6. 해석할 때 지킬 기준

- 자격 취득 건수와 현재 활동하는 지도자 수를 동일하게 보지 않습니다.
- 자격 집계 레코드 수와 취득 인원 합계를 구분합니다.
- 프로그램 수·신청률로 실제 고용 수요를 확정하지 않습니다.
- 합성 신청 데이터는 서비스 시연 자료로 표시합니다.
- 정제 건수와 분석 지표에는 원천 데이터 시점·기준일·규칙 버전을 함께 기록하는 방향으로 개선합니다.

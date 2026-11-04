# KREAM | 데이터 정제와 검증

[프로젝트 소개](README.md) · [원본 저장소](https://github.com/ericsw2727/MLO_2nd_project_KREAM)

## 데이터 규모와 분류

원천 검수 CSV의 실제 수신량은 파일명과 달리 **335,000행**입니다. 조인 통합본 335,000행은 같은 업무를 다른 형태로 표현한 자료이므로 서로 다른 검수 670,000건으로 합산하지 않습니다.

| 분류 | 행 수 | 원본 대비 | 처리 |
|---|---:|---:|---|
| Silver | 271,787 | 81.13% | 검수별 최고 유효 개정 |
| Quarantine | 33,854 | 10.11% | 오류 원본·후보·사유 보존 |
| Excluded | 29,359 | 8.76% | 동일 중복 9,598 + 이전 유효 개정 19,761 |
| 합계 | 335,000 | 100% | 원본 수신량과 일치 |

검수 현재 상태는 271,787건이지만, 이전 유효 개정을 포함한 작업 이력은 291,548건입니다. 정상·격리·보관은 현재 상태 구성과 작업 이력을 구분하기 위한 분류입니다.

## 정제 규칙의 핵심

| 대상 | 규칙 |
|---|---|
| ID·상품명 | 확정된 표기 규칙과 유효 상품 코드·표준 사전으로 복구 |
| 날짜 | 시간대 누락은 KST, UTC 표기는 KST 변환 |
| 시간 순서 | 입고 ≤ 검수 완료 ≤ 추출, 역전 행 격리 |
| 금액 | 단위를 해석하고 USD는 과제 규칙의 고정 환율 1,350원 적용 |
| 소요시간 | 초를 분으로 통일, 결측·음수·-999 등 격리 |
| 재직여부 | 원문이 정확한 대문자 Y일 때만 Y, 나머지는 N |
| 동일 개정 충돌 | 충돌한 양쪽 모두 격리 |
| 최신 개정 오류 | 오류를 격리한 뒤 최고 유효 개정 선택 |
| 배상금 | CLAIM_FAKE_PASS인 경우 거래가 × 3, 그 외 0 |
| 메모 | 원문 보존, 메모의 지시문을 정제 규칙으로 사용하지 않음 |

고정 환율은 이 프로젝트의 변환 규칙이며 현재 시장 환율이 아닙니다. 현재 퇴사 상태만으로 과거 정상 검수까지 격리하지 않았고, 이름 대신 사번으로 정상 마스터와 재조인했습니다.

상품명 복구 예시에서는 유효 상품 코드가 정확하게 연결된 경우에만 표준명을 적용하고, 나머지 오류 플래그를 그대로 유지했습니다.

근거: [정제 정책](https://github.com/ericsw2727/MLO_2nd_project_KREAM/blob/ba2570024e9b8a2d0c107eed0d4b084934864a45/docs/PROCESSING_RULES.md) · [상품 코드 기반 복구 SQL](https://github.com/ericsw2727/MLO_2nd_project_KREAM/blob/ba2570024e9b8a2d0c107eed0d4b084934864a45/sql/repro/54C_rebuild_disposition_with_product_code.sql)

## 표준 사전과 모델

| 구성 | 범위 |
|---|---:|
| 표준 단어 | 149개 |
| 표준 용어 | 189개 |
| 타입 도메인 | 43개 |
| 공식 코드 | 32행 |
| 전체 컬럼 매핑 | 393개 |
| 문서화 대상 | 28개 테이블·393개 컬럼 |
| 정상 Silver·Gold 검증 대상 | 18개 테이블·188개 컬럼 |

문서화 범위와 정상 데이터 검증 범위가 다릅니다. Bronze·감사·격리 이력 등까지 포함한 컬럼 수를 정상 검증 대상 컬럼 수와 혼동하지 않습니다.

![핵심 물리 ERD](https://raw.githubusercontent.com/ericsw2727/MLO_2nd_project_KREAM/ba2570024e9b8a2d0c107eed0d4b084934864a45/assets/physical_erd.svg)

직원·카테고리 마스터와 검수 테이블의 실제 FK, 검수와 결합 테이블의 복합 FK를 확인한 모델입니다. 상품 표준 사전과 검수 사이의 논리적 조회 관계를 실제 FK로 표현하지 않습니다.

근거: [모델 상세](https://github.com/ericsw2727/MLO_2nd_project_KREAM/blob/ba2570024e9b8a2d0c107eed0d4b084934864a45/docs/MODEL.md)

## 검증을 분리한 이유

| 검증 | 저장된 결과 | 의미 |
|---|---|---|
| SQL 품질 스냅샷 | 14 PASS·1 NOT_CHECKED·1 RECORDED_DIFFERENCE | 저장된 업무 품질 규칙의 상태 |
| Pandas 독립 검증 | 482개 검사·실패 0 | 자료형·필수값·키·참조·허용 코드·계산 관계 |
| 원본 추적 | 13개 검사 위반 0 | 원본 행 분류·충돌 격리·변환 대조 |
| SQL 진단 대조 | 8개 지표 차이 0 | 원본 진단과 기존 SQL 수치 일치 |

Pandas 검증 코드는 OUTFILE의 SQL NULL인 `\N`을 문자 NULL과 구분하고, 헤더가 없는 CSV를 컬럼 메타데이터로 읽습니다. 금액 검산에는 Decimal을 사용하며, 정상 마스터 재조인 결과와 예측 기준 시각 이후 기록의 혼입 여부도 확인합니다.

근거: [검증 코드](https://github.com/ericsw2727/MLO_2nd_project_KREAM/blob/ba2570024e9b8a2d0c107eed0d4b084934864a45/src/verify_final.py) · [검증 요약 JSON](https://github.com/ericsw2727/MLO_2nd_project_KREAM/blob/ba2570024e9b8a2d0c107eed0d4b084934864a45/reports/pandas_verification_summary.json)

## 재현 결과 비교

원본 최종 ZIP과 재실행 ZIP의 CSV는 각각 29개입니다.

- 업무 CSV 27개는 바이트 단위로 일치했습니다.
- 품질 검사 파일은 검사 시각이 달랐습니다.
- 내보내기 목록은 출력 파일명과 기록 시각이 달랐습니다.

업무 결과와 실행 메타데이터 차이를 분리하여 재현성을 설명했습니다.

근거: [해시·변경 컬럼 비교](https://github.com/ericsw2727/MLO_2nd_project_KREAM/blob/ba2570024e9b8a2d0c107eed0d4b084934864a45/evidence/export_comparison.json)

## 남은 검증 과제

상대팀의 제시 기준은 Silver 280,800행·격리 21,600행·보관 32,600행입니다. 현재 분류와의 차이는 각각 -9,013행·+12,254행·-3,241행입니다. 총합은 같지만 세부 분류는 다릅니다.

목표 건수에 맞추기 위해 규칙을 바꾸지 않고, 기준 차이와 점검 조건의 미검증 상태를 남겼습니다. 본 문서는 기존 검증 기록을 정리했으며, 이번 작성 과정에서 MySQL 파이프라인이나 Pandas 검증을 다시 실행하지 않았습니다.

[검증 범위와 차이 설명](https://github.com/ericsw2727/MLO_2nd_project_KREAM/blob/ba2570024e9b8a2d0c107eed0d4b084934864a45/docs/VERIFICATION.md) · [프로젝트 소개로 돌아가기](README.md)

# 한국 세금 경정청구 환급액 계산을 위한 법률적 데이터 모델링 및 시뮬레이션 입력 항목 분석

## 1. 개요 및 핵심 요약

대한민국 조세 체계 내에서 경정청구(Rectification Claim)를 통한 세금 환급 시뮬레이션을 구축하기 위해서는 「조세특례제한법」(이하 조특법), 「법인세법」, 「소득세법」 및 「농어촌특별세법」에 산재된 복잡한 감면 요건을 '데이터 필드' 단위로 정밀하게 구조화해야 합니다. 특히 최근 세법 개정으로 인해 통합고용세액공제(조특법 제29조의8)가 신설되고, 혼인세액공제(조특법 제92조)가 도입되는 등 변동성이 크므로, 시뮬레이션 알고리즘은 연도별 적용 법령의 차이를 반영할 수 있는 시계열 데이터(Time-series Data) 처리가 필수적입니다.

본 보고서는 사용자가 요청한 12가지 주요 세액공제 및 감면 항목에 대하여, 법제처 국가법령정보와 국세청 유권해석을 기반으로 환급액 계산에 필수적인 '납세자 입력 데이터 필드'를 정의합니다. 각 항목은 법적 근거와 계산 로직을 분석한 후, 실제 시스템 구현 시 필요한 테이블 스키마 형태로 정리하였습니다.

**주요 고려 사항:**
*   **상호 배제 및 중복 적용:** 조특법 제127조에 따른 중복지원 배제 로직을 구현하기 위해서는 각 감면 항목의 선택 여부를 판단하는 플래그(Flag) 필드가 필수적입니다 [cite: 1, 2].
*   **최저한세 및 농어촌특별세:** 감면 세액이 산출되더라도 조특법 제132조의 최저한세 미달 여부와 농어촌특별세법에 따른 비과세 여부(조특법 제6조 등)를 판별하는 필드가 필요합니다 [cite: 3, 4, 5].
*   **사후 관리:** 통합고용세액공제나 통합투자세액공제 등은 감면 이후 고용 감소나 자산 처분 시 추징세액이 발생하므로, 사후 관리를 위한 이력 데이터 필드가 요구됩니다 [cite: 6, 7].

---

## 2. 항목별 데이터 필드 상세 분석

### 1. 조특법 제6조 (창업중소기업 등에 대한 세액감면)

창업중소기업 세액감면은 창업 지역, 대표자의 연령(청년 여부), 업종, 창업 시기에 따라 감면율(50%~100%)이 달라집니다. 특히 2025년 이후 창업분의 경우 수도권 과밀억제권역 외 지역 감면율 조정 등 개정 사항이 반영되어야 합니다 [cite: 8, 9, 10].

#### 1.1 법적 근거 및 로직
*   **감면율 결정:** 수도권과밀억제권역 여부, 청년(15~34세) 여부, 수입금액(4,800만 원 이하 등), 신성장 서비스업 해당 여부에 따라 50%, 75%, 100% 차등 적용 [cite: 8, 11].
*   **창업의 정의:** 합병, 법인전환, 폐업 후 재개업 등은 창업으로 보지 않으므로 이를 필터링할 데이터가 필요함 [cite: 8, 12].

#### 1.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제6조 제1항 | `biz_start_date` | Date | 필수 | 개업일. 법 적용 시기 판단 (2018.5.29 이후, 2025.1.1 이후 등) [cite: 8] |
| 제6조 제3항 | `industry_code` | String | 필수 | 한국표준산업분류코드. 감면 대상 업종 여부 판단 (제조, 건설, 통신판매 등) [cite: 8] |
| 제6조 제1항 | `rep_birth_date` | Date | 필수 | 대표자 생년월일. 청년창업(15~34세) 여부 판단. 병역 기간 차감 로직 필요 [cite: 13] |
| 제6조 제1항 | `military_service_period` | Int | 선택 | 병역 이행 기간(개월). 청년 나이 계산 시 차감 (최대 6년) [cite: 13] |
| 제6조 제1항 | `location_zone_code` | Enum | 필수 | 사업장 소재지 구분 (수도권과밀억제권역, 수도권외, 인구감소지역 등) [cite: 8, 11] |
| 제6조 제1항 | `revenue_amount` | Long | 필수 | 연간 수입금액. 생계형 창업(수입금액 4,800만 원 이하 -> 8,000만 원 개정 예정) 판단 [cite: 14, 15] |
| 제6조 제10항 | `is_new_biz` | Boolean | 필수 | 원시 창업 여부 (법인전환, 승계, 폐업 후 동종 재개업 아님을 확인) [cite: 12] |

---

### 2. 조특법 제7조 (중소기업에 대한 특별세액감면)

이 조항은 중소기업의 업종, 소재지, 규모(소기업/중기업)에 따라 5%~30%의 세액을 감면합니다. 최근 개정으로 감면 한도(1억 원)가 설정되었으며, 상시근로자 감소 시 한도가 축소되는 규정이 중요합니다 [cite: 16, 17, 18].

#### 2.1 법적 근거 및 로직
*   **감면율 매트릭스:** (수도권/비수도권) × (소기업/중기업) × (도소매/의료/기타업종)의 조합으로 결정 [cite: 16, 19].
*   **한도 축소:** 기본 한도 1억 원에서 상시근로자 감소 인원 1명당 500만 원씩 차감 [cite: 18, 20].

#### 2.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제7조 제1항 | `biz_size_type` | Enum | 필수 | 기업 규모 (소기업, 중기업). 매출액 기준 판단 [cite: 19] |
| 제7조 제1항 | `biz_location` | Enum | 필수 | 사업장 소재지 (수도권 내, 수도권 외) [cite: 16, 17] |
| 제7조 제1항 | `main_industry_type` | String | 필수 | 주업종 분류 (도소매업, 의료업, 제조업 등 분류에 따라 감면율 상이) [cite: 16] |
| 제7조 제3항 | `emp_count_current` | Int | 필수 | 당해 과세연도 상시근로자 수 [cite: 20] |
| 제7조 제3항 | `emp_count_prev` | Int | 필수 | 직전 과세연도 상시근로자 수. 감소 인원 계산용 [cite: 20] |
| 제7조 제1항 | `calculated_tax` | Long | 필수 | 산출세액. 감면 대상 소득 비율만큼 안분 계산 필요 [cite: 21] |

---

### 3. 조특법 제10조 (연구·인력개발비에 대한 세액공제)

R&D 세액공제는 '일반', '신성장·원천기술', '국가전략기술'로 나뉘며, 각각 공제율이 다릅니다. 또한 계산 방식이 '당기분 방식'과 '증가분 방식' 중 선택 가능하므로 이를 시뮬레이션해야 합니다 [cite: 22, 23, 24].

#### 3.1 법적 근거 및 로직
*   **기술 분류:** 일반 R&D(최대 25%), 신성장·원천(20~40%), 국가전략(30~50%) [cite: 22, 23, 25].
*   **계산 방식 선택:** Max(당기 지출액 × 기본율, (당기 - 전기) × 증가분율). 단, 직전 4년 평균보다 적으면 증가분 적용 배제 [cite: 22].

#### 3.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제10조 제1항 | `rnd_tech_category` | Enum | 필수 | 기술 구분 (일반, 신성장·원천, 국가전략) [cite: 22, 24] |
| 제10조 제1항 | `rnd_expense_current` | Long | 필수 | 당기 R&D 지출액 (인건비, 재료비 등) [cite: 23, 24] |
| 제10조 제1항 | `rnd_expense_prev` | Long | 필수 | 직전 과세연도 R&D 지출액 [cite: 22] |
| 제10조 제1항 | `rnd_expense_avg_4y` | Long | 필수 | 직전 4년간 R&D 연평균 발생액. 증가분 방식 적용 가능 여부 판단 [cite: 22, 26] |
| 제10조 제1항 | `revenue_amount` | Long | 조건부 | 매출액. 신성장·원천기술 공제율 산정 시 매출액 대비 R&D 비중이 필요할 수 있음 [cite: 22] |

---

### 4. 조특법 제24조 (통합투자세액공제)

기존 투자세액공제들이 통합된 조항으로, 자산 유형(일반, 신성장, 국가전략)에 따라 기본공제율이 다르고, 직전 3년 평균 투자액 초과분에 대해 추가 공제를 제공합니다 [cite: 7, 27, 28].

#### 4.1 법적 근거 및 로직
*   **대상 자산:** 기계장치 등 사업용 유형자산 (토지, 건물, 차량 등 제외). 단, 운수업 차량 등 예외 존재 [cite: 7, 29].
*   **공제 구조:** 기본공제(1~25% 등) + 추가공제(증가분의 10%, 기본공제의 2배 한도) [cite: 28].
*   **수도권 배제:** 수도권과밀억제권역 내 투자는 원칙적으로 배제되나, 중소기업의 대체투자 등 예외 로직 필요 [cite: 7, 30].

#### 4.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제24조 제1항 | `asset_class` | Enum | 필수 | 자산 분류 (일반사업용, 신성장사업화시설, 국가전략기술시설) [cite: 28] |
| 제24조 제1항 | `invest_amount_current` | Long | 필수 | 당해 연도 투자 금액 [cite: 31] |
| 제24조 제1항 | `invest_avg_3y` | Long | 필수 | 직전 3년간 연평균 투자금액 (추가공제 계산용) [cite: 28] |
| 제24조 제1항 | `asset_location_zone` | Enum | 필수 | 투자 자산 소재지 (수도권과밀억제권역 여부 판단) [cite: 7] |
| 제24조 제1항 | `biz_size` | Enum | 필수 | 기업 규모 (중소/중견/일반). 공제율 차등 (예: 중소 10%, 일반 1%) [cite: 27, 28] |

---

### 5. 조특법 제29조의8 (통합고용세액공제)

2023년부터 도입된 제도로, 고용 증가 인원에 대해 정해진 금액을 세액공제합니다. 청년, 장애인, 경력단절여성 등 '청년등' 우대 그룹과 '청년외' 그룹을 구분하여 계산합니다 [cite: 6, 32, 33].

#### 5.1 법적 근거 및 로직
*   **공제액:** (청년등 증가인원 × 우대공제액) + (청년외 증가인원 × 일반공제액) [cite: 32].
*   **청년등 범위:** 15~34세, 장애인, 60세 이상, 경력단절여성(남성 포함 가능성 법령 개정 주시 필요) [cite: 6, 32].
*   **사후관리:** 2년(중견/대기업) 또는 3년(중소) 내 인원 감소 시 공제액 추징 [cite: 6, 32].

#### 5.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제29조의8 제1항 | `emp_youth_current` | Decimal | 필수 | 당기 청년 등(장애인, 고령자 포함) 상시근로자 수 (매월 말 인원 평균) [cite: 32, 34] |
| 제29조의8 제1항 | `emp_youth_prev` | Decimal | 필수 | 전기 청년 등 상시근로자 수 [cite: 6] |
| 제29조의8 제1항 | `emp_total_current` | Decimal | 필수 | 당기 전체 상시근로자 수 [cite: 34] |
| 제29조의8 제1항 | `emp_total_prev` | Decimal | 필수 | 전기 전체 상시근로자 수. 전체 증가분이 청년 증가분보다 적으면 전체 한도 적용 [cite: 32] |
| 제29조의8 제1항 | `biz_location` | Enum | 필수 | 수도권 여부. 공제금액 차이 (예: 수도권 청년 1,450만 vs 지방 1,550만) [cite: 35] |
| 제29조의8 제7항 | `post_mgmt_flag` | Boolean | 선택 | 사후관리 기간 중인지 여부 (인원 감소 시 추징 로직 발동) [cite: 6] |

---

### 6. 조특법 제127조 (중복지원의 배제)

동일한 사업연도나 투자 자산에 대해 여러 세제 혜택이 겹칠 경우, 하나만 선택해야 하는 규정입니다. 시뮬레이터의 핵심 제약 조건(Constraint)입니다 [cite: 1, 2, 36].

#### 6.1 법적 근거 및 로직
*   **투자세액공제 간 중복 불가:** 통합투자세액공제와 다른 투자 관련 공제 중복 불가.
*   **감면과 공제 중복 불가:** 창업중소기업 감면(제6조)이나 중소기업 특별세액감면(제7조)을 받는 경우, 통합투자세액공제 등 특정 공제 적용 배제 [cite: 1]. (단, 고용증대 등 일부 예외 존재 [cite: 36]).

#### 6.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제127조 제4항 | `selected_reduction_code` | List | 필수 | 사용자가 선택한 세액감면 코드 (예: Art6, Art7). 중복 배제 필터링용 [cite: 2, 37] |
| 제127조 제2항 | `asset_overlap_flag` | Boolean | 필수 | 동일 자산에 대해 다른 공제를 신청했는지 여부 [cite: 36] |

---

### 7. 조특법 제132조 (최저한세액에 미달하는 경우의 조세특례 배제)

세액공제를 많이 받아도 납부해야 하는 최소한의 세금(최저한세)을 계산하여 공제 한도를 설정합니다 [cite: 4, 38].

#### 7.1 법적 근거 및 로직
*   **최저한세율:** 중소기업 7%, 일반법인 10~17%, 개인사업자 35%~45% (감면 전 산출세액 기준) [cite: 4].
*   **적용 배제:** 일부 세액공제(예: 중소기업 사회보험료 세액공제 일부 등)는 최저한세 적용 대상에서 제외되므로 구분 필요 [cite: 38].

#### 7.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제132조 | `tax_base_before_exempt` | Long | 필수 | 감면 전 과세표준 [cite: 4] |
| 제132조 | `taxpayer_type` | Enum | 필수 | 법인/개인 및 기업 규모 (최저한세율 결정: 7%~17% 등) [cite: 4] |
| 제132조 제1항 | `excluded_deduction_amt` | Long | 선택 | 최저한세 적용 제외 대상 공제액 합계 (예: 연구인력개발비 등) [cite: 4, 38] |

---

### 8. 법인세법 제57조 (외국납부세액공제)

해외에서 납부한 세금을 국내 법인세에서 공제하거나 손금에 산입하는 제도입니다 [cite: 39, 40].

#### 8.1 법적 근거 및 로직
*   **한도액 계산:** 산출세액 × (국외원천소득 / 과세표준) [cite: 40].
*   **방법 선택:** 세액공제 방식 vs 손금산입 방식 비교 시뮬레이션 필요.

#### 8.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제57조 제1항 | `foreign_income_amt` | Long | 필수 | 국외원천소득 금액 [cite: 40] |
| 제57조 제1항 | `foreign_tax_paid` | Long | 필수 | 외국 납부 세액 [cite: 40] |
| 제57조 제1항 | `total_tax_base` | Long | 필수 | 당해 사업연도 과세표준 총액 [cite: 40] |
| 제57조 제1항 | `total_calculated_tax` | Long | 필수 | 당해 사업연도 법인세 산출세액 [cite: 40] |

---

### 9. 소득세법 제59조의2 ~ 제59조의4 (특별세액공제 등)

근로소득자 및 성실신고사업자 등의 연말정산/확정신고 시 적용되는 공제 항목입니다 [cite: 41, 42, 43].

#### 9.1 법적 근거 및 로직
*   **자녀세액공제:** 자녀 수(1명 15만, 2명 30만 등)에 따른 정액 공제 [cite: 43].
*   **연금계좌:** 납입액의 12% 또는 15% (소득 기준) [cite: 44].
*   **특별세액공제:** 보장성보험료, 의료비(총급여 3% 초과분), 교육비, 기부금 등 [cite: 42].

#### 9.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제59조의2 | `num_children_8yo` | Int | 선택 | 8세 이상 자녀 수 [cite: 43] |
| 제59조의3 | `pension_deposit_amt` | Long | 선택 | 연금계좌 납입액 [cite: 44] |
| 제59조의4 | `med_expense_total` | Long | 선택 | 의료비 지출 총액 (총급여 3% 공제 문턱 계산용) [cite: 42] |
| 제59조의4 | `total_salary` | Long | 필수 | 총급여액. 공제 문턱 및 공제율(12% vs 15%) 판단 기준 [cite: 42] |

---

### 10. 조특법 제92조 (혼인에 대한 세액공제)

*사용자 질의 상 '제95조'로 언급되었으나, 최신 법령 및 개정안에 따르면 혼인세액공제는 제92조(신설)에 해당하거나 관련 논의 중인 항목입니다. 제95조의2는 월세세액공제입니다.* [cite: 45, 46, 47]. 여기서는 2024~2026년 적용 예정인 혼인세액공제(최대 100만 원)를 기준으로 작성합니다.

#### 10.1 법적 근거 및 로직
*   **요건:** 2024.1.1~2026.12.31 혼인신고한 거주자. 생애 1회. 부부 1인당 50만 원 (합산 100만 원) [cite: 9, 46, 48].
*   **특징:** 나이 제한 없음, 재혼 포함, 초혼 여부 확인 필요.

#### 10.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제92조(신설) | `marriage_report_date` | Date | 필수 | 혼인신고일 (2024~2026년 여부 판단) [cite: 46, 49] |
| 제92조(신설) | `is_first_claim` | Boolean | 필수 | 생애 1회 적용 여부 체크 [cite: 46, 50] |
| 제92조(신설) | `spouse_income_flag` | Boolean | 선택 | 맞벌이 여부 (부부 합산 공제 시뮬레이션용) [cite: 9] |

---

### 11. 조특법 제30조의4 (중소기업 사회보험료 세액공제)

고용 증가 인원에 대한 사회보험료 사용자 부담분을 공제해 줍니다. 2025년 이후 경정청구 시에는 통합고용세액공제와의 선택 적용 또는 일몰 여부를 확인해야 합니다 [cite: 51, 52].

#### 11.1 법적 근거 및 로직
*   **대상:** 중소기업 (중견기업 불가).
*   **공제액:** 청년등 증가인원 × 보험료 × 100%, 청년외 × 보험료 × 50% [cite: 52, 53].
*   **주의:** 통합고용세액공제(제29조의8)와 중복 적용 불가 [cite: 6].

#### 11.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 제30조의4 제1항 | `emp_increase_youth` | Decimal | 필수 | 청년 등 상시근로자 증가 인원 [cite: 51] |
| 제30조의4 제1항 | `insurance_premium_paid` | Long | 필수 | 사용자가 부담한 사회보험료 총액 [cite: 53] |
| 제30조의4 제1항 | `is_sme` | Boolean | 필수 | 중소기업 여부 (중견기업 적용 불가) [cite: 52] |
| 제30조의4 제2항 | `emp_decrease_flag` | Boolean | 선택 | 사후관리(2년) 중 고용 감소 여부 (추징 계산용) [cite: 54] |

---

### 12. 농어촌특별세법 (농특세)

조특법상 감면을 받을 경우, 감면세액의 20%를 농특세로 납부해야 하는 것이 원칙이나, 창업중소기업 감면 등 특정 항목은 비과세됩니다 [cite: 3, 55, 56].

#### 12.1 법적 근거 및 로직
*   **과세표준:** 조특법에 따라 감면받은 세액.
*   **세율:** 20% [cite: 55].
*   **비과세:** 창업중소기업(제6조), R&D(제10조), 전자신고세액공제 등은 비과세. 통합투자세액공제(제24조)는 과세 대상 [cite: 56, 57, 58].

#### 12.2 필요 입력 데이터 필드

| 법령 조문 | 필요 입력필드명 | 데이터타입 | 필수여부 | 설명 및 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 법 제5조 | `reduced_tax_amt` | Long | 필수 | 조특법에 의해 감면받은 세액 (과세표준) [cite: 55] |
| 법 제4조 | `exemption_code` | Enum | 필수 | 농특세 비과세 대상 여부 코드 (예: R&D=비과세, 투자세액=과세) [cite: 56, 57] |

---

## 3. 결론

위에서 정의한 데이터 필드들은 한국 세금 경정청구 시뮬레이션을 위한 최소한의 필수 단위입니다. 특히 **조특법 제127조(중복지원 배제)**와 **제132조(최저한세)**, 그리고 **농어촌특별세법**은 개별 세액공제 계산이 끝난 후 최종 환급액을 확정 짓는 '필터링 단계'에서 핵심적인 역할을 합니다.

시뮬레이션 시스템 구축 시에는 사용자가 입력한 `개업일`, `업종코드`, `상시근로자 수(시계열)` 데이터를 기반으로 각 조문의 적용 가능 여부를 1차 판단하고, 이후 `중복 배제` 및 `최저한세` 로직을 통해 최적의 공제 조합을 도출하는 알고리즘 설계가 필요합니다. 특히 **통합고용세액공제(제29조의8)**와 **사회보험료세액공제(제30조의4)**는 연도별(2023~2025) 선택 적용이 가능하므로, 이를 비교 분석할 수 있는 시나리오 기능이 포함되어야 합니다.

**Sources:**
1. [sootax.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFik8OfaDuOmqqRlLH09qRjvICnvt74WRfmGbN-yJjoUoK99taVN1i7jH2tuhfvqYRZlCbMSa8kg9o7tuTONBENSszTrWyJfumXeEloBn07)
2. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQE_jgJhZeMCOf91--v1IqnyqqyZhH0YUC-HQVc8mHzy-UGL1zeANzDON0RkY8tZOoyEnRZEBW_VljwHrMVayt1-AEuLgwxJfu5MGD8FfLeYH2h8WgsY0HWkDGfVXZpIw4Q9xkPv5MpEDdS2VDz06_U7Hdp09yekSkvcv_nnM-R8Z4nzMcaOJv8j7ohafoyFRkLAJMAYFXXfVqWPo42AiQ==)
3. [bznav.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHDHvIC3naKRCWBh_CS3HZ4PWaWfnMbi7joGhDjoKyn_ex9BpkHvCdAPMmhj_XKI9ZupAZ_PHw7kwb4E8p5xPUIxurjJaAPJJryMai7J4w7NxebY2ieRF1YEmk=)
4. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEkv2fKgN6RmYVHbYCdy82IhkQqQeUuN86Z6YLd6oKAG47wr8jQdbuSsNyQH72O5nkroWCY9voaosmfq8GFvrbBX_ti4NGjcBPLTdOIZz86_6By3eo=)
5. [bznav.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQE_ANhW7CrK0oA2gplQrV9zhlIiPLd_f1Z6kKc0myI3AycGtJe0xQPm5Km4d4nCuFF-xJ2wUb6hr436DIZ2G8etkI8MMEuPRAnrIBqFcYuVR9iyZ4w1DV7ssPw=)
6. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFIjldjmQrsOV64lPMjK5N15ZYHtZ0Nu8iPZd0O9v1PSshOuD02zVe2PnJrN9cKYq94tgLMKsDS26BRzorpxZq-R1VqgqLG94YHWFtkCzeqcgSAKHNFiz4c9NZeOMLN7rph0_Fdg9jQMUn0Wzg_VbxqCMIFe04bDXf2Yza9kT3E_IXbiAAUAUJthbKz2Tj3JU_xZtv4h0MQGWGIxHfCENfrTe9AP1sN3EoHUiVBT57oUH771smLWSZQpAy4JO8RttlmIbPIgQSbAkcaYdsMQ_X9D_pCU_Yz674mWEVZIDKIOMz9iqNcZQ==)
7. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQESjv74Ld-hTL8LUKb_VAbmIbzTa3eXPiW2nsFtSOvaeC8uxPfmFbGp_NZGRzAYH72lbbNSbiHjegO-CVqziqmq2Z7NQ2QZbW9W9AR4j_m1ExyvtF9JWBdFid6o3CWuzek8Y_Me341af29JGc7fxANPJF6soe-Gq0LEcW_2847iocvnJ3OOIZ81FqownlyJ8bEO9TA4_0kaErWto8UhApRJYgD-c3QfgqZ8rASqKGfxz6MnDftkJjT-PcpG6_FiOw1OzbxJ6T8Bcbt1z9WkAdMSCsPmpySoMLpg9ehqI6I6DjcvBbwjiSRFGQcay-e2aekqKKNHxBhj6jZ87-oTj4tTYW2_FLBMc2fO)
8. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHP57dbwoxSbnTZ_f22Bvhc65EuLDw25JPj1P7bbt1-O2PY5FsLS87gfqDoTL-NefTpNEPfbmHwAZqJ9KtBA_KJlG4hRqKa_sG50ygV7OW_rsBe2bOxMo57XeeG-hypwhOSSEE4FgNX50wmMQ_s8FaVs1ctVAt6HHnlrQiKWW5KUmFC-JT6TFVZ20rzXg9sPmtNiBwzYvMVkOXSJmqV)
9. [taxtimes.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQF14-YuCFgUq-jNQdVAbYvaimbJKyRKrefUgyS7BqoXHUMC5XZy_9SwRGrMMzkipxeyeDx5h8yRGt9ULh3K4Ms_3NtqurlXoq6GswAonBsPJBLfT7GWbhltg6m01GwaEqnEVSu9xTC3q6sKOHg=)
10. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHsNB3q8AzBaBh55jlhr9BYH5k98lAKoiKwOgktAGnQtmQw18iZafWX0ZQceBl5VIxGwbnI1AAbHJG-IPEflCzHvMsxPD26SYUnsyBAttW5Oz4_wDC4iLXQ09qUI1hpt8O0CLRohfzNvUyRcEgZA4QZ1gkUEcMlYgnWbQwdePlbrNL43q4oLnJ2cVbEh3kKpOWpwD88VYr4lVNM8znmcWhdihz7mQtIjVOCLY7momr_hDaZyE5opyPgoj3ctTlD7JKYbZKMrRV6JtxdJnWSrl8nry8Gh2NZvGtpHhWEHbk_mtewXuZ8wqTjIZoDFwSVi9jKwl3rEqRCjMYVeqGP8JbTGXV9sTz6-MlwhGQPBzsyfo2jB_AFokSV1v83SucpVqa6rFY-eHd9lBcoQVWeIc6Y5_HSUk1bYuyJqR2G8IIFRsO8B0jWGGayNVSlg3MUkh4IBgG_3uLvKT4JrNg0h6xcDckS2Favl_FaMhLDqWdFkwF3yt-EredYCsU1GK12FjacNu_2eIWyd6VmXR2iKQE=)
11. [wellobiz.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFb0u_GiQpfkYPDnxq-iQ8UNvOU3qZqnGNh3DQYN3vcyivKs3TKi-eIQv7vv8arC2ntwXpxdJP7eqhP48KmiGivSaIPKRvPdYEp_lvVsR5kr_NlB14BYZxZ-G27I5TjvFGZHdJ-51_6l451eul2L5FG8CaOTqvyqUdvlw==)
12. [oopy.io](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGKUM-KrzvrIjmS9Wvob2rjFYaBv2kgJnM0xBqBl0n6o5Yohy7e3K7OkHyBqPx8pIGXqRLzT3b-aaoFtVkrVe088H4avLXOWqnvOrPY1VCsREco484X8k_sAhclg6IFK_o7ySoReEgFPZGljuQsML3p4RA=)
13. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFhd0Rlubgb4ZNlJ_c8xqah-E8d0yvHNJB0pSIvTmn7A2Z8_RJBcWo54rs9ZDfstGb4pIq5XccxJXxeta3CL8BIRRCHRxz3G_aiatPLK4KF1MU3pPNKbaCq0hYq-KzTkNp_Fpg8XGJhrJvEpA9xDg0YJW7N_wvB8oCW9cBJ)
14. [creativepartners.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHS70MdUjWUsaVUV50cIqLwm2USH5XpvZbHdQBEaHzyHoLltOYrRLdcFaDj89hqy0PLkecbTGZjTsqwWgbupaYJ4-N1WgO4yb3IfTzr1HEn0utDh4njlijAPDe9HUJDHiKT7nZuXnDvDSbPTMOR8KFbK21nXZHDU90AICuNRFQ=)
15. [moleg.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEHSeg3Nay85sDa0-l8JaDCEWygDsOsksLvswZgrohLQ2sXF6tt3sDgQgnnEnvm3mbQqBApWN2sFfHf7kTHFpi7YVgAdGsQ3q3MbbbyareDww5vHabT1c3dny3vQ4RgVdlcEqQNmpiYjNrfSlus2zjQkQWS8SrMtT4hGsRK9lGeidkeM5SfVK5NaOrTDSAR3BoaaA==)
16. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHnaYisombgBwxsIA5t17rWZMmedNhQ-TfCFC8OilEacwDuu6tUqH7pOYjcansba8xTIyXqX5GCQ813u0BhBF_X5PZCNNVlbHjzJevpJKMYVrz-OqQkt6_6SQYzdLslIPAk0wwbzWlenHNlQUmxeSAL93laNYqD3jalSBjue7kzCDBKePZNEdMtM0Nw7It0339xwAkhWh4BnaRFjuztfEqots_fZDOYoT7oBh4s6uN7agiMiCzrV2OKRpZC5wRcoIMjK-GwEyzfV-PUWlI88Tl3DG2xeXRt9uOdHt48lmfmNg7ElClUUgDEv-dMld2s)
17. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHQOKtq0Ima_Xcpmj6QvSBhRMJNwTFEAZ-phRNizqSsi9-WxALcX2RitcIIfEhjj7vVQx6Bhx6Jz2Iwae4gspWtJAUIEe4nSNwn1F5ew8dRNuA-TJcJenh2jx4akr4-vfz_YPL_ldSKKyH-RwYLJlWhbhiHyjkyEG1fBvwjAMEZneozQNJ6Pf1M8wIiZGMAiEfNYLZSVo3FOaBRiHY3Mw==)
18. [save-tax.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHMFhbPMlxQZem5WuhH_66Jm05j9thDCaYqdoW5jvlq-cFyBDezFuVDCJ6snFMlDlPZnYkavG_9ypWlhGrwoHSWKOIb4co0wETXCmQM7UPmjLwkTYXzl4DH0niU1S9PBVi4DSNIi6P5lOdSyGp3m1M1OQSIS-Qe6qVGE3BsCa5YwgXlxAK52Tah-9Tvdgge-gY8lyzG36n3_V_V1_kw4oFyBGg348Upf8k3D7E7Dq2R_zaYs4ZvYOIK3Z4lUqrWPel6T59vqtZyy4HLhk6I9l83-5Ifaq9NNge1JK-3YY6pwmzlZqQvBmX0PI6MkZ-iPDOhpxLKznqaLV8SzkBQSGvNTb_CzY8MyejRYxEclbX5btSqnba2cg==)
19. [samili.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGH77ej-cOxPCBuffxHyHEe_nwbfywnK5ULBlDJpPFM9UQTjI-z87O8n5V9nsaH32KtQAwitDx6pfMc6hTRwOikC003zhyDYqxhSkzPPYFjodoNg1j-4hgaMC6yncaBuLPZnl4idTdgxR11E36aSBpBJu1efOYMyi08iaVOYZq3HZIWXYyzJI7lch9lXyRcEQ==)
20. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGDq4SZkMdFngG1FgrigxyHZ1ufu-IMR9BeVlNeba7auFhVdJrP2oE-I5ClAPt95zVZYDNdAPBJRM_WPpBgO2S-JH_-SZ8KEUOD99Pg2_CjQ9VXtt8_JjeA1J9vgS9x_a1E95nwF4RN1jmOxeGrViI6rvVJRCE-4KJGaTFf)
21. [findsemusa.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHG2A-0qscOWOGJkT1R7y3MQFGwIf_Ew-iUgGeTY3uiTizTYH-ODl3PwFunlIyQYfk1cfRVUmdDhACxsEpLO-jbRYZ6R8BhBidl3fuy1fd9NpatXWvI11z6q6X9J_3CRVOdXdHRWtt_AqHWq5qs5izn6PIGIZhzLA==)
22. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHdjrfvA4aDpHkCAQf7Kd7-FNpk3mymTPhFjBIUjQjREnnktkJMyz6-dgzRcmT48K2NuzY6ytABYVBCYZX7F5aLewq818TNBFgAV94Cy0I2ZZVmuwG8y43zq0ZBbiCqPx1gBDIrKZ-X0w_i7WytRV_nrunqSHpEnS9OKIGlnxk3TVjLRW1VqR7VTwKMGsYYlsSH_0Y_0MttXRt0pnJOxppWVQnm5phpwZKPcICJ-KedFcte9EADAbLnbDLP5ne2ziJOdfYFI2HPAQMPtN1QY2kClL6jfHTYJGku19foH83-NQB7aMPgDRtHEi4-579aOANTkaRjmg5qbDY=)
23. [kipf.re.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGtzMzi97ErqIaV-UFwqESqnX3iKOB4RQMwyFjZ6To8ocyz5JnXZIIqJUNfzzIb8QEjE9cziTVgbnquLjCzhAxRNqnOu6g3qya3vy1cmTzWf_mFT9lHffm0S6tr1MD0klOqWKbxMadroMTL6G0lUQYLRCtM_io9TDw4nW6CSboIfEFALgU4xs4ihg==)
24. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQE3r0jge9-oCjBB_kibhsA8wIStLrZBkvpFIpUL86lD6ybkeDTrEWwB1MFs7UU-xPuxaRR8GT4brwR9HxsDZSQCDHr1Zme7D8C-7xMYKPA7UfN1dMx7Bo7whGAlNLal_5Hl8KinB_j2LpBTIArT0Q==)
25. [cpa4.me](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGB_QJl-MYYrSxKmbKIC8YofztUZRbbq_B8O7zAoQLTorTp1UDVLAqScJ9_wIpgS6PUSeh7q3q7gbV1VyqbljAGE-xN9nRt6FTT0pjF_FHpzHnTfTVFTuPygYHZN9qSuiMlBqSThMiyypgP9hoBOg==)
26. [intn.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFykpmRpoNXF-rWG_XDPCbY-YY9XTqb5rYuydp4O1DHMtJM1OuxsvgIe2yYi7VkImOXSCh4xGA72G7iubkVeKSlqaO4V7zOCUNjPOtZb6jOm4dh76rCKKZHn-D6NTmj6rNQxImgFxBwzgQaZoCtRTo0)
27. [sootax.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHlkqlegDx0J7DWOxutFFxQyfuqJMEKEmFlKAVRJEfekM0_iQROK-Zx4m-SHwTWt6aSsTqO1KEpuV2Hhdqn8tNLyLMYYfuY86nfCpuqRxHN)
28. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHGFCaTLzDDFFWjaIE0eUezeLkGgHRhKFXeio9E2_uQJgBE5crSk_zH0JI26r0MvivVlrVWhs_BQN7Fo-XFMUII8kl3Sfmk5nIU95o980ecz4sQv3CLvn6aXvOgv81pJIIP3VbDQHQWtBzmFMSMiRxdJdBp_1N0q7-ku44guIn8mrTsd4kkHNxKZiNLNnQIFOI=)
29. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEJtJ4yMQU78OVVBy9StwqvJD1PZr6S1ZKkFGJIpBNMnxlbcnKoeqDmTqEwKXwYGDinF9f5kWJTrlD_r_yqSLhrlfJwpJ0x_W6WzmgfCKAf5HHfLSDVJ5yHazUIIbw3lcTJIyI7xYSRjk9quLM01lzMQ-dXS6VjPo1H2M5ZR4Vjp15_fmcErNQ=)
30. [daum.net](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQF4OeKqT9hZI6JMD7S5RlWA2LwhGP3XUT6l4dsyO1-gK8v0nQC9TMy-eWzE4wryaovMhRYKbQtpG1zh8Ior29Au71r2Km1Dx_wqG6Hoa-u2AMfcq1t3ZWFW53lYtfA=)
31. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHRIRGo5vRWqB5iBY7kl41wLamIogJsEy57p3oYHKacFeqizWxzAazsX2ZEytOFaSWym_M8FAh6ciS4GCAuPqjJzqS7FklzxzVUFmSJ3T7NItJZh1bW3DTFu-vriR8xwrdNS4Yaei398OxtU9oZvrhwz03FStQZvJy5re-X)
32. [clhs.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFoFVa8b3Wh7nhY_m0BN8moLY8dzCnB-yPTBfu7AqStOxMLeyRdxSiWXYfu1OoABhjEpxsUb2SjDaJHMnKL_ivIrnAhVa83gToIzHF5LRTprTlJkekFuL-W5woSncJIOWB1swwoLdRx5Uw4frtYvqD17hjeHAfROq-UqA3sNiUkOuFNXK2myjfX4KYOhosXrLn5dx61En1pf5njka738Xiz98E=)
33. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEYReR3pFrKyke9KmOghVcWv48gu9J3qM9rr2u_EBUwPKw5nlIQ5bGz2Lr3nJvU7u1l9wYdt1UtKnMg5JLILtL5FtrsytK6qUqaEs5RUg4yjzsEooNE8CeLDIMimhDUBDuLvRjtuBZ-wsAszPCQuRd3SZ15Hq68CN8RHkZUHf47irfUva8VUG49-k5cP6eLuTkR1WWfmCbjrmqQUUTuQyrPSqj79aLNB0MTukO9YasYFSwOxparu5o_FH1UaVJX_hJ49SKs-7LjEtPuZzkiTZXI8cug9VP8HxbpvCv6DwSAiQ==)
34. [bznav.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFzU5iqp-u2kiuk6wLSez5zLWRWsPzGGqr4oaTSRf5Zl81rnsu02e0K08g1VwS1GbKORBK8yAqbM8YVrwlAJepUsDs3I7iPtD3p8KY91nMK4zy_1-hDDwfJKt0=)
35. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEIjYIg1HLaiSHBmUH4NxvCzNuCvdC80VcRgWWTZH-78vRUOgOjw4TyGytjmHm1IXElZuGaU1G3c-CQcRSDMuue56CrO2sh4hRxbWr_YFwQkpM6wv1uJZNRz5ew8TFeaTJfXXIwVD8nKOToR7prexwTlU5w5xy8o2cF5bLg)
36. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHTnIKJEAP_IxIlp1ZV7oxMEh0wru9xizf6HOhUuaP0nHCh5dd3fZ--stTmDS5l1JY-d7M7_44TjwyfBmcUio3PkgG1EZhU7IgBBswsDS2gs_06hQi-pDobSgnchGdU2BtjulErQi4JO69tdz738GUldxyUTyzIxZIsItinnTMABIeP2SUm-ms8zWZ2Xibxt3HO3cjbcY2SZIhzu-1rJYpuzTpBbLeO6Tlj6tRY4nzyh8iFSHSt4ENCIBv1VaLvawiWN_rq9cD0u8uELrqf)
37. [lawnb.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHelFCRBaPAW0Cn875VhAXMd_fUCYxjhKto7Km02vS3DjcBCuenq8xpo-kr7jiVxN2Ozpfr7e-Y8e1nQEyZnfPmYQ3K-azAZnypXYHNNjSRu6XIWnWExqHuxWJQX018dhVlLQiWJutB6ILqtpyTYp_X)
38. [bznav.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFXFP5fFVtSkgZy6qJTwyJoy7yyiPLvKIp0MMTwUwtRlDIy2PdnZAFsadnsjJKjSm1_fVwtny4jVVgMO1US_thVdbAHSeL4E83UoBgwsc7N2RQBP-yQlM-76Ac=)
39. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFsB0QazeJsAoX4DZgxf3yUtCNqzFVb9AcTbDSvvQ2yVfcFNfO4ssLYcYmjMCVc29NvW0lrDTvhfUDprb24CkAhu2632e1MLE0oIwdeNmIkjxfsDdgxqkSVKayuCcDBvz7fVgus-3c9Q83HQADJ-kV3GCO7PtvyZaCy8BqU)
40. [bigcase.ai](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGS10VDfgNqRkwH6edqji78C_Q0vwDtGR5Tjcu8TJc-4rVvRlANSfFNSf5bS6wTvIYQ9_4pT85j_HqX79kCtLv137G29di9EO9NAfmPgI0i9GVjgeqhHP7zM8hkOGee5TwohBiHmDlvBfMpnXZjDAlQvl_2jo7E3SS6eHlfTMjlZff5lKyGc-3j8L6rWpXeG-GfeZLOusbE)
41. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHXRoiONc1BUYfgwG4j5TfCSqF3cDfJFw5bM6GCIzB0r5cRNIY_NfXX8nkw3ZjZ1_awXXPaXDLYWDZESr9vSIYlTlYjlK9qocRC-FzeMki0vJe-_t34REcqoznhKD3LfcrAZg_zZni3rO6JsPeLUKNpYIb_YDCMG3Dk69Bmg-wa7MEUk7mx7Gf63iitzNhgFEuIIxC30RFLaiGdM1U0WQ==)
42. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGzG7SfwS9H-TB76hKUCAvfPU46buqqt_1IDamd9ztwGkS59BACHcl45NNx4ohXV6A65JlPnRXz30S5CEU00G-zLNMUkcnc3aQ0AuzUNgmzf5A0m_-PJguFy7j5cEbyonZfUcxvO-N5Be73Pt7V8OelUwBdhYLyj49VcVgALDfovjUXuYL6ZWMIlwMqVOJ5z8nDUcrZGbkz4D2W_emcVg==)
43. [easylaw.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGTG_ruy-3DXNq2HZLJPEsAoWnejGYX7Xh7Rps5HHkGQCiVL6nmhyk-5IoAvcaSxEboFzsx1PBOlak6KSg2kbSzqyheNk0_S68M3rsz1_saYQIqSlV_SCJy0oDM-VkKDCGphzDDvxvOIjFlfmDvO-J417dt26N1x0N_6iKBUDTK025PHC60S9-8gHG6NA3AvEo=)
44. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEjWTwMk4BTm2ldKyDZ1fHuF7FVUnJhVRrSW7Sdp-MBCPNXEooTBkJLWWVhgGuu54Le2PizJ4walkrPRY1NYEZbN4RWn2Ssdau3BlRWIxnoaLJOM7BIfWgs1FFPEPBYFyGYvfMzBk_ph0aMmME7ML5e3-BLoTKsoS-2DHQ5)
45. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEwYbTeSwi613HfDZ01544PV0QnJKrlxSEcslZEerrfOQhmiC4jJYeyN5ciccUs8bL4GOwX4w3DxMNeZFuUB2HTyVQr3CPq-Nrr2l_bG1BMJeIiH1GckGiWxszibmEhSS3INPSbG-hgiiQ83T9xnt36hSXB-IwNR6qZOyGcW3T7K8oRj3AfeDx5waaUWJpXCQjC2eDmMXUvmzAyT9Th6w25WQ2bg4ED7I2Ge2bTyzWangn2cjB8dByVnREWuOANY-cj7bB5p5zQN1s=)
46. [clhs.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGUgFOmdtn17tDQ2FAYJF0nSA29yNMScSItLXmfV3-jtSLcZjuBZ4Ri0-G8d1gE8_Zo7W0Yufad5Q-zVGHZcKN9oei1-84SWS5P_ox2zp9HxMfVytQv1Banh8tiwzetMApT5o9nhGv1i9oAiz5SicBBRb9zQc29RZl2R1rRmXSbbZ_9IfSoBufWQ_7FUdv9qh_8VmAltis9aNt3NJnY0Q==)
47. [nhis.or.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFbqueVJGR6FwBxK4gsx5qSW3NiSY3OtZOb32dYaNwuqe7Si9Bn9jX2Bt9FQA1hgPwcvN1FWZ452_9ZfjjhsB7tTJf-SW1hztl7A9yyRlnfbAy81Mpehf-fVbKWHZrHE46rn5l4lKr0gl_Eg3i7oCrLah27mKFah2jelGEwxwY0SGMqROp0OoPLO74S7A==)
48. [welfarehello.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFDyzY_I18nmEmxyF2wZFKSL-3dYrN-BjjeLLAzHjZ6dWS1C9jh_OvRVxPRLJTOi7yM1vucozBWUuzTwHjv3IPqpvJSnCwrUvkmYV6jZ6ug9T3NYorfPvk7SsdK4uwq3qz_qy2Z91Vt7SssrX0YMFvnKeior86Lv5R8Vh57wx_f6m8pBp9ZVLkOOBDwFlg-)
49. [bznav.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEqlNeALUoT5IHDQYljH9zuSjErI4kgU7xSjf2Th8XmE-H_-MU6ZAJDLLlbY-E_9SsSWIcAntOmU9q8yMtLzEZc5Et4Dt1U2Vy6ion5GwTucFMhOX_DZRegwjB2)
50. [moef.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGa8KkTL4oIoTdCXycbmNVYQBzmpb7QeRkSG29dzTz6JGLLD4APW9dZaKmHtCSWLI4DjzgJgFdo9GD4MqTPZBUyuxPkvMvA7ys0jDvnbEdF8fbkDWXNHX0nRr9O6tT5Ept-WVJXvrpnzq3vXocLcYVTkx7J6fj8QKcFXlQ35rlW)
51. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGPbA0C6XfKaWGTa93z8tCFivV47JV15tD-xOzVNKyz2_WOuK_Lrgm7PXpU-Y1u8tiZ4yga9bbT9R0G3wxo6w2upUfy4qzO7YI7u_HHuFrfSy7ygkd6AdAzDcNqa3eIOHSVVXzxYezowX5Wu0k-soHIL7QDf9jrPgKqQAyvXa9vnsKlr7-5O8f9h0yn_VmUq0i8Xfy2wqtRLRwR_DkA9gDFJCtmUBHSOUHBIvQRnf0ln01-4apkYh9xC3q6_1vrA61yeFWNjO37ZKvz0zEs68bWlmA5J9W4b27IcJUvzrn9ZsikvnftiZqz)
52. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQH1k9iG8Rti0smDch5bMvSCTUWTvfcoDYNFWWXOmqxHuspn-5kSl-tDDzj40JG2oLMuHsPtYsWxmNYaZ_QwlnZ8tZQ7QmZEatuFvOA57dM2mIwKUiZDPs3Vfyt5KAf6SCWL7Y0yCaaptHNVvfTd3PPU6VvScjy_dkhk7rbDgP9sB28=)
53. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEzVY0bmN8fl8m7kBfxQGNtF5619sNbs1My6Px9fkfrIX3qJJV_NHhFQ1BPDod6XQ2x0M7rkLBx5-3utjcd0ySFNCQynP6YLmyRFrWOiZVvPPuiQOZYl1iBRZvwX_IIYeTgvNEwtDfPbsZWne4PQ66wjVLI4pwnyvfBAlZEGnNPWztxPm1mKKxaYHgZuAPE8I2MCmonnpNMoHNzjsLK6KlVTlireQqbc0ieCGa2cIS4oXsEoPg7eTFzm7DOTDxHLehSI5sWwR6IMJJibWcBcGUXzwjmS0kIs2ApeduuOBldZf-VW0DDUMkW5oA=)
54. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHWKdA7Bb88dkLKfwXo75bKk8rR8JuOtYZxs5Pxxn-5nJH4J83v_6_onTpdwNHAa4fwIRZ-_S4se-trfF1PiJAL-HL589d8qjD-yK4zkjjgc-6PXt9Hs0wbISHl2t0tTpoT8NcvS5XTJ6J7mQQLilYGI-cWhcU6ag3OuRdt)
55. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEifWhBkfrIdCMdhT980S1HFqxHav9ZoaoDffPrLjkRb6JYdt_xRP4zQHa4xz4j1en1VNAyxOCMnEvUcDU8qeNAixGgBOIKDxATCFwvKmJTxiDJ1C3Afzh84cdUB-aS-gznpvsBor3H91whWo1DdGw2inuKBmiwdvrsIeX2xACgvg930KXJFxge0tFlS2ORxAjU0vaPP0gx)
56. [bznav.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGM8YuqvHO71Mg3q-syTIz5whRCKHgF08JrCmdQJZwbogqf4w4Gr_lPPR4semgys2A8m5DJLlFVSUnbw2W6kf8egBXW76gVbpTMyRur1-EBWFygjnP7CZQdqco=)
57. [ezbungae.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQExeIDVM4LpCoqvbfqpGXCjogPdRCemnMnq-8i1NIuBnw4AV64_CLJ8QS60S_msYhLH606c5Ccw5hfC74zXfOnFJbi7ws6MB8cY-LAmNrFhplVbf25jNoNa66yfLGt7jFZQja1ZGQlf55p6-e9CTPzMIx_s7L3B2fDAHyMf_g==)
58. [bznav.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQF1Bk0m4LaRAtt9TM2_5wAHk03A4ih2yxg1JTdYBT3fC0SOYvYkeTSlnVsXzfSn-xuurQvk1eH0PNXswErMBtNby8phT7Pr5oc8rMJy-yAOGjmsfeiCmjESXQ==)

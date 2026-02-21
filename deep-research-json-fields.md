# 한국 세무 경정청구(환급) 시뮬레이션 시스템 데이터 스키마 설계 보고서

## 1. 개요 및 설계 방향
본 보고서는 한국 세무 경정청구(Correction Claim) 시뮬레이션 시스템 구축을 위한 데이터 필드 명세서입니다. 더존 Smart A, 세무사랑 Pro, 위하고(Wehago), 홈택스(Hometax) 등 국내 주요 세무 프로그램의 데이터 구조와 법정 서식(법인세법 별지 제3호, 소득세법 별지 제40호, 국세기본법 별지 제16호의2 등)을 기반으로 작성되었습니다.

시스템 설계 시 데이터는 크게 **기초 정보(Identity)**, **당초 신고 내역(Original Filing)**, **경정 청구 내역(Rectified Filing)**, **부속 명세서(Schedules)**로 구분됩니다. 본 스키마는 RESTful API 또는 NoSQL Document 구조에 적합하도록 JSON 포맷을 가정하여 정의되었습니다.

### 핵심 설계 고려사항
1.  **비교 구조(Comparison Structure):** 경정청구는 '당초 신고'와 '경정 청구'의 차액을 소명하는 것이 핵심이므로, 주요 세액 필드는 `initial`과 `corrected` 값을 쌍으로 관리해야 합니다 [cite: 1, 2].
2.  **공제 감면의 복잡성:** 최근 경정청구의 핵심인 '고용증대세액공제' 및 '통합투자세액공제'를 위해 사원별, 자산별 상세 데이터가 필수적입니다 [cite: 3, 4].
3.  **소프트웨어 호환성:** 더존/세무사랑의 엑셀 업로드 양식과 홈택스 전자신고 파일 레이아웃(Electronic Filing Layout)을 포괄하는 필드명을 사용합니다 [cite: 5, 6].

---

## 2. [공통] 경정청구 및 공제감면 데이터 스키마

이 카테고리는 법인과 개인사업자 모두에게 적용되는 경정청구서 본문과 핵심 세액공제(고용, 투자, R&D) 관련 필드입니다.

### 2.1. 경정청구서 기본 (국세기본법 별지 제16호의2)
경정청구의 주체와 청구 이유, 그리고 당초 신고와 수정 신고의 세액 비교를 위한 필드입니다 [cite: 2, 7].

| field_id | field_name_ko | data_type | required | description |
| :--- | :--- | :--- | :--- | :--- |
| `filing_year` | 귀속년도 | string | Y | YYYY 형식 (예: 2023) |
| `claim_date` | 청구일자 | date | Y | 경정청구 제출일 (YYYY-MM-DD) |
| `taxpayer_id` | 사업자등록번호 | string | Y | 하이픈 제외 10자리 숫자 |
| `corp_reg_no` | 법인/주민등록번호 | string | Y | 법인등록번호 또는 대표자 주민번호 |
| `biz_name` | 상호(법인명) | string | Y | |
| `claim_reason_code` | 경정청구 사유코드 | string | Y | 홈택스 코드 (예: M01 세액공제 누락 등) |
| `claim_reason_desc` | 경정청구 사유상세 | string | Y | 구체적인 청구 사유 서술 (예: 고용증대세액공제 추가 적용) |
| `initial_filing_date` | 당초신고일 | date | Y | 최초 정기신고 또는 수정신고일 |
| `refund_bank_code` | 환급은행코드 | string | N | 금융결제원 은행코드 (환급 발생 시 필수) |
| `refund_account_no` | 환급계좌번호 | string | N | 하이픈 제외 숫자 |
| `tax_base_initial` | [당초] 과세표준 | number | Y | 당초 신고된 과세표준 |
| `tax_base_corrected` | [경정] 과세표준 | number | Y | 수정된 과세표준 |
| `calc_tax_initial` | [당초] 산출세액 | number | Y | |
| `calc_tax_corrected` | [경정] 산출세액 | number | Y | |
| `total_tax_credit_initial` | [당초] 공제감면세액 | number | Y | 기신고된 공제/감면 총액 |
| `total_tax_credit_corrected`| [경정] 공제감면세액 | number | Y | 추가 공제 반영 후 총액 |
| `add_tax_initial` | [당초] 가산세 | number | Y | |
| `add_tax_corrected` | [경정] 가산세 | number | Y | |
| `total_paid_tax` | 기납부세액 합계 | number | Y | 원천징수, 중간예납, 수시부과 등 합계 |
| `refund_tax_amt` | 환급신청세액 | number | Y | (기납부세액) - (경정 결정세액) |

### 2.2. 상시근로자 월별 상세 (고용증대/통합고용 세액공제용)
가장 빈번한 경정청구 사유인 '고용 관련 세액공제' 시뮬레이션을 위한 핵심 데이터입니다. 더존 Smart A의 사원등록 및 급여대장 데이터와 매핑됩니다 [cite: 3, 6, 8].

*   **Data Structure:** Array of Objects (`employees`)

| field_id | field_name_ko | data_type | required | description |
| :--- | :--- | :--- | :--- | :--- |
| `emp_code` | 사원코드 | string | Y | ERP 내부 관리 코드 |
| `emp_name` | 성명 | string | Y | |
| `res_id` | 주민등록번호 | string | Y | 청년 연령(15~34세) 계산 및 중복 체크용 (암호화 필수) |
| `join_date` | 입사일자 | date | Y | 근로계약 시작일 |
| `leave_date` | 퇴사일자 | date | N | 근로계약 종료일 (재직 중이면 null) |
| `is_youth` | 청년여부 | boolean | Y | 세법상 청년 해당 여부 (자동 계산 권장) |
| `emp_type` | 고용형태 | string | Y | `REGULAR`(정규직), `CONTRACT`(계약직), `DAILY`(일용직) |
| `monthly_salary` | 월급여액 | array | Y | 1월~12월 급여 배열 (최저임금 미만자 배제 로직용) |
| `contract_period_start` | 근로계약 시작일 | date | N | 계약직의 경우 필수 (1년 미만자 배제용) |
| `contract_period_end` | 근로계약 종료일 | date | N | |
| `social_ins_enroll` | 4대보험 가입여부 | boolean | Y | 국민연금/건강보험 가입 여부 (미가입자 상시근로자 배제) |
| `is_relative` | 특수관계인 여부 | boolean | Y | 대표자의 친족 등 (공제 대상 제외) |

### 2.3. 투자자산 명세 (통합투자세액공제용)
사업용 자산 투자에 대한 세액공제 계산을 위한 필드입니다 [cite: 4].

*   **Data Structure:** Array of Objects (`investments`)

| field_id | field_name_ko | data_type | required | description |
| :--- | :--- | :--- | :--- | :--- |
| `asset_name` | 자산명 | string | Y | 기계장치, 운반구 등 |
| `acq_date` | 취득일자 | date | Y | 세금계산서 작성일자 기준 |
| `acq_amt` | 취득가액 | number | Y | 부가세 제외 공급가액 |
| `asset_class` | 자산구분 | string | Y | `NEW_GROWTH`(신성장), `ENERGY`(에너지절약), `GENERAL`(일반) |
| `is_used` | 중고여부 | boolean | Y | 중고 자산은 통합투자세액공제 제외 |
| `location_region` | 소재지(지역) | string | Y | `CAPITAL`(수도권과밀억제권역), `LOCAL`(지방) |
| `depreciation_method` | 감가상각방법 | string | N | 정액법/정률법 |
| `useful_life` | 내용연수 | integer | N | 기준내용연수 |

### 2.4. 이월세액공제 명세 (세액공제조정명세서 3)
과거에 공제받지 못하고 이월된 세액공제액을 관리하여, 이번 경정청구 시 사용할 수 있는지 판단합니다 [cite: 9, 10].

| field_id | field_name_ko | data_type | required | description |
| :--- | :--- | :--- | :--- | :--- |
| `credit_code` | 공제종류코드 | string | Y | 예: 11(고용증대), 23(R&D) |
| `occur_year` | 발생연도 | string | Y | 세액공제 원천 발생 연도 |
| `occur_amt` | 발생액 | number | Y | 당해연도 계산된 공제액 |
| `deducted_amt` | 기공제액 | number | Y | 이미 공제받은 금액 |
| `carried_over_amt` | 이월잔액 | number | Y | 금번 청구 시 공제 가능한 잔액 |
| `expiry_year` | 소멸연도 | string | Y | 이월공제 기한 (보통 10년) |

---

## 3. [법인세 전용] 상세 데이터 스키마

법인세 과세표준 및 세액조정계산서(별지 제3호 서식)와 주요 부속서류를 기반으로 합니다 [cite: 11, 12].

### 3.1. 법인세 기본 및 세무조정 (별지 제3호)

| field_id | field_name_ko | data_type | required | description |
| :--- | :--- | :--- | :--- | :--- |
| `net_income_book` | 결산서상 당기순이익 | number | Y | 손익계산서상 순이익 |
| `adj_add_amt` | 익금산입/손금불산입 | number | Y | 세무조정 합계 (가산) |
| `adj_sub_amt` | 손금산입/익금불산입 | number | Y | 세무조정 합계 (차감) |
| `donation_limit_excess` | 기부금 한도초과액 | number | N | 손금불산입된 기부금 |
| `entertainment_excess` | 접대비 한도초과액 | number | N | 접대비 조정명세서(갑) 결과값 [cite: 13] |
| `biz_car_private_use` | 업무용승용차 사적사용 | number | N | 운행기록부 미작성/사적사용분 [cite: 14] |
| `non_biz_interest` | 업무무관 가지급금이자 | number | N | 지급이자 손금불산입액 |
| `foreign_tax_paid` | 외국납부세액 | number | N | 외국납부세액공제 대상액 [cite: 15] |
| `rnd_credit_amt` | 연구인력개발비세액공제 | number | N | 최저한세 적용 제외 항목 |
| `min_tax_base` | 최저한세 적용대상 과표 | number | Y | 감면 전 과세표준 |

### 3.2. 수입배당금 명세 (지주회사 등)
익금불산입 계산을 위한 데이터입니다.

| field_id | field_name_ko | data_type | required | description |
| :--- | :--- | :--- | :--- | :--- |
| `investee_corp_name` | 피투자법인명 | string | Y | |
| `is_listed` | 상장여부 | boolean | Y | 상장/비상장에 따라 공제율 상이 |
| `holding_ratio` | 지분율 | number | Y | 보유 주식 비율 (%) |
| `dividend_amt` | 배당금수익 | number | Y | 수령한 배당금 총액 |
| `exclusion_rate` | 익금불산입률 | number | Y | 법인세법에 따른 적용률 (30~100%) |

### 3.3. 기납부세액 상세 (법인세)

| field_id | field_name_ko | data_type | required | description |
| :--- | :--- | :--- | :--- | :--- |
| `prepaid_interim` | 중간예납세액 | number | N | 8월에 미리 납부한 세액 |
| `prepaid_interim_date` | 중간예납 납부일 | date | N | |
| `withholding_tax` | 원천납부세액 | number | N | 이자소득 등에서 원천징수된 법인세 |
| `occasional_tax` | 수시부과세액 | number | N | |

---

## 4. [종합소득세 전용] 상세 데이터 스키마

개인사업자 및 근로소득자 경정청구를 위한 별지 제40호 서식 기준입니다 [cite: 16, 17].

### 4.1. 소득유형별 금액 (종합소득 합산)

| field_id | field_name_ko | data_type | required | description |
| :--- | :--- | :--- | :--- | :--- |
| `inc_interest` | 이자소득금액 | number | N | 필요경비 없음, 총수입금액 |
| `inc_dividend` | 배당소득금액 | number | N | Gross-up 적용 전/후 확인 필요 |
| `inc_business` | 사업소득금액 | number | N | 총수입금액 - 필요경비 |
| `inc_wage` | 근로소득금액 | number | N | 총급여 - 근로소득공제 |
| `inc_pension` | 연금소득금액 | number | N | 총연금액 - 연금소득공제 |
| `inc_other` | 기타소득금액 | number | N | 필요경비 차감 후 금액 |
| `fin_inc_global_tax` | 금융소득 종합과세여부 | boolean | Y | 이자+배당 > 2,000만원 여부 |
| `rental_inc_tax_type` | 주택임대 과세방식 | string | N | `SEPARATE`(분리과세), `GLOBAL`(종합과세) |

### 4.2. 사업소득 보정 (개인사업자 경정청구 핵심)

| field_id | field_name_ko | data_type | required | description |
| :--- | :--- | :--- | :--- | :--- |
| `book_type` | 기장의무 | string | Y | `DOUBLE`(복식부기), `SIMPLE`(간편장부), `ESTIMATE`(추계) |
| `joint_biz_ratio` | 공동사업자 지분율 | number | N | 공동사업자인 경우 본인 지분율 (%) |
| `missing_expense_amt` | 필요경비 누락액 | number | N | 경정청구 시 추가할 경비 (예: 대출이자 누락분) |
| `adjust_income_deduction`| 소득공제 조정액 | number | N | 노란우산공제 등 한도 초과/미달 조정 |

### 4.3. 인적공제 및 소득/세액공제 상세

| field_id | field_name_ko | data_type | required | description |
| :--- | :--- | :--- | :--- | :--- |
| `dependents` | 부양가족 목록 | array | Y | `[ {relation: "부모", name: "...", res_id: "...", is_disabled: false} ]` |
| `deduct_pension_ins` | 연금보험료공제 | number | N | 국민연금 납부액 (전액 공제) |
| `deduct_med_exp` | 의료비세액공제 | number | N | 총급여 3% 초과분 등 계산용 입력액 |
| `deduct_edu_exp` | 교육비세액공제 | number | N | 본인/부양가족 교육비 |
| `deduct_donation` | 기부금세액공제 | number | N | 법정/지정 기부금 구분 입력 |
| `credit_rent_monthly` | 월세세액공제 | number | N | 총급여 7천만원 이하 근로자 등 해당 |
| `credit_pension_account` | 연금계좌세액공제 | number | N | 연금저축 + IRP 납입액 |

---

## 5. 데이터 검증 및 시뮬레이션 로직 참고사항

본 데이터 스키마를 활용하여 시뮬레이터를 구현할 때 다음 로직을 반드시 포함해야 합니다.

1.  **청년 연령 판정:** `employees.res_id`를 기반으로 귀속년도(`filing_year`) 기준 만 15세~34세 여부를 판별하되, 군복무기간(최대 6년)을 차감하는 로직을 적용해야 합니다 [cite: 8].
2.  **최저한세(Minimum Tax) 계산:** 법인세 및 개인사업자 감면 시뮬레이션 시, `tax_base_corrected` 기준으로 최저한세를 재계산하여, 공제 가능한 세액의 한도(`min_tax_base`)를 검증해야 합니다 [cite: 18, 19].
3.  **사후관리 위배 체크:** 고용증대세액공제를 받은 후 인원이 감소(`employees` 배열 비교)한 경우, 추가 납부세액이 발생할 수 있으므로 이를 경고하거나 계산에 반영해야 합니다 [cite: 20].
4.  **중복 공제 배제:** 동일 자산에 대해 `investments` 데이터 처리 시 통합투자세액공제와 중소기업특별세액감면 등 중복 적용 배제 규정을 체크해야 합니다 [cite: 21].

이 스키마는 REST API의 Request Body로 사용하거나, 데이터베이스(MongoDB 등)의 Document 스키마로 직접 활용할 수 있습니다.

**Sources:**
1. [bizforms.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQE60YuCc7tDSqD2OTSHEUG58kKcMrDm0_Zs2vz1Bz_wTl2y78aalcsil4JNSzzh2Xkz6ELHDM1LRf9iLOBoMr7u1pLAWKLej_VIvXVQ75G4aVWXpydUveyWZv7AlIcnW0eHgY7TTt0cnZDiCdtpvjzjOtwAyItxAcVv4B1DJa-xx56Aysd8d5TQ)
2. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHr1GoYvKnMiFP4DSjvKJTIGWWB0sSgrgVWAow-EgqEcOaYfRI6dABZLoExHTdJ_tjZi8WyAyeAV13MHGomWXFtpCzoc0l0hJKo2aE7JnYDIRIP-4Oul0vgSB6oRziSwjIcGX0UkeU4dSaQH36IzSYH5yZlvX3ZWC_GRy1p2bqG2S4d_EJwd_BlsF7SxZQXod6sXcI7o2LDkIiLL-sCOkcK57hurJ4tIT_EOnBmy5YkeyaIRq6fxAy5otem7BcNU3OBbNv9TvBHnoUdQb9R5joQib4YT0WZE98VjbraBmRKYp8kQ9T89h2dk208VggyJk_Glf7DLN90Nqi8UQ_p91meFpAIZr08_DhHlxcKzXJ_95B99sCLZdKHaH_fFDuOCWbd0bhW1ePiA9rwfNtLTpOf8OXgA84ZuAq-qet2uljJFEgb4jVn6vj-Wtes2g==)
3. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGlxrJJ9UdGqSIj5IEikkLv6tIwAttdFZ8dh6Km24TX6wS7F-_uS5fRvXH5Bn-6xP4OJEHwOreb-bVzSaOhOm_ZX8Qo-moRD0u5XqWICcYU4CawKokk9sq448jkrKTL4s45__uJqkQivTPKPGRR-3KImvoB3TAVAl3XX2l2AEUMPeo2i7050w3Q3BevVgtNx8jVKPxayeikzu1agFseCRzTPJ3NBC-UM4d5wlIqGXjvO85NTZLFMguesCE4mz5Br39ItJ2083qBdg7NGXMNvSXm3w6dMU2JzWU9Lo7xILdVaBtc8uQdhds8ypXelNI9HkdlXnqpJ4n1RZ-4eIqz1-Xv9t_zki1VmFPpIFQEG4PvYMA-HKQAqlAlWBBpPQTAjEXuuJcEklrj0LQ1NueRK41NJ5ib4TV8odgwV3K5HR40n7ojVCNfSztGfjCVcQ==)
4. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFGecB3ZxB0vUq81wWndMLUfHHrJwiwy1212LY6sWXpevFDFuKF7jAi5HMzyWwrCigEXgxoT82zlOmgUkm1-ZF-7zAjvK3XqF9I1no0lVOwIvZrsAxOCAeh8vaiZ3T3x_A2SX_Ath8omhF9kOAzfr6Y6DYFTynOs0G_Jm7r8aaI-Dhj9fH-yfn3tDnpof7N_1kMjw-DnGixstW6q9PDdYE5nAUzr89qX7oR2_xM6ma5PK2L1EdpjelFDsyfn2uXnSVPZWXpfbN4h24MMuAI5Hk4Wd9EFR6dXsO45TSjRG2aLa_IOw5yZW9GdVBld_ogyowmlGnkdLIgr4dtAOn8RVE-T6J7g3OSrFGQ2uwXau-CjYpkbfLFQkvUWWg=)
5. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEDYr1A3i31vsaZx0utSenPcTiJZ7n0Wwy1P0U5Yf-NN1OkF5QsxOBMSBRDJTXuorxU4ypqO6OUr1CcAaRdNxXx956nYD0-2CEHOUi2VBWeqD-xekGW63crPKn14eyWaFQJ)
6. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHplRb-Gt18OqDr3o7Sift5TsIrHs4cqe8_ETJdFYyWdq_wCCkTHOx1EVAT0OkL4vkn76_g7O8qun-z69W11deisx1C5XTGFrzJNd2YMGp-g4N0YcO9R_gLahFcOlEuA0rbs6Z3ZR4wTuBWAI4it14dG3OQ45Au4DLcVANI_Tm134TrlnFwtRQpKK0_T7UXwMY5emXEE1cQd342KfvJHn-VVecOrqEidfLleRA4tFI9U9a-LZowJAX9pSUXlFblYF3XKA2ak3c5mhMefHBaZBdHhOSt_fnTwR3v8DFHqk7RraqrQf_wHf-9AW1VFojjFsG6JepA64nF8bxEU_ptHfCxeEvdgo-bQvdx7WtK1efdUdCQBbnt58Zhob8V2UmncU-r99PEFOsjMj-yQoLBAUq7Ukj_q4wNpx2H6yrY3katlumVBRLW4NuHuFFmGfufbn0cdsLC8sjGhp0uFOrHM0pmU-gJU8sucHJWw0RA7sKoaXeda44aIR8jBSielTjGzASSJmTrFhzEtYVXYX01MA==)
7. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGN0bvmqpfluinMtCrKw1VR898gdYYKA6EShRsyyhyxFFTb9wwtDdq75-O57vXm5QLOZslM6UmHG44UhDPql1APP3mKIFD3eUU9P7-3Vlm4z2C7I1VQtYbi9QH4JMhvkY50BOPaqmyg-FCYV9Y=)
8. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGRVMOpi0u1vaem1vgjjMGF3x5LP6JeCl8_zhBy07SpM0J3tGdXY06iVkZQ-SvJY_z1b3re_C9DMz6SkuYfAolwX1OMOQoPS7dsoXxE00IRDu051A==)
9. [daum.net](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEewrBFdRtRff_cVDoEsjiELe4KgXQDqeZX0oaiuBUuqBe-jIf6WoBscn8jpEG3EirLPP6EY4ewDVoXEM7CpgCIX4ezLwm9JkrRnS8vGUoy4Yfc0Vj79ccxShVGTSk=)
10. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFPMLH3GNtgQDqVDwdiXNaXs42LfVwGMfrOi7n8_SAT6Om6nF6niNfmVLLhw0jAlipDltbOU0-Pe1hYMXWsjYt9rz68cG1B9HAT6XVN5JH4vdehhc_peTeCbTl4C332gDeu)
11. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGE2vdst_m53bJKdLi1bhxeedFqX01101DMEkLmKCDxVCXZePWjlwAp_KG50I2lbCwACqKcjIp_lDtMZvNioYlbu3ix_x8_d9RV0-B6K5TGHKKkro7OpnNBWqGauX17LHNZXrhKYGAEfmSDGPOeyjpDfPHhMk2SkI8gUpbLL44CRuGc5kM20ki_NPm_O-NKlKgVqtDjaGVkB3-TZsHzq10CebeG6ejjeZxyQ9XJdmSiwxkUfkyBDwv_tql_-pRzlHT-o85mIAWYjRneKEvb37O23Q_J8h790sSxv2a4yHE9eC3y4CX8bf2AZZ4H0mSNgPb412PcRaG79cGxIsfh7jOr0fQfUicEG--CTSLdemJ5Ij4EDz16anjiyv_5gVnrx4xMcXnHPGStNo-sJ7ZYqW7Ye6Pk7xA3iEphcdNq)
12. [samili.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFq-DEi_JpwZkCYCoAa8k2yufRNco4vQVZQ2eiMmu7Wl4X59gXe7c0uXhMvyI5V631aBznTndqBSZJJB0UEmNsF4oN-0gCpOvekivhJyP5-wRFy0jxPnJAwG6KynbTlvHEF10YLK0uBAZ3gXXb3)
13. [bizforms.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGfYdA69HG8ZC5epdHfX21kqxWpgmLrtrSrY_rY19XfmERP_sAG1BRz7m6v0vYJKTkfke3v4rLuYeBhMwVkIGDz9o-Y1z_vwtWaggctv5hhIWD6qCoCrQfbS0zQSbfNOsIsh6nW0FkoEnZtwHPIhFnHw0eDhB6GKhcRTr5MslH2D8zdfVRXtdYv1UmC86sU3rtxMoA7rqg2xgGrcDrqREqVOm0gV9aIfSp7QfcEON767CeOVXPauUYIHjmyhQIc0FWA9P8lCDwuCxElBQZ87a0mGgTaVBsJ4-rpmGzYq80JfLX0Nz_R-fUkHMYQ3w==)
14. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQH6jEstHu7jVz5dA-fOMaKLxYQVJlz51az4RqIDJSmI-ffH4OgqrnNidCQ1T-s2MmVyaRqRKIQJJ4l1tjBEx8oms84yukYBeuv7c-jwVwKyzFEFZlcQhm25Cex1Jl9HHEy6NFKsdsvnUIT7Lm-h1rTuhm3ggvnW2xifE55ZPJzACA==)
15. [yesform.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHJnmk0NlmJ0IEKE_cVD9dLwMujEvHN-6FfZxolJseJSLXE5oNnmL0cP1jJlb9WBamz4da614SozYshLp92NoQE8MpKqOuURK8Pz6Cyzy9-H3x44FS3uIFi33jAxQ99sMosiBE=)
16. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFWRuQ4mmiv35sCPnnaics9zcpqEshzNjeFiHVinN8X4vTQcmWbSHCiLLzovYDSNGq6balSyHmMpFhTm2a8h5hLiTr8Y0j5drOTAUmXO1NACfFcG1Tdrro8Q7CmAyH7QQHIs5CuJb21Rw_xUdIilsI401mcyLC3NHS8glRNs9itpQ3ZYC4mWFkCy5aZ_0_33hweb28tsAz8jh36nkvtmvJRxil26TQzAMN7TlOQs4Ib__hrbEtw1CRAMrWH2aY-5RikqGga3xUdsed8fCc8IRG0yd-43dgOPdhuyYWjqLCJF5r_Kv_SJCxxEwPvrvB7ZXHmjFjLSz_laVSR9Guc8riRS_PSklUlRp_rzBWezq_UBRMCmZ84yc0LbPmpC8lp5KSCrazD8jKwQjfDOEunQtyPs6lVQn0aRcErthqtIsdbGzKFbxfdSeyrLiuAYCYWuupVuVfMvT6yhwxV_SGWkzi4nLkAJsI7O5mKaHCe6Q8K1jMeEKhZIzUEP0Bvnvzd3Oj4kom4acEQxhQK5bc8sL_ZG8VzAUGXuNyedu2fn3DY5MpKrpFGzWuru7wSwLbBPnxUCCf8gXPXcbEf7AgFW4KcGWa5y-LXqjAdEy_d)
17. [samili.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFLxch_s2vaToNst5tfvUrTpHLm_HaefA6_kEbhPm5cxcYVr5P4j9x7mos5-FfR4DgM7wd2Idm4QMe7nSV0rGMeUQ9xS1njKsfXbf0qp0sY3HKktnXWsBM1bTHB2_48qRZuuUIqUs6uYaMKhiFfOFs=)
18. [calculate.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEyhNWSFchIwqWOMIU1Un8QwESlyRrwa_DSrks-NMP3NBbmWc0j54cEjwsQA5MXO8aklNYYJrVAAsnPlCplabpRYwodCYoHjmGlDJzV9feYDeX5uVVkm8m7Inb2Yd1lrJgtKobhLB3bLPwCtUK1nH4Yb2EbSZquazkQkWWUC1fnknQLwlJFsCTMw8XSL12Fhls951F9gXkCS5jMcaB9)
19. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGuihIu5a1_fwaLN60oQRqKU89R00rDyf_ggJH3w5PM-AaK4Hbyt3kZf--eW-toye3i11pOsWqvd-GIpYJbtQVpzFFVExQpzGi5skTvPjE4YopLqKec8PKgaR8ZvHmt4Fgf)
20. [korcham.net](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFyHvhYknIEgVGc801usa1L5EXVKYQtbfM78ycuhnEDZiUKvpua6Y0i-WnqlLFdZH2iZZ1bz_XrXl5Z5Ds4nGgl7Lly7WqZQGxki8xtL2kWymHXiJmCk0tgp5O2cofoLWNjNzruPFbRg5z_8gO30SNJ8pslonD23rVdD1XMv9OqArBiIpuO14bBhMpmzmH4DOha2QMlsDD9IzqaKdUOgItHTEsBlHLXP9645TntzWfraR7QpJW_29K6ZDpwAPnRujkVU1BfLg==)
21. [tistory.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQF2k85PZUWcrm3dI8qGwK3i98o6tvkzPQvAqs6lR7tBWUE-4nZIpRga3m7YL1j-tGId9NwmHaW_oDCwGOJtVRlcQa_j5GIFHE2qOU-PQGwCbducApvCNW25EiLMA-67169Tc1_XlMQNcDiWNI2HNTEpMmYMBnLGdytbGlMSBdUZBN6vAAqN5GP1ozu5Hx9XbTsR_xq8e-JH8wl2cuUfuGi1fRhJzgHppQ9FkVXrzhu2W3VDhSTh1UACne7yZSHzoYQXifsLB5HoL7Op6lIcea0U8AGRoHg-nPy7bY9njrNUDFibtv41BshSYm4EYbrEwp_PdqnhG60O3RCfmSe1MYg656Hw5LelYw3DBazg9yG5)

# 한국 세금 환급금(경정청구) 계산 시뮬레이션을 위한 법령 기반 데이터 필드 분석 보고서

## 1. 서론 및 주요 요약

본 보고서는 2025년 12월 2일 국회 본회의를 통과한 세법개정안(법인세법, 조세특례제한법 등)과 최신 법령정보(www.law.go.kr, taxlaw.nts.go.kr)를 바탕으로, 한국 세금 환급금(경정청구) 시뮬레이션 시스템 구축에 필요한 입력 데이터 필드를 심층 분석하였다.

특히 2025년 세법개정안은 법인세율의 전 구간 1%p 인상(2026년 귀속분부터 적용), 국가전략기술의 범위 확대(AI 포함), 통합투자세액공제 및 통합고용세액공제의 구조 개편 등 시뮬레이션 로직에 중대한 영향을 미치는 요소들을 포함하고 있다. 따라서 시뮬레이션 엔진은 귀속년도(2020~2026)에 따라 상이한 세율과 공제율을 적용할 수 있도록 동적인 데이터 필드 설계가 필수적이다.

### 핵심 변경 사항 요약
*   **법인세율 인상:** 2026년 1월 1일 이후 개시하는 사업연도부터 법인세율이 과세표준 구간별로 1%p씩 인상되어 10%~25% 체계로 환원된다 [cite: 1, 2].
*   **국가전략기술 확대:** R&D 및 통합투자세액공제 대상인 국가전략기술에 '인공지능(AI)' 분야가 신설되고 관련 기술이 구체화되었다 [cite: 3].
*   **통합고용세액공제 개편:** 2025년 이후 지원 대상 및 사후관리 요건이 변경되며, 특히 중견·대기업에 대한 '최소고용증가인원수(5명/10명)' 요건이 신설되었다 [cite: 3, 4].
*   **창업중소기업 감면:** 2026년 이후 수도권 과밀억제권역 외 지역의 청년 창업 등에 대한 감면율 조정 및 생계형 창업 기준 수입금액 상향(1.04억 원) 등이 포함되었다 [cite: 5, 6].

---

## 2. 공통 기본 정보 및 과세표준 (점검 0 ~ 점검 2 관련)

세액공제 및 감면 시뮬레이션의 기초가 되는 납세자 정보 및 과세표준 산정을 위한 필드이다. 2026년 세율 인상을 반영하기 위해 귀속년도 입력이 필수적이다.

### 2.1. 법인세법 제55조 및 소득세법 제55조 (세율)

2025년 세법개정안에 따라 2026년 귀속분부터 법인세율이 변경되므로, 시뮬레이터는 `귀속년도`에 따라 분기 처리된 세율 테이블을 호출해야 한다.

**[표 1] 기본 정보 및 세율 적용을 위한 입력 필드**

| 필드명 (Field Name) | 데이터 타입 | 필수여부 | 설명 및 시뮬레이션 로직 | 법적 근거 |
| :--- | :--- | :--- | :--- | :--- |
| **귀속년도** (Attribution Year) | Integer (YYYY) | **Y** | 세율 및 적용 법령 버전을 결정하는 기준 키 (예: 2024, 2025, 2026). 경정청구 가능 기간(5년) 판단의 기준. | 국세기본법 제45조의2 |
| **기업형태** (Company Type) | String (Enum) | **Y** | 개인사업자 / 법인사업자 구분. | 법인세법 제1조, 소득세법 제1조 |
| **기업규모** (Business Scale) | String (Enum) | **Y** | 중소기업 / 중견기업 / 대기업. 조특법상 감면율 적용의 핵심 기준. | 조특법 시행령 제2조 |
| **수도권 소재 여부** (Is Capital Area) | Boolean | **Y** | 수도권 과밀억제권역 내/외 구분. 감면율 차등 적용(50% vs 100% 등)의 핵심 변수. | 조특법 제6조, 수도권정비계획법 |
| **과세표준** (Tax Base) | Long (Currency) | **Y** | 산출세액 계산의 기초. | 법인세법 제13조 |
| **산출세액** (Calculated Tax) | Long (Currency) | **Y** | 각종 공제감면의 기준점. (최저한세 계산 전 세액) | 법인세법 제55조 |

**[참고] 법인세율 로직 변경 사항 (2026년 귀속분부터)** [cite: 1, 2, 7]
*   **2025년 귀속까지:** 2억 이하(9%), 200억 이하(19%), 3000억 이하(21%), 3000억 초과(24%)
*   **2026년 귀속부터:** 2억 이하(10%), 200억 이하(20%), 3000억 이하(22%), 3000억 초과(25%)

---

## 3. 창업중소기업 등에 대한 세액감면 (조특법 제6조)

창업 초기 기업에게 가장 강력한 혜택(50%~100% 감면)을 부여하는 조항으로, 2025년 세법개정을 통해 생계형 창업 기준 완화 및 감면율 조정이 발생했다.

### 3.1. 주요 개정 및 시뮬레이션 로직
*   **청년의 정의:** 만 15세~34세 (병역 이행 기간 최대 6년 차감). 2026년 개정안에서도 청년 연령 기준은 유지되나, 수도권 내 감면율이 세분화될 수 있음 [cite: 6].
*   **생계형 창업:** 연간 수입금액 기준이 기존 8천만 원에서 **1억 400만 원**으로 상향 조정됨 (2025년 개정 반영) [cite: 5, 6].
*   **감면율 매트릭스:** 창업 지역(수도권 과밀억제권역 내/외), 청년 여부, 업종에 따라 50%~100% 차등 적용.

**[표 2] 조특법 제6조 시뮬레이션 입력 데이터 필드**

| 필드명 | 데이터 타입 | 필수 | 시뮬레이션 적용 로직 | 법적 근거 |
| :--- | :--- | :--- | :--- | :--- |
| **창업일자** (Startup Date) | Date | **Y** | 창업일이 2025.12.31 이전인지 이후인지에 따라 적용 조문 변경. 감면 기간(5년) 카운트. | 조특법 제6조 제1항 |
| **대표자 생년월일** (Rep Birth Date) | Date | **Y** | 창업 당시 연령 계산 (청년창업 감면 적용 여부 판단). | 조특법 시행령 제5조 |
| **병역이행기간** (Military Service) | Integer (Month) | N | 창업 당시 연령에서 차감하여 청년(만 34세 이하) 요건 충족 여부 재판단. (최대 6년) | 조특법 시행령 제5조 |
| **창업 지역 코드** (Region Code) | String (Enum) | **Y** | 수도권과밀억제권역 / 수도권(과밀억제권역 외) / 비수도권 / 인구감소지역. | 조특법 제6조, 국가균형발전법 |
| **감면 대상 업종 여부** (Target Industry) | Boolean | **Y** | 조특법 제6조 제3항에 열거된 업종(제조업, 건설업, 통신판매업 등) 해당 여부. | 조특법 제6조 제3항 |
| **연간 수입금액** (Annual Revenue) | Long | **Y** | 생계형 창업중소기업 판단 (개정: 1.04억 원 이하 여부 확인). | 조특법 제6조 제5항 [cite: 5] |
| **최초 소득 발생 연도** | Integer (Year) | **Y** | 감면 기간(5년)의 기산점 설정. | 조특법 제6조 제1항 |

---

## 4. 중소기업에 대한 특별세액감면 (조특법 제7조)

가장 보편적인 감면 제도로, 업종·지역·규모에 따라 5%~30%의 감면율을 적용한다. 2025년까지 유효하며, 2028년까지 연장 논의가 진행 중이다.

### 4.1. 주요 체크포인트
*   **감면한도:** 1억 원 (상시근로자 감소 시 한도 축소 페널티 존재).
*   **중복배제:** 창업중소기업 감면(제6조)과 중복 적용 불가 [cite: 8]. 통합투자세액공제 등과는 중복 가능.

**[표 3] 조특법 제7조 시뮬레이션 입력 데이터 필드**

| 필드명 | 데이터 타입 | 필수 | 시뮬레이션 적용 로직 | 법적 근거 |
| :--- | :--- | :--- | :--- | :--- |
| **주업종 코드** (Main Industry Code) | String | **Y** | 도소매, 의료업, 제조업 등 업종별 감면율 매트릭스 매핑 (조특령 별표6 참고). | 조특법 제7조 제1항 |
| **사업장 소재지** (Location) | String | **Y** | 수도권 / 비수도권 구분에 따른 감면율(예: 도소매업 수도권 10%, 지방 10~30%) 결정. | 조특법 제7조 제1항 |
| **기업 규모(소/중)** (Size Detail) | String | **Y** | 소기업 vs 중기업 구분에 따라 감면율 차등 (소기업이 더 높음). | 조특법 시행령 제6조 |
| **감면 대상 소득** (Taxable Income) | Long | **Y** | 전체 과세표준 중 감면 대상 사업에서 발생한 소득 비율 계산. | 조특법 제7조 |
| **상시근로자 수(당기)** (Emp Count Current) | Integer | **Y** | 1억 원 한도 적용 여부 및 고용 감소 시 한도 축소(1억 - 감소인원×500만원) 계산. | 조특법 제7조 제3항 |
| **상시근로자 수(전기)** (Emp Count Prior) | Integer | **Y** | 당기 근로자 수와 비교하여 한도 축소 여부 판단. | 조특법 제7조 제3항 |

---

## 5. 연구·인력개발비 세액공제 (조특법 제10조)

2025년 세법개정의 핵심으로 **국가전략기술** 범위가 대폭 확대되었다.

### 5.1. 주요 개정 및 시뮬레이션 로직
*   **기술 분류:** 일반 / 신성장·원천기술 / 국가전략기술.
*   **국가전략기술 확대:** 기존 반도체, 이차전지, 백신, 디스플레이, 수소, 미래이동수단, 바이오의약품에 더해 **AI(인공지능)** 분야가 추가됨 [cite: 3, 4].
*   **공제율:** 국가전략기술 중소기업의 경우 최대 40~50% 공제율 적용.
*   **계산 방식:** 당기분 방식(당해 지출액 × 공제율)과 증가분 방식(증가액 × 고율 공제) 중 선택.

**[표 4] 조특법 제10조 시뮬레이션 입력 데이터 필드**

| 필드명 | 데이터 타입 | 필수 | 시뮬레이션 적용 로직 | 법적 근거 |
| :--- | :--- | :--- | :--- | :--- |
| **R&D 비용 구분** (R&D Category) | Enum | **Y** | 일반 / 신성장·원천 / 국가전략기술(반도체, AI 등). 카테고리별 공제율 매핑. | 조특법 제10조 제1항 |
| **당기 R&D 지출액** (Current Expense) | Long | **Y** | 공제 대상 금액 기초 데이터. | 조특법 제10조 |
| **전기 R&D 지출액** (Prior Expense) | Long | **Y** | 증가분 방식 계산 시 필요. | 조특법 제10조 |
| **직전 4년 R&D 연평균** (4yr Avg Expense) | Long | N | 증가분 방식 선택 시 비교 기준(직전 4년간 연평균 발생액보다 당기가 커야 함). | 조특법 시행령 제9조 |
| **매출액** (Revenue) | Long | **Y** | 신성장·원천기술 및 국가전략기술 공제 한도 계산 및 추가공제율 산정 기초. | 조특법 제10조 |

---

## 6. 통합투자세액공제 (조특법 제24조)

2025년까지 **임시투자세액공제(Temporary Investment Tax Credit)** 기조가 유지되거나 특정 기술(국가전략기술)에 대한 높은 공제율이 2029년까지 연장되는 등 투자를 강력히 유인하고 있다.

### 6.1. 주요 개정 및 시뮬레이션 로직
*   **기본공제율 상향(임시투자세액공제):** 2024~2025년 투자분에 대해 일반 자산도 공제율 상향 적용 가능성(법령 개정 추이 반영 필요).
*   **국가전략기술 투자:** 반도체, AI 시설 투자 시 중소기업 최대 25~35% 공제 [cite: 9].
*   **증가분 추가공제:** 직전 3년 평균 대비 증가분에 대해 10% 추가 공제(임시투자세액공제 적용 시 한도 상향).

**[표 5] 조특법 제24조 시뮬레이션 입력 데이터 필드**

| 필드명 | 데이터 타입 | 필수 | 시뮬레이션 적용 로직 | 법적 근거 |
| :--- | :--- | :--- | :--- | :--- |
| **투자 자산 유형** (Asset Type) | Enum | **Y** | 기계장치(일반) / 신성장사업화시설 / 국가전략기술시설(AI, 반도체 등). | 조특법 제24조 제1항 |
| **당기 투자 금액** (Investment Amt) | Long | **Y** | 기본 공제액 계산 (투자금액 × 기본공제율). | 조특법 제24조 |
| **직전 3년 연평균 투자액** (3yr Avg Inv) | Long | **Y** | 추가 공제액 계산 ((당기 - 3년평균) × 10%). | 조특법 제24조 |
| **투자 완료일/취득일** (Acquisition Date) | Date | **Y** | 2025년 임시투자세액공제 적용 기간 해당 여부 판단. | 조특법 시행령 제21조 |
| **중소기업 졸업 유예 여부** (SME Grace) | Boolean | N | 중소기업 졸업 후에도 일정 기간(3~5년) 기존 중소기업 공제율 적용 여부 체크 [cite: 10]. | 조특법 제24조 |

---

## 7. 통합고용세액공제 (조특법 제29조의8)

2025년은 기존 고용증대세액공제 체계에서 **통합고용세액공제** 체계로 완전히 전환되는 시점이며, 2026년부터는 '계속고용 지원형'으로 구조가 전면 개편된다.

### 7.1. 주요 개정 및 시뮬레이션 로직
*   **구조 개편(2025년 개정안, 2026년 시행):** 기존 '3년간 지원' 방식에서 고용 유지 기간에 따라 인센티브가 강화되는 구조로 변경 [cite: 3, 11, 12].
*   **최소고용증가인원수 도입:** 중견기업(5명), 대기업(10명) 이상 증가 시에만 공제 적용 [cite: 3, 4].
*   **청년 연령:** 만 15세~34세 유지. 계약 체결 시 34세 이하면 이후 연령이 초과되어도 일정 기간(4년) 청년으로 간주하는 규정 신설 [cite: 4, 13].

**[표 6] 조특법 제29조의8 시뮬레이션 입력 데이터 필드**

| 필드명 | 데이터 타입 | 필수 | 시뮬레이션 적용 로직 | 법적 근거 |
| :--- | :--- | :--- | :--- | :--- |
| **총 상시근로자 수(당기/전기/전전기)** | Integer | **Y** | 전체 고용 증가 여부 판단 및 2026년 계속고용 지원형 로직 적용을 위한 과거 데이터. | 조특법 제29조의8 |
| **청년 등 상시근로자 수** | Integer | **Y** | 청년, 장애인, 60세 이상, 경력단절여성 등 우대 공제 대상 인원 산출. | 조특법 제29조의8 제1호 |
| **청년 외 상시근로자 수** | Integer | **Y** | 일반 공제 대상 인원 산출. | 조특법 제29조의8 제2호 |
| **기업 소재지** (Location) | Enum | **Y** | 수도권(850만/1450만) vs 지방(950만/1550만) 공제액 차등 적용 (중소기업 기준). | 조특법 제29조의8 |
| **신규 채용자 명세** (New Hires List) | List<Object> | N | 개별 근로자의 근로계약일, 연령 등을 입력받아 '청년' 유지 기간(4년) 자동 계산 로직 적용 [cite: 13]. | 조특령 제26조의8 |
| **최소 증가 요건 충족 여부** (Min Increase) | Boolean (Logic) | **Y** | (중견/대기업) 증가 인원이 5명/10명 미만일 경우 공제액 0 처리 [cite: 3]. | 조특법 제29조의8 (2025 개정) |

---

## 8. 중복지원 배제 및 최저한세 (조특법 제127조, 제132조)

시뮬레이션의 마지막 단계에서 공제세액을 확정짓는 '필터링' 단계이다.

### 8.1. 중복지원 배제 (제127조)
*   **원칙:** 동일한 투자 자산에 대해 제24조(통합투자)와 기타 세액공제 중복 불가. 동일한 과세연도에 제6조(창업)와 제29조의8(통합고용) 중복 불가(일부 예외 존재) [cite: 14].
*   **매트릭스:** 시뮬레이터는 사용자가 선택한 감면 항목 간의 충돌을 감지하고, 더 유리한(Max Benefit) 조합을 제시해야 함.

### 8.2. 최저한세 (제132조)
*   **세율:** 중소기업 7%, 일반법인 10%~17%.
*   **적용:** 각종 감면 후 세액이 최저한세보다 적을 경우, 최저한세만큼은 납부해야 함. (단, 중소기업의 사회보험료 세액공제 등 일부는 최저한세 제외 [cite: 15]).

**[표 7] 제한 및 한도 계산을 위한 입력 필드**

| 필드명 | 데이터 타입 | 필수 | 설명 및 로직 | 법적 근거 |
| :--- | :--- | :--- | :--- | :--- |
| **감면 전 산출세액** | Long | **Y** | 최저한세 비교 기준. | 법인세법 제55조 |
| **최저한세 제외 공제액** | Long | **Y** | 최저한세 적용을 받지 않는 세액공제 합계 (예: 중소기업 R&D 등). | 조특법 제132조 |
| **최저한세 적용 공제액** | Long | **Y** | 최저한세 적용 대상 공제 합계 (예: 통합투자세액공제 등). | 조특법 제132조 |
| **중복 배제 선택** | Enum | **Y** | (창업감면 vs 통합고용) 등 상호 배제 항목 중 사용자 선택 값. | 조특법 제127조 |

---

## 9. 기타 법령 및 경정청구 (국세기본법)

### 9.1. 사회보험료 세액공제 (조특법 제30조의4)
*   **일몰:** 2024년 12월 31일로 종료되나, 경정청구(과거 5년치) 시뮬레이션에는 필수적임.
*   **로직:** 청년 100%, 기타 50% 공제. 최저한세 미적용 특례 확인 필요 [cite: 16].

### 9.2. 국세기본법 제45조의2 (경정청구)
*   **기한:** 법정신고기한 경과 후 **5년 이내**.
*   **후발적 경정청구:** 소송 판결 등 사유 발생 시 안 날로부터 3개월 이내.
*   **시뮬레이터 필드:** `최초 신고일자`, `경정청구 신청일자`를 입력받아 `DATEDIF` 로직으로 청구 가능 여부(Valid/Invalid)를 리턴해야 함.

---

## 10. 결론: 시뮬레이터 로직 구현을 위한 제언

2025년 세법개정안은 '복원(법인세율 인상)'과 '전략적 지원(AI, 고용)'이라는 두 가지 축으로 요약된다. 따라서 경정청구 시뮬레이터는 단순히 과거 법령만 적용하는 것이 아니라, **귀속 시점(2020~2026)**에 따라 달라지는 세율 테이블과 공제율 매트릭스를 DB화하여 관리해야 한다.

특히 **통합고용세액공제**의 경우 2023년(신설), 2025년(개정), 2026년(구조 변경)의 로직이 모두 다르므로, 입력 필드 설계 시 '상시근로자 수'를 단순히 연도별 총합으로 받지 말고, **[연도] × [근로자유형(청년/일반)] × [근속월수]** 형태의 상세 데이터를 입력받아 내부 로직에서 연도별 법령에 맞게 매핑하는 방식이 권장된다.

### [부록] 2025.12.2 국회 통과 반영 데이터 구조 (JSON 예시)

```json
{
  "simulation_meta": {
    "target_year": 2026,
    "law_version": "2025_amendment_passed"
  },
  "basic_info": {
    "biz_type": "CORPORATE",
    "scale": "SME",
    "location": "CAPITAL_AREA",
    "tax_base": 500000000
  },
  "employment_data": {
    "2025_emp_count": 10,
    "2026_emp_count": 12,
    "increased_count": 2,
    "youth_count": 1,
    "min_increase_threshold_apply": false
  },
  "investment_data": {
    "tech_type": "NATIONAL_STRATEGIC_AI",
    "invest_amount": 100000000,
    "credit_rate": 0.25
  },
  "tax_calculation": {
    "standard_tax_rate": "applied_2026_rates", // 10~25%
    "minimum_tax_rate": 0.07
  }
}
```

**Sources:**
1. [yna.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHuAuh6zjrfh6rIt399VkOHhuNm_8nB7-OsD4ksD3NJNNtDq9huaDJNCKl43KHYMXKkQwwPT6WIcb18jqeFJr1X65afflAU_7wUtBY2egcjAp7w1uIvZfF3KMq-tcAcV6xMGekB0Q==)
2. [Link](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGhEyzbdCV35Nc78l3IC7OSfQzXBYS4QvIVMcrP1gzrCwPr-harWD6NqzVHaem4C73wz3TMshMeDOJu15ux7HUgd2daHLqykqxFUUYx0os9URwVZn-dQlmyblvv0AK2jUhpQhWFUoB6)
3. [Link](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGve6wRkz24yN5rpSKj2TCXXlkuWkPOdjH5JQUQEpx-K7gZFPofxVXbgJFpd_8rF2f4MOI31qKxiyvTtPu6tUGzWALJOBrDCP0UE1uBYP2qfrktWGyILYrSBZDGyrMWIehjNqkOlU3lcdwkSVgaKL1mJoxe3RrSxC9xywlSYixGQeBqaVpdlX7BtXp8PwEm9sjhjSrtbdMqgTjL9LYkICpHyNeuFcPVG6gb9FkKdO8AN-evXDM=)
4. [shinkim.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFC9NCmh3M56k1cnNciGjUpLY2AAShP2XgjF8mSmZj9h9Fym4vbKqoccP8r8CLM5KdNEbOlDGZRrmduszowga_4wVJSYzov7RO7lIo9not-bwdCUZjE0K4x0TKEqI4KTHMkRx2iZT5Z)
5. [clhs.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFu_tsYf73zhMtC-Pue1OVElHP-5-c5ooxuuEmtjNufXA08hmTopjnCw9nFZafVqtA4dwB1YFMr-W-Kf0GTX1sO1bpa0f5GNDnMEed9aK37uawIRWAfCKxoq_9ydqKXg8NZvrRdq_ypQ-8jupp4cikay4tGDCZmfVZdWFeeE0UjmDpvvz8qgiJtvs3JC8C7DEpjZoys8lS0Tv3bc_VWkw==)
6. [rodemtax.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGDcAVIAbad153xJgb3ShPe8UlFuHWfnn54UGbvkciX0BPRJmvVj3kdHlEWbG36vWK_LerVN4CGeMXYqmDlx0IUuAPi8cRrT45xePrQ8vJ1Gr4nPQ2n3v0UIBbTgQ==)
7. [taxguide.im](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEttYLXQmVGePGJuv5hwEHGlHrw69U-ztGuav6BsUInuApxdGwNhPY_Fyxw2B_bO5zMTx2yGyAphDo6VIxnLYxEug90vE6nFKYyldI3Bjpx-WqIJ7xusJoGiXJU)
8. [samili.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFzr6YnJzN8bsAfHD5zQGmIwqxXzEDGWN2l4Bd0vsGS5Dd27qIN6-8huCp8Q7ZvbrF1qH2DYrcZuQ5xa2CzhYGxT_-lDuGkJuHI9riAq5JJhZkfB8rWS8997Vq1F6uPbvvcFRbStsYm0Gne9P5hJHz7UkrvpKmt_XDMZ1SEBhDXdEDEPRfsNB55zjEOv5gUK56ezE_UR7ln2SDw2UQhDrx9VZJOzWWmJJBksezLjAmGBhYhbRU1siG-l2Sg0XJMGb81AI6uupeO1g1OOkd7n5RTNpM8W8RX60pQnPndg8Vu403CU8459lrkYNQ=)
9. [Link](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQF2AyjdS0if7JoJbjxP8v1LKFFf3tQ6yM_GuOxo9YeCt0NIhoWJZbPMLjInSOxGrzb1KbUg-BQU8Cwqs-L7WyehoiGdKyr71fUEbCKjaMSKn4KPnjFI3kH3xfD1yW_LaF5U19ejoiiVCT5Jowgpu4zvXg0zvxqOfxlAXAWXxJOr-_lpLgZpJxBD1GlETXiygPM=)
10. [moleg.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHyC4K6WbqHcRHFYoUWvm6pcUH2zSmBTx_MBHCjPVAbAwJqZ-yfl5XIrgRh6nrog6cvpOkU546tTAwytpuLl5Vj9kD0oegjk2xSXBjrugZ5fYA9n85zk_AcTkN50z9eTm1eL4UODSUrangS3QhHX3zb7L-NXiVttDvp22lAcqf9MD2Zm2Gwe-gU-FMeUaW9e9TU4-C8RGa6uw==)
11. [moef.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFYuNRtfJHtb-0qcfI5h6AFZf24uGLwasC8xfPpV6XZeY46grnsAwNXz1f-NpkXMqu2DjA43HosX4ac6ZrofTD9noPqt1klTrpUHUWzjR8E2gJOJYwEiW-BSEvvbgQnLcBP_rlmDx2jEQlao8alTznALbZYpQ-zaFGOpxsY20S9Fq0KOXdZdTHkq-NHckXLrkgVgc79Tki39INsw4uElsahtwyTacphcU926qSrl2aSpmnCv6BWYXg=)
12. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEHfYhrd0fCYpAV9txI3A6iP72l40ZNR83k4QJCs6WflrlWN1JENXjiyJDm244PAlydtT0PM9Xag-xs5ncYcS0rD0tYiek0_C2HnG0TfW63Mh35JYsGHOUaapkGvy7DOmY=)
13. [kacta.or.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHDVwcZpaZv5yt2J1dbIa3uOc26jzOiUW82B_4BD_4e84GTGzwEcxBv-Y87JeG_BYQf_QWvHYjfW_dndWgJg7Ea-KtuPcDc6nV7tALOZhTXUTqxZX5dff2Pg6vZWzqbt6jjtNcfDP5OmiKDuh0i0y6gweJz)
14. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEUQ25o5R6cdeUCrOPUMM94bSpl1GKIPdfYmriYAYlEaND3BbsMqP_3LU0K_Z-Uln13FhbE7Q6OBHlrznL6kjhq0RAOVIga1mchdKrs_calURZA8IEWwiwrD-aL70E-8V81MkECK9PJnQ4mPHWXKFR7rI74_WRqAzXwbO_Mzx95c5hR7pORUizlmsTDLTbJY0X8Muwur6ZPEalPIAIQJA==)
15. [bznav.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGpKs5UXVUL0NduOFieexaRBw8WgKamEnOdykGPKVbqyno-cm-7yy6ClZdBfU24UaVA_u3ImwvFMg_-dEWGEfw8nBWGBECNEFjHdJIgC2hi7gVyF94e9ssOV2Y=)
16. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHq0IvaHgZejBkUhI70QvCSYnC8HMgyp4ueJUhQbVXvduSEcpeqR3ahOMRy1MW5bZpDjJvJt4t_5UKjslYTJq7OJa7OOts_rYDioPpW9Q3YLwx6dkovH3VlgHwXMlRSkJx6qy0q91TeyW-8I7OOy735u6mLbtULbn-RnbljuxVxbn0=)

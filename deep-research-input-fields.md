# 한국 세무 경정청구 소프트웨어 및 홈택스 신고서 작성 필수 입력 데이터 필드 조사 보고서

## 1. 개요 및 요약

본 보고서는 한국의 주요 세무 회계 소프트웨어(더존 Smart A, 위하고, 세무사랑 Pro, NICE 등)와 국세청 홈택스(Hometax) 시스템에서 법인세 및 종합소득세 경정청구(환급) 신고서 작성 시 요구되는 구체적인 '데이터 필드'를 심층 분석한 결과이다. 단순한 첨부 서류 목록이 아닌, 실제 환급액 계산 로직(Algorithm)을 구동하기 위해 납세자 또는 세무 대리인이 시스템에 입력해야 하는 정량적·정성적 데이터 항목에 초점을 맞추었다.

경정청구는 당초 신고한 과세표준 및 세액이 정당한 세액을 초과하는 경우, 법정신고기한 경과 후 5년 이내에 관할 세무서장에게 이를 바로잡아 줄 것을 청구하는 제도이다 [cite: 1]. 이를 위해 소프트웨어 및 홈택스 화면에서는 **① 당초 신고 내역(기존 데이터)**, **② 경정(수정) 내역(신규 데이터)**, **③ 과세표준 및 세액 증감 내역(비교 데이터)**, **④ 환급 계좌 정보**가 핵심 입력 체계를 구성한다. 특히 고용증대, 통합투자, R&D 등 주요 세액공제 항목은 별도의 복잡한 부속 명세서 작성을 위한 기초 데이터(월별 인원, 투자 취득일 등) 입력이 선행되어야 한다.

---

## 2. 법인세 환급액 계산을 위한 납세자 입력 항목

법인세 경정청구는 당기순이익에서 시작하여 세무조정 과정을 거쳐 최종 납부 세액을 산출하는 구조를 가진다. 환급액 계산을 위해서는 `법인세 과세표준 및 세액조정계산서`의 수정과 이를 뒷받침하는 `세액공제조정명세서`의 데이터 입력이 필수적이다.

### 2.1. 법인세 과세표준 및 세액조정계산서 상의 입력 필드
이 서식은 법인세 신고의 최종 집계표 역할을 하며, 대부분의 필드는 하위 서식(수입금액조정, 과목별 세무조정 등)에서 불러오지만, 경정청구 시에는 수정된 값을 확정하기 위해 다음 필드들의 검증 및 재입력이 요구된다 [cite: 2, 3].

| 구분 | 주요 입력 필드 (필드명/항목) | 데이터 성격 및 설명 |
| :--- | :--- | :--- |
| **소득조정** | **101. 결산서상 당기순손익** | 재무제표(손익계산서)상 확정된 당기순이익/손실 (당초 신고와 동일해야 함) |
| | **102. 익금산입** | 세무조정으로 인해 증가된 익금(수정 사유가 익금 누락인 경우 수정 입력) |
| | **103. 손금산입** | **(핵심)** 경정청구의 주된 원인. 비용 누락, 추가 공제 반영 시 입력 |
| | **104. 각 사업연도 소득금액** | (101 + 102 - 103) 자동 계산 필드 |
| **과세표준** | **109. 이월결손금** | 과거 15년(19년 이전 10년) 이내 발생한 미공제 결손금액 |
| | **110. 비과세소득** | 법인세법상 비과세되는 소득금액 |
| | **111. 소득공제** | 유동화전문회사 배당가능이익 공제 등 특정 소득공제액 |
| **세액계산** | **121. 최저한세 적용대상 공제감면세액** | **(핵심)** 고용증대세액공제 등 최저한세 적용 대상 공제액 합계 |
| | **123. 최저한세 적용제외 공제감면세액** | R&D 세액공제(중소기업) 등 최저한세 배제 대상 공제액 |
| | **133. 감면분 추가납부세액** | 사후관리 위반 등으로 인해 토해내야 하는 세액 (차감 요소) |
| | **128. 기납부세액(중간예납/원천징수)** | 이미 납부한 세액 (당초 신고서의 기납부 세액과 동일하게 입력) |

**소프트웨어 작동 방식 (더존 Smart A / 세무사랑 Pro):**
*   `법인세 과세표준 및 세액신고서` 메뉴에서 신고구분을 '2. 수정신고' 또는 '4. 경정청구'로 선택한다 [cite: 4, 5].
*   당초 신고 데이터를 불러온 후, `103. 손금산입`이나 `121/123. 공제감면세액` 필드를 수정된 금액으로 덮어쓴다. 이때 수정 전 금액은 붉은색, 수정 후 금액은 검은색으로 표기되어 비교가 가능하다 [cite: 4].

### 2.2. 세액공제·감면 적용을 위한 부속명세서 입력 항목
법인세 환급의 가장 큰 비중을 차지하는 세액공제를 적용하기 위해, `세액공제조정명세서(3)` 및 각 공제별 계산서에 다음과 같은 세부 데이터가 입력되어야 한다 [cite: 6, 7].

#### 2.2.1. 고용 관련 (고용증대세액공제, 통합고용세액공제)
고용 관련 세액공제는 '상시근로자 수'의 변동을 계산하는 것이 핵심이며, 이를 위해 인사/급여 데이터가 월별로 정밀하게 입력되어야 한다 [cite: 8, 9].

*   **상시근로자 현황 데이터 (사원별 상세):**
    *   **입사일/퇴사일:** 근로제공 기간 산정을 위한 필수 필드.
    *   **주민등록번호:** 연령(청년 여부) 및 내국인 판별용.
    *   **월별 근무 현황:** 1월~12월 각 월말 현재 재직 여부 (O/X 또는 1/0.5/0.75 등 단시간 근로자 가중치 포함).
    *   **우대 요건 플래그:**
        *   청년 여부 (만 15세~29세/34세, 군복무 기간 차감 계산).
        *   장애인 여부 (장애인증명서 기준).
        *   60세 이상 여부.
        *   경력단절여성 여부.
    *   **임금 데이터:** 연간 총급여액 (비과세 소득 제외, 최저임금 미만자 배제용).

*   **공제세액 계산서 입력 필드 (집계 데이터):**
    *   **직전 과세연도 상시근로자 수:** 전체 / 청년 등 / 청년 외 (전년도 신고서 값).
    *   **해당 과세연도 상시근로자 수:** 전체 / 청년 등 / 청년 외 (당해 연도 월별 평균값).
    *   **증가 인원 수:** (해당 연도 - 직전 연도) 자동 계산되나 한도 적용을 위해 수정 가능.
    *   **공제금액 단위:** 1인당 공제액 (수도권 내/외, 기업 규모에 따라 400만원~1,550만원 등 코드값 선택) [cite: 10].

#### 2.2.2. 투자 관련 (통합투자세액공제)
2021년 이후 통합투자세액공제를 적용받기 위해 자산별 세부 내역이 필요하다 [cite: 11, 12].

*   **투자자산 명세 (건별 입력):**
    *   **자산명:** 기계장치, 운반구, 공구기구 등.
    *   **취득일자:** 투자가 완료된 시점 (세액공제 귀속 시기 판단).
    *   **취득가액:** 실제 투자 금액 (정부보조금 차감 후 금액).
    *   **자산 구분 코드:**
        *   일반 사업용 자산 (기본공제 대상).
        *   신성장·원천기술 사업화 시설 (우대공제 대상).
        *   국가전략기술 사업화 시설 (최고 우대공제 대상).
    *   **수도권 과밀억제권역 여부:** 증설 투자인 경우 공제 배제 여부 판단 필드 [cite: 12].
    *   **감가상각 내용연수:** 사후관리 기간 확인용.

*   **투자 세액 계산 필드:**
    *   **해당 연도 투자금액 합계:** (A).
    *   **직전 3년간 연평균 투자금액:** (B) = (직전 3년 투자액 합계 / 3).
    *   **초과 투자금액:** (A) - (B) (음수일 경우 0).
    *   **공제율:** 기본공제율(%) + 추가공제율(3% or 10%).

#### 2.2.3. R&D 관련 (연구·인력개발비 세액공제)
R&D 비용을 일반연구와 신성장/원천기술 연구로 구분하여 입력한다 [cite: 13, 14].

*   **연구개발비 명세서 (비용 항목별):**
    *   **인건비:** 전담 연구요원의 성명, 주민번호, 연간 급여총액 (비과세 제외, 퇴직금 제외).
    *   **재료비:** 연구용 시약, 부품 등의 구입 비용 합계.
    *   **위탁·공동연구비:** 외부 기관(대학, 연구소)에 지출한 용역 비용.
    *   **기술명/과제명:** 신성장·원천기술 해당 시 관련 기술 코드 및 프로젝트명.

*   **증가분 방식 계산 필드 (선택 시):**
    *   **해당 연도 발생액:** 당기 총 R&D 비용.
    *   **직전 4년간 발생액 합계:** 연평균 발생액 산출용.
    *   **직전 연도 발생액:** 증가 여부 판단용.

#### 2.2.4. 이월세액공제 내역
경정청구 시 당기 공제액이 최저한세에 걸리거나 납부세액이 부족할 경우 이월된다.

*   **세액공제조정명세서(3) 또는 (6) 이월액 계산 탭:**
    *   **발생 연도:** (예: 2020, 2021, 2022...).
    *   **공제 항목명:** (예: 고용증대세액공제).
    *   **발생금액:** 당초 발생한 총 세액공제액.
    *   **기 공제액:** 과거에 이미 공제받은 금액.
    *   **당기 공제 대상액:** (발생금액 - 기 공제액).
    *   **이월액:** 당기에도 공제받지 못하고 다음 해로 넘길 금액 [cite: 15].

---

## 3. 종합소득세 환급액 계산을 위한 납세자 입력 항목

개인사업자, 프리랜서, 근로소득자가 종합소득세 경정청구를 할 때, `종합소득세 과세표준 확정신고서(별지 제40호 서식)`를 재작성해야 한다. 홈택스와 세무 소프트웨어는 이를 화면상의 탭(Tab) 형태로 구현하고 있다 [cite: 16, 17].

### 3.1. 종합소득세 확정신고서(별지 제40호 서식) 상의 입력 필드
*   **신고유형:** 자기조정, 외부조정, 성실신고, 간편장부, 기준경비율 등 (당초 신고 유형 유지).
*   **소득금액 집계:** 이자, 배당, 사업(부동산임대 포함), 근로, 연금, 기타 소득금액의 합계 [cite: 18].
*   **종합소득 공제 계:** 인적공제 + 연금보험료 + 특별소득공제 합계.
*   **과세표준:** (종합소득금액 - 소득공제).
*   **산출세액:** 과세표준 × 기본세율(6%~45%).
*   **세액감면/공제 계:** (수정 대상 핵심 필드).
*   **가산세:** 당초 신고 불성실 등으로 인한 가산세 (경정청구 시에는 보통 변동 없으나, 수정신고 시 중요).
*   **기납부세액:** 중간예납, 원천징수세액 합계.

### 3.2. 사업소득 외 합산소득 상세
종합소득세는 모든 소득을 합산하므로, 누락된 소득이 있거나 수정될 소득 명세를 입력한다 [cite: 19].

*   **근로소득:** 총급여액, 근로소득공제(자동계산), 근로소득금액.
*   **금융소득(이자/배당):**
    *   소득 구분 코드 (예: 비영업대금의 이익 16, 배당소득 21).
    *   금융회사명 / 사업자등록번호.
    *   총수입금액 (세전 이자/배당액).
    *   원천징수세액 (소득세/지방소득세).
    *   **Gross-up 금액:** 배당가산액 (배당소득 2천만원 초과 시 자동 계산 필드).
*   **기타소득:** 총수입금액, 필요경비(60% 의제 등), 소득금액.

### 3.3. 소득공제 항목별 금액
경정청구의 다수는 연말정산 시 누락된 소득공제를 반영하는 것이다 [cite: 20, 21].

*   **인적공제:**
    *   **기본공제 대상자:** 성명, 주민등록번호, 관계코드(본인/배우자/자녀 등), 내/외국인 구분.
    *   **추가공제 체크박스:** 경로우대(70세 이상), 장애인, 부녀자, 한부모.
*   **연금보험료 공제:** 국민연금, 공무원연금 등 본인 부담금 납입액.
*   **특별소득공제 (근로소득자):**
    *   **보험료:** 건강보험료(노인장기요양보험 포함), 고용보험료 납입액.
    *   **주택자금:** 주택임차차입금 원리금 상환액, 장기주택저당차입금 이자상환액 (상환 기간, 고정금리 여부 등 조건별 입력).

### 3.4. 세액공제 항목별 금액
*   **자녀세액공제:** 7세 이상 자녀 수 (자동 계산).
*   **연금계좌세액공제:**
    *   구분: 연금저축 / 퇴직연금.
    *   금융회사명, 계좌번호, 당해 연도 납입금액.
*   **특별세액공제:**
    *   **보장성 보험료:** 일반/장애인 전용 보험료 납입액.
    *   **의료비:** 본인/경로우대자/장애인 의료비, 난임시술비, 산후조리원비 등 구분하여 지급액 입력 (실손보험금 수령액 차감 필드 필수).
    *   **교육비:** 취학전 아동, 초중고, 대학생, 본인, 장애인 특수교육비 구분 입력.
    *   **기부금:** 법정/지정기부금 구분, 기부처 사업자번호, 기부 금액 [cite: 20].
*   **기장세액공제:** 간편장부 대상자가 복식부기로 기장한 경우 산출세액의 20% 입력.
*   **월세액 세액공제:** 임대인 성명/주민번호, 주택유형/면적, 임대차 계약기간, 연간 월세 지급액.

### 3.5. 기납부세액
*   **중간예납세액:** 11월에 고지되어 납부한 세액.
*   **원천징수세액:** 사업소득(3.3%) 원천징수영수증상의 소득세 합계.
*   **수시부과세액:** (해당 시) 수시 부과되어 납부한 세액.

---

## 4. 공통 입력 항목: 경정청구서 및 비교 분석

법인세와 종합소득세 모두 최종적으로는 `과세표준 및 세액의 결정(경정) 청구서`를 작성하여 제출해야 한다. 이 서식은 국세청 전산에 환급 신청을 등록하는 마스터 서식이다 [cite: 22, 23].

### 4.1. 경정청구서(수정신고서) 작성에 공통으로 필요한 필드
*   **청구인 기본 정보:** 상호(성명), 사업자등록번호(주민번호), 주소, 전화번호.
*   **신고 내역 정보:**
    *   **법정신고일:** 당초 정기신고 기한 (예: 2023년 5월 31일).
    *   **최초신고일:** 실제 신고서를 제출한 날짜.
    *   **경정청구 사유 코드:** (중요) 후발적 사유, 계산 착오, 법령 적용 착오, 공제 감면 누락 등 선택 [cite: 24].
    *   **청구 이유 상세:** 서술형 필드 (예: "2021년 귀속 고용증대세액공제 누락분에 대한 경정청구").
*   **국세환급금 계좌신고:**
    *   거래은행명 (코드).
    *   계좌번호 (반드시 본인/법인 명의).

### 4.2. 감면·공제 전환 시 비교 분석에 필요한 기존 신고 데이터
경정청구서의 핵심은 **[당초 신고]**와 **[경정 청구(수정)]**를 나란히 비교하는 표이다. 소프트웨어에서는 당초 신고 데이터를 자동으로 불러오지만, 수기 작성 시 다음 필드들을 직접 대조 입력해야 한다 [cite: 22, 25].

| 비교 항목 (데이터 필드) | 당초 신고 (A) | 경정 청구 (B) | 증감액 (B-A) |
| :--- | :--- | :--- | :--- |
| **과세표준 금액** | 당초 과세표준 | 수정 후 과세표준 | (변동 없을 수 있음) |
| **산출세액** | 당초 산출세액 | 수정 후 산출세액 | (-) |
| **공제·감면세액** | 당초 공제액 | **(증가)** 수정 후 공제액 | **(+) 환급 발생 원인** |
| **가산세액** | 당초 가산세 | 수정 가산세 | (-) |
| **납부할 세액 (결정세액)** | 당초 납부세액 | 수정 후 납부세액 | **(-) 환급신청액** |

**데이터 흐름 요약:**
1.  **ERP/회계프로그램 (Smart A, Wehago 등)**에서 기초 데이터(인건비, 투자액, 영수증 등)를 입력한다.
2.  **부속 명세서 (공제감면계산서 등)**를 작성하여 공제 가능 금액을 산출한다.
3.  **과세표준 및 세액조정계산서**에 산출된 공제액을 반영하여 '납부할 세액'을 낮춘다.
4.  **경정청구서** 서식에서 당초 신고분과 비교하여 '환급받을 세액'을 확정하고 마감한다.
5.  **홈택스 전자신고** 메뉴에서 마감된 파일을 변환하여 전송한다 [cite: 26].

---

## 5. 결론 및 시사점

한국의 세무 경정청구 프로세스는 **'부속 명세서 상세 입력' → '세액조정계산서 반영' → '경정청구서 비교표 작성'**의 3단계 데이터 흐름을 따른다.
*   **법인세 환급**은 `상시근로자 수`와 `투자 자산 상세` 데이터가 환급액을 결정하는 핵심 변수이다.
*   **종합소득세 환급**은 `소득공제 및 세액공제 항목(의료비, 기부금 등)`의 누락분을 정확히 입력하는 것이 관건이다.
*   모든 경정청구는 **당초 신고 데이터**와의 정합성 검증이 필수적이므로, 최초 신고서의 데이터를 정확히 이관(불러오기)하는 기능이 소프트웨어 활용의 핵심이다.

이 조사 결과는 세무 소프트웨어 개발, 경정청구 자동화 서비스 설계, 또는 세무 실무 매뉴얼 작성 시 필요한 데이터 스키마(Schema) 정의에 직접적으로 활용될 수 있다.

**Sources:**
1. [tpiinsight.co.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEN4qWtIrzP1TDj6IQLWurcFUhdkms0Bjh1ioQWbekxlhC4RnJOoEQvA59bpa8jvJRE2XfktGwT8eVVdpCokGDRDU9fSxNQPYOd51IvkCUB1qVVhVRS9k0b3pYLFS3GGfzTat2MATvj3juBun05bvVo8g6nT56cRsEMAyuIg8RtvUNvTWiRGm14gWjhqYUDZXUUJjsg1iT8sjzVHbFv0Q8ndPlFdMsIL-qeku8H1iF7VxmP15H1anX3s6tYMat-SV9v6bR8bI9OF-BEO-e_UKiT_f8s4q8PVC-5kVvjN862DgDAXCo2gJuRh9MOSe5MVLGsScr8vyB3Rnx8yNQ=)
2. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGOmrfteCQpA6UI_RGOfivr8vgdXksNYo7N2m-ptP2lFx_RyGp55bXuUWKBzi47GOzX3zY-8r8AQtZYx0jYbv9yfbtb6HfimSVDNc5PscsCCuJw00BA5bF9yAamRongbO9i)
3. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEIyWIR1EnN2f6tzTckVv-jbhpoo0O4bXQ772XwpzXLinZQBYMeQmwgdAkB6VQRQHLTPdj0fkR6wZVX0a8KkKrEQdIoFabg7uhOyCudDM6c0sNwNF2KZAPB05rgz8xsoFGaaKeCrck7S8zwOSbdTk5coNwJ3VtmwepCIsYVGbpXew==)
4. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGdEL8rAKuACCtFbGc1NJqnkDkIu-qOYawDANT3uquMhFtG0UBAl4U-6TuYA_6qr4Wedcq-poxcaF4YXU-Ly23bhZcRIgwwrXWgS-KjwPDrdp1wYtumBMQ5EYmIgWp9ABoY)
5. [bestonesol.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQH3aZhY6eTRV-8SxV5OdrUimWPSyT4hbXyVlm9evkUEdBCHucrFENU9-q1flQWL9w9SO8Ia071x7aXusKctp-PSqx7z3jP03sDZdRmPT_0esP0GBZ4ujHg-PC91cIwf_3Cv0JDuOMBHoL_vl1ICSW-v9_a8txSc1XwpHuu1R2W8j3p8b4uY6Yxusl0G2-vEfOjIMIb8PRV0f-VUd3E=)
6. [kacpta.or.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQH8iEiXywAakTdUIteiUTPEpj1s8RW-HdZCAfJQk3f_Up_hu5EbClTnM60N-QP3lUQoLuHyBfbizCveX1FncqoBASfbaVmUVO1ayr-xNg7PWD298Q4F0SNhTKfMtZ3FsyAyPIU=)
7. [bznav.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGmZNzMZZJbbXf-DdCiUyeSX0mSkiCCFXufGDUC7kTn4rV4ZEQ2FU8aTxgjySL_P7WpHKvrQ3cBUY4o4RsHJn75faNml2ZwzYu96qdQ3rhAZYaq1AlG9GH8AJaf)
8. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEbEaA31bbW__FxZvVWjesLHmwS5Ub6NqSnv6psQ9ukcoNgOZHUwO2ar26i8Frl6v_JipNBpwsjIATHao_cZSO258BSJwrXNnDf4TbU48F8U3Jiu0KXmZkDQmRKwaaYzDyIo148foem95cj2yABI9zr5EaQ_WWTceuLLAIL_RwTRYdCrfmCKNxP8imN5w==)
9. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHQlx_g8RTZX9UocK2gXK3L8afPbO0HHzi9LVC0zehJ-LrqFmLRSP66GeZly6Bba5xuHG9Jn9tKthOoUsL9knSl5RRHbZQ5Azj8wuA-2adsmwxP7GLsMo8Jgi7jt8NJQTi9)
10. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEgHqNLSNsLSzKkaSYiHgCTSjF0k4AwiD9Bg8U81hvBSUBoWQZq3AOCvlwodNv1Dn7A37pR1_YKzXS1snfMnACp37saAGeouoNcPKQqBsPITZj5mo1cU22m-4HwWVTz5O9N)
11. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGH0gRpepoaUXlBahwOY1A2D9OyB3AFZaXvOXRr9_xg-cvmzkqLof_bJMuMyCaO3q7lumTYpLD9E5wqIQ4ZzzQwdWMSke0yUpKC3L-mkrzB2iajOI6ncmt-syohfa8CGyJY)
12. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHUgXSmr2TJYBfzXVGvb3beDQCkcRgH9EIn67Il0YRuGNHQji0rJ7iEWNCcKS8TTYJqR8igWY_i0EqcdkKDgzWdgrzBVytCSpHWYaS7cIIxSnJ48Tm6SqwE8qsMuXalJUSDbjLCPiwdXfuj4q6tzR2vef4_zkBzzypRDYgmdzpGmwIOiDWwMq1ewkiXIg==)
13. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGuX1Tks7uwrUk669fWp0xMr0UCZGiAXGODLvX02DRrTX-ECefm3L_1Hzrq0-i1lf8GbeokiQYM23TKxOUI5Abi0Eaz7oRWJ0jPsRYP4ntG7H8_kdzSUAVBbA6_MbZ3iBI3YhMiWr-TRViaR7UsxbBBUnnNjRG6Y2v0C5HRWz4=)
14. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFo7gEfUfD56RI2Xj8Z_1ip58D3YlSYkuEQ8a8fIPyoTi1WNTT1CSVo4dFpydpgCy07qzO0HaGH6_eZ7EYwiQFj8MxWIVr_d_d9nJZTeuMIS02UWzag20imucWsW-cOXMGK)
15. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEkP2rA6bJPHYa4JpaTaeES3rK6pTrFBjcECm9cp-zjmk6OYaz7ECp3u__C6LTtRCDf9Muav_knJ8keDWCSbBFsorCWUGqnU0ddKHApVFehr-wK3tXeKsaEUbm94vydUWyo)
16. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFAbWSh5NIRNC7lD4cFB1w1e3AdYCWq63TQ2LpY6GJQMLje3BEdwU6iLMkjfcayv0Uhn9GfXZmKYEX7geha2OWuyjKuBXRVwBOgqDu41NFED_-KMvVaDvCCMTW4tiWBy1aM)
17. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEigbctyX6n71Zbz9wnCzQRrwy3vhN6K2YVKCYOvVMvJ2pBXDT5Fwj91JmByk3P-rlORPzk0ExBhoqv9u9C4D-fOID2ll05CwgiS7U4LSPG41X7ftmrtUSvo4tw06n7BEjZ)
18. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQETMUNxTgjI0bZ7_IhUi7TzExSdU5e1fEwetf3Sps47o1HPFhx-OrPCqzf_xT746RIFA3az2m7gtBl1ABsavUIPtgMbAcpHeTVEKWT6wITNCMc3Mmi32ZSeSMjWuurNRqiW1bpGdLyLOxLceNcucBj41J-VIIpZwQpD-PaVIX0Bh2dF0DJj7p-3MY5-0g==)
19. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEN23eqh2TTZYSUJrl423pD9B1vqaHOWprsypYGHNiK71977zj3WSr3j7zce4eKIKv6lEqWim2p_9ibv6JQGholHYZD6M3rw6SvyYoDreND7NgvrhxcXjPWHbXVCdwC08S9)
20. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGhOqzxv6fkil1LNg87Jc6U4HsxYXI_EBa8f9z3UwMhDPqYSpvxVbPV11U3o3OG9jQOpEgl8sHR2zEF4HUeLAGbrbCPLVrjDUbMj7OSbZ2zNv1a0zrrFO3SFZQLATw_55Bg)
21. [hometax.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQG7hkGBPZt0k9VODiO9baO5RKcDY2Q9SyEduqlDMBn-h9JsbjAO9Yeshf8ejJxJLKIbSXBWFpO2Vi-fF_LMwJp3xI8w9c_dDKDDt_MacMPQI0kDK_OfkeE5fteDZtpCx2BCsbU1GIkHLPNG4PJYpE6TfBmIPwI77f-hj-jrsy01dU4GfHUIuU8m7lfQmHWJgEozB7ERfPVWZ7zGVKpNXMMGa1wYekbNkRJE7APOzoa_Yw5oc3CCeL9TbAB5KZkKIwTKTQ==)
22. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGWxdm37z8S5rIbBByR4Gt04wKydeknebRlO4528SSuYh_REggj0PBw2G9CQZt5pdGFFdoUpaT6x4NSdO5DuPE6APf9Rf8DQxSu-ePYjZkuu3KEF6V-YWKpAjG21HTq8hB7gln1G_0hfMOz9BU=)
23. [law.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQH_Ixw7qlllJ3ljW6g4UFyXB0_lDdAP7xETs2O4LFig9glU-_Ctav2NF4g4QZlt62NNVdYvQN5E-SY-mIYs0AnK7mR8c1nBSa_MWAVK_HYKUQASOWpwltF1Dltv27lYGJkFuKCvWhfXJ6jIqLnaJwrV0MQUfAg2MRlYADZbG7VuxrzE1Lcak3HR-nyzIqUmF2EFCDyreaH_oRdT-B8J02vAsZ-goVP6Uk8ZXtzsZIe-hFgkyo6c2r5J9_qQmZpEPlckA4_wNEO7b2W4gytjwSWrfB9I8Z78Tty4Qeg-onGrXhR1kGGrYgfNIe34s01u1EKZ7CwDC-ZGznXw0BTalk6jAOvCy0RW7q4RLJoldw2IIDmg67tzvFyTGXDsMNetboNrEWJFCrQ8ihdiWc1vTqmfqCxEpB4ywIfnNABPsw5oMCKOBlIpdNmN54KgTg==)
24. [nts.go.kr](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEOtJTBDQsZj1BklSKL9f0jUSTsJu4q_rPmd0f1uL0EfDKAhIoZ3T82UJYBaPgcY-PQs8oJUXbdn8d9DkR9AalTgaBdiB6vEj6G84ZYCGcJiNCIRcohCGXI4T-nMHD5Slo1y763BQkEdECOI__6m0SqWd7A_vUFMklJA91Z71FkNA==)
25. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFyivL5frduk-RJ2prfxeG1rEhMa3ErZcjdOVMRUC4Ms5gBiGJZlQ39Vk-VU_LV2vs72VgJrOm-qA8y9pA2G3NhEurLGUf8vpBTqVMzqWLYO3E7Gh4pkGpv0Pcqy3sa5hgV)
26. [youtube.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFxIXxeAeDeRKKhoYuKCNjE69DidRa5H3uIRj68-uNrbG2p8msv4fGTW1xf34Mi_lmJofnhPFAM6m-O3psQMzlUjCM4xj5IFvzGwKRm102V0DsrDx-7F7Wfji61EKeVRj6w)

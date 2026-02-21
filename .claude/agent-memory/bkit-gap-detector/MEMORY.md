# Gap Detector Agent Memory

## Project: TaxServiceENTEC-ATR-01re

### Key Document Paths
- Input fields spec: `입력항목_통합정의서.md` (302 fields: 64 existing + 238 additional in 48 groups)
- Tax refund calculation: `통합_환급액계산.md` (very large, ~104K tokens, requires chunked reading)
- Gap analysis v1: `갭분석_입력항목vs점검항목.md` (79.0% match rate)
- Gap analysis v2: `갭분석_입력항목vs점검항목_v2.md` (100.0% match rate)
- Gap analysis v3: `갭분석_입력항목vs점검항목_v3.md` (93.9% strict / 97.6% relaxed)
- Gap analysis v4: `갭분석_입력항목vs점검항목_v4.md` (93.9% strict / 100.0% relaxed) -- FINAL

### Document Structure Notes
- `통합_환급액계산.md` has 4 stages of inspection items (점검 0 ~ 점검 37-3)
- Inspection items are split by tax type: 공통(common), 법인(corporate), 개인(individual)
- Stage 3-2 (line 3208~) = corporate-only, 3-2-2 (line 3321~) = individual-only
- Stage 3-3 (line 3492~) = shared items with dual numbering (법인/개인)

### Field ID Convention
- CMN_B### = Common existing, CMN_M### = Common missing/additional
- CRP_B### = Corporate existing, CRP_M### = Corporate missing
- INC_B### = Individual existing, INC_M### = Individual missing

### Gap Analysis History (2026-02-21)
- v1: 79.0% match rate (158/200), 42 gaps (HIGH 15 + MEDIUM 17 + LOW 10)
- v2: 100.0% match rate (202/202), 0 gaps -- all 42 gaps resolved with +50 fields
- v3: 93.9% strict / 97.6% relaxed (212 data points), 5 MISSING + 5 WEAK
  - Method: backward tracing of calculation logic (not just explicit mapping)
  - 212 data points = 202 (v2) + 10 newly discovered implicit requirements
  - HIGH gaps: GAP-01 (carry-forward loss structure), GAP-05 (original filing structure),
    GAP-07 (prior year tax/taxable income for loss carryback)
  - MEDIUM gaps: GAP-02 (expense rate type), GAP-04 (dispatch worker), GAP-09
    (employment credit transitional), GAP-10 (past business history structure)
  - LOW gaps: GAP-03 (metropolitan sub-zone), GAP-08 (NK defector), GAP-11 (multi-industry)
  - Recommended 23 new fields: 11 HIGH + 7 MEDIUM + 5 LOW
- v4: 93.9% strict / 100.0% relaxed (230 data points), 0 MISSING + 0 WEAK -- FINAL
  - All 10 v3 gaps (GAP-01~GAP-11) completely resolved by 23 added fields
  - 302 total fields = 64 existing + 238 additional
  - 14 NOTE items = system-determinable/reference data (properly excluded per design)
  - Additional coverage: deep-research fields (9 items OK), 2025 tax reform (7 items OK)
  - Status: COMPLETE -- ready for JSON schema/data model implementation

### Key Learnings
- When reading 통합_환급액계산.md, use chunked reads (500 lines per chunk)
- 기본 정보 입력 section (lines 107-210) contains explicit field requirements
- Boundary items (증빙/documents) should be excluded from input field mapping
- Multi-business-site (다사업장) fields use array of object structure
- v3 insight: text-type fields used in calculations need structured array of object
  (CRP_B013, CMN_M011c, INC_B017 are key examples)
- v3 insight: loss carryback (결손금 소급공제) requires prior year data not in current spec
- Backward tracing reveals gaps that explicit mapping misses (10 additional points found)
- v4 confirmed: text-type fields (CRP_B013, CMN_M011c, INC_B017) retained as reference-only;
  structured replacements (CRP_M013, CMN_M020, INC_M019) handle all calculations
- v4 confirmed: 302 fields fully cover all inspection items; next step is JSON schema

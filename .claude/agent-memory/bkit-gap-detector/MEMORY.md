# Gap Detector Agent Memory

## Project: TaxServiceENTEC-ATR-01re

### Key Document Paths
- Input fields spec: `입력항목_통합정의서.md` (279 fields: 64 existing + 215 additional in 48 groups)
- Tax refund calculation: `통합_환급액계산.md` (very large, ~104K tokens, requires chunked reading)
- Gap analysis v1: `갭분석_입력항목vs점검항목.md` (79.0% match rate)
- Gap analysis v2: `갭분석_입력항목vs점검항목_v2.md` (100.0% match rate)

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
- Key additions: CMN_M013a-g (process fields), CMN_M014a-d (social insurance),
  CMN_M015-M018 (disaster/retention/video/tenant), CRP_M009-M012 (corporate),
  INC_M015a-g (multi-site), INC_M016-M018 (individual), INC_M002h (installment)
- Structural issues resolved: multi-site (INC_M015), R&D types (CMN_M003j-p),
  corporate shared items (CRP_M010-M011, CMN_M018a)
- 11 recommended fields in Section 6.3 for precision improvement (not required)

### Key Learnings
- When reading 통합_환급액계산.md, use chunked reads (500 lines per chunk)
- 기본 정보 입력 section (lines 107-210) contains explicit field requirements
- Boundary items (증빙/documents) should be excluded from input field mapping
- Multi-business-site (다사업장) fields use array of object structure

# FAERS Data Structure — 2025 Q4 Analysis

> Analysed from sample files in `testing/` before designing the extraction script.

---

## ZIP File Layout

The quarterly ZIP (`faers_ascii_2025Q4.zip`) contains this structure, consistent across all quarters:

```
ASCII/
  DEMO25Q4.txt        (385,288 records, 25 cols)
  DRUG25Q4.txt      (1,815,349 records, 20 cols)
  REAC25Q4.txt      (1,349,105 records,  4 cols)
  OUTC25Q4.txt        (289,721 records,  3 cols)
  RPSR25Q4.txt         (10,694 records,  3 cols)
  THER25Q4.txt        (454,746 records,  7 cols)
  INDI25Q4.txt      (1,168,789 records,  4 cols)
  *.pdf               (data dictionaries for each file)
  ASC_NTS.pdf         (overall format notes)
Deleted/
  DELETE25Q4.txt        (4,497 primaryids of reports to be excluded)
FAQs.pdf
Readme.pdf
```

**Naming convention:** `{TABLE}{YY}Q{N}.txt` — e.g., `DEMO25Q4.txt` for 2025 Q4.

---

## ASCII Format

### Parsing properties

| Property | Value |
|----------|-------|
| Delimiter | `$` (dollar sign) |
| Quoting | None — fields are never quoted |
| Header | Single header row on line 1 only (no mid-file repeats) |
| Encoding | UTF-8 |
| Row integrity | Zero malformed rows (verified on DEMO 385K rows and DRUG 1.8M rows) |
| Free-text fields | `lit_ref` (DEMO), `dose_vbm`/`dose_form` (DRUG), `indi_pt` (INDI) — none contain embedded `$` |

### Column schemas

**DEMO** (25 cols) — patient demographics and report metadata:
```
primaryid, caseid, caseversion, i_f_code, event_dt, mfr_dt, init_fda_dt, fda_dt,
rept_cod, auth_num, mfr_num, mfr_sndr, lit_ref, age, age_cod, age_grp, sex,
e_sub, wt, wt_cod, rept_dt, to_mfr, occp_cod, reporter_country, occr_country
```

**DRUG** (20 cols) — drug information per report:
```
primaryid, caseid, drug_seq, role_cod, drugname, prod_ai, val_vbm, route,
dose_vbm, cum_dose_chr, cum_dose_unit, dechal, rechal, lot_num, exp_dt,
nda_num, dose_amt, dose_unit, dose_form, dose_freq
```

**REAC** (4 cols) — adverse reactions (MedDRA preferred terms):
```
primaryid, caseid, pt, drug_rec_act
```

**OUTC** (3 cols) — outcome codes (death, hospitalisation, disability, etc.):
```
primaryid, caseid, outc_cod
```

**RPSR** (3 cols) — report source:
```
primaryid, caseid, rpsr_cod
```

**THER** (7 cols) — drug therapy dates:
```
primaryid, caseid, dsg_drug_seq, start_dt, end_dt, dur, dur_cod
```

**INDI** (4 cols) — drug indication:
```
primaryid, caseid, indi_drug_seq, indi_pt
```

**DELETE file** — plain list of `primaryid` integers, one per line, **no header, no delimiter**.

---

## XML Format

The XML files follow the **ICH ICSR v2.1** standard (same format used for regulatory submissions). Three files are shipped per quarter, split purely by volume:

| File | Reports |
|------|---------|
| `1_ADR25Q4.xml` | 124,158 |
| `2_ADR25Q4.xml` | 123,766 |
| `3_ADR25Q4.xml` | 137,364 |
| **Total** | **385,288** — matches DEMO row count exactly |

Individual XML file sizes range from ~690 MB to ~839 MB uncompressed.

### Hierarchy

```xml
<ichicsr>
  <ichicsrmessageheader>...</ichicsrmessageheader>

  <safetyreport>                              <!-- one per adverse event report -->
    <safetyreportid>25866800</safetyreportid>
    <serious>1</serious>
    <seriousnessdeath>2</seriousnessdeath>
    <seriousnesslifethreatening>2</seriousnesslifethreatening>
    <seriousnesshospitalization>1</seriousnesshospitalization>
    <seriousnessdisabling>2</seriousnessdisabling>
    <seriousnesscongenitalanomali>2</seriousnesscongenitalanomali>
    <seriousnessother>1</seriousnessother>
    <receiptdate>20251001</receiptdate>
    <primarysource>
      <reportercountry>US</reportercountry>
      <qualification>5</qualification>      <!-- 1=physician, 5=consumer, etc. -->
    </primarysource>
    <sender>...</sender>
    <receiver>...</receiver>

    <patient>
      <patientonsetage>53</patientonsetage>
      <patientonsetageunit>801</patientonsetageunit>
      <patientsex>2</patientsex>

      <reaction>                            <!-- 1..N per report -->
        <reactionmeddraversionpt>28.1</reactionmeddraversionpt>
        <reactionmeddrapt>Gastrointestinal pain</reactionmeddrapt>
        <reactionoutcome>2</reactionoutcome>
      </reaction>

      <drug>                               <!-- 1..N per report -->
        <drugcharacterization>1</drugcharacterization>
        <medicinalproduct>BIMEKIZUMAB</medicinalproduct>
        <drugdosagetext>320 MILLIGRAM, EV 4 WEEKS...</drugdosagetext>
        <drugdosageform>Solution for injection in pre-filled pen</drugdosageform>
        <drugadministrationroute>058</drugadministrationroute>
        <drugindication>Psoriasis</drugindication>
        <drugstartdate>20250916</drugstartdate>
        <actiondrug>4</actiondrug>
        <activesubstance>
          <activesubstancename>BIMEKIZUMAB</activesubstancename>
        </activesubstance>
      </drug>

      <summary>
        <narrativeincludeclinical>CASE EVENT DATE: ...</narrativeincludeclinical>
      </summary>
    </patient>
  </safetyreport>

</ichicsr>
```

---

## Format Recommendation: ASCII

**Use ASCII** for the pipeline. Reasons:

1. **Trivial parsing** — `pd.read_csv(..., sep='$')` or BigQuery `LOAD DATA` with a single delimiter config. No tree traversal needed.
2. **Already normalised** — the 7 relational files map one-to-one to the planned BigQuery tables.
3. **No integrity issues** — zero malformed rows, no embedded delimiters, consistent column counts across all files.
4. **Predictable volume** — ~5.5M rows total per quarter, well within BigQuery capacity.
5. **XML overhead is unwarranted** — three files of ~700 MB each (uncompressed) per quarter for minimal additional field coverage.

The only field the XML adds that ASCII lacks is the individual seriousness sub-flags (`seriousnessdeath`, `seriousnesshospitalization`, etc.) as separate boolean columns. In ASCII these are derivable from the `outc_cod` values in OUTC (`DE`=death, `HO`=hospitalisation, etc.), so XML is unnecessary.

---

## Key Parsing Gotchas

| Issue | Detail |
|-------|--------|
| DELETE file has no header | Read as a plain list of integers, not CSV |
| DELETE primaryids must be filtered | Strip these from DEMO (and related tables by join) before loading to BigQuery to avoid warehousing retracted reports |
| `primaryid` vs `caseid` | `primaryid` is unique per report *version*; `caseid` links versions of the same case — use latest `caseversion` per `caseid` in the mart layer |
| Date fields are `YYYYMMDD` strings | Cast to DATE in dbt staging models; many fields are sparsely populated |
| Drug names are free-text | Normalise with `UPPER(TRIM(drugname))` in dbt staging |
| Older quarters used legacy format | Scope pipeline to 2019–present; pre-2012 data has a different schema |

---

## Importance of DELETE file

● Based on the data structure analysis, the DELETE file (DELETE25Q4.txt) contains a list of primaryid values for adverse event reports that the FDA has retracted or    
  invalidated for that quarter.                                  

  FAERS is an append-only reporting system — the FDA cannot go back and delete rows from previous quarterly files already distributed. Instead, each new quarterly      
  release ships a DELETE file listing primaryids that have since been determined to be:
                                                                                                                                                                        
  - Duplicate submissions (same case reported multiple times by different sources)                                                                                      
  - Erroneous reports (data quality issues identified post-submission)
  - Withdrawn reports (manufacturer or reporter retracted the submission)                                                                                               
                                                                                                                                                                        
  How to use it in the pipeline                                                                                                                                         
                                                                                                                                                                        
  When loading a quarter's data into BigQuery, you need to filter out any primaryid present in the DELETE file before or after loading raw data. The standard approach: 
   
  1. Load the DELETE list as its own table (faers_raw.deleted_reports)                                                                                                  
  2. In dbt staging models, add a WHERE primaryid NOT IN (SELECT primaryid FROM faers_raw.deleted_reports) filter
  3. For backfills, each quarter's DELETE file applies retroactively — a DELETE25Q4.txt entry may reference a primaryid from a 2021 report, so deletions need to be     
  applied across all historical data, not just the current quarter                                                                                                      
                                                                                                                                                                        
  Scale context                                                                                                                                                         
                                                            
  The 2025 Q4 DELETE file has 4,497 entries against ~385K reports in that quarter — roughly a 1.2% retraction rate. This is consistent with known FAERS data quality    
  characteristics.

*Analysed: 2026-04-01 — based on 2025 Q4 sample files*

# Healthcare Ecosystem Analysis: BSI Share & Transaction Flows

A SQL pipeline that measures how much of the Indonesian healthcare ecosystem is already served by **Bank Syariah Indonesia (BSI)**, and maps where money flows inside that ecosystem.

---

## What it does

| Part | Question it answers | Method |
|------|---------------------|--------|
| **Part 1** (Steps 1-9) | Which healthcare organisations are already BSI customers? | Fuzzy name matching between BSI's deposit master and official registers |
| **Part 2** (Steps 10-20) | How does money move between the parties of the ecosystem? | Classifying counterparties in transaction-flow data and filtering flows by party type |

### Organisation types covered

Clinics, community health centers (puskesmas), hospitals, insurance companies, pharmacies and pharmaceutical companies, medical-device (alkes) companies, and payroll customers (by facility and by healthcare-worker view).

### Money flows analysed (Part 2)

```
Public ──► BPJS Kesehatan                         (Flow A)
Public ──► Insurance companies                    (Flow B)
Public ──► Healthcare facilities                  (Flow C)
BPJS Kesehatan ──► Healthcare facilities          (Flow D)
Insurance companies ──► Healthcare facilities     (Flow E)
Healthcare facilities ──► Payroll                 (Flow F)
Healthcare facilities ──► Pharma wholesalers PBF  (Flow G)
Healthcare facilities ──► Medical-device firms    (Flow H)
```

---

## How the name matching works

Names from BSI's deposit master are compared with official registers (Ministry of Health, insurer lists, medical-device licences, etc.).

1. **Clean** both sides: strip punctuation, collapse whitespace, remove legal-form words (`PT`, `CV`, `TBK`, `LTD`) and hospital prefixes (`RS`, `RSUD`, `RSIA`, ...).
2. **Score** each pair with a normalised Levenshtein similarity:

   ```
   similarity = 1 - (levenshtein_distance / length_of_longer_name)
   ```

3. **Accept** a match when the score is above a threshold:
   - `> 0.9` for clinics, puskesmas, insurers, pharmacies, medical-device companies
   - `> 0.95` for hospitals (stricter)
4. **Exclude** matches rejected during manual review (`flag_exc = 1`).

---

## Steps

**Part 1: Which healthcare organisations are BSI customers?**

1. Shared lookup: cleaned names of BSI corporate customers
2. Clinics
3. Community health centers (puskesmas)
4. Hospitals
5. Insurance companies
6. Payroll customers, facility view
7. Payroll customers, employee view (healthcare workers)
8. Pharmacies and pharmaceutical companies
9. Medical-device (alkes) companies

**Part 2: Money flows inside the healthcare ecosystem**

10. Shared lookup: universe of healthcare-facility names
11. Shared lookup: combined TFD table (Corporate/Commercial + other segments)
12. Flow A: public → BPJS Kesehatan
13. Flow B: public → insurance companies
14. Flow C: public → healthcare facilities
15. Flow D: BPJS Kesehatan → healthcare facilities
16. Flow E: insurance companies → healthcare facilities
17. Flow F: healthcare facilities → payroll
18. Flow G: healthcare facilities → pharmaceutical wholesalers (PBF)
19. Flow H: healthcare facilities → medical-device (alkes) companies
20. Summary of all outgoing facility transactions (PBF + alkes)

---

## Data sources

| Schema | Content |
|--------|---------|
| `edl_datamart` | BSI deposit master (`master_dpk_bsi`), payroll (`master_payroll_phr_kolaborasi`), transactions (`dmx_transaction`) |
| `edl_external` | Ministry of Health registers: SatuSehat facilities, SIRS hospitals, domestic medical-device licences |
| `edl_sandbox` | Intermediate and result tables created by this script |

Result tables are written as Parquet (`CREATE TABLE ... STORED AS PARQUET AS SELECT ...`). Key outputs include:

- `hasil_similarity_klinik_puskemas`, `hasil_similarity_puskemas_only_update`, `hasil_similarity_rumahsakit`, `hasil_similarity_asuransi`, `hasil_similarity_farmasi_apotek`, `hasil_similarity_alkes`, `hasil_similarity_payroll_pov_faskes`, `hasil_similarity_payroll_pov_pegawai` (Part 1 matches)
- `da_wholesale_nama_universe_faskes_clean` (facility name universe)
- `whmart_union_compre_lite_aug_fix` (combined TFD table)
- `da_wholesale_trx_outgoing_faskes_pbf_only`, `da_wholesale_trx_outgoing_faskes_alkes_only`, `da_wholesale_trx_outgoing_faskes_all` (outgoing facility flows)

---

## Before you run it

- **Run the steps in order.** Later steps depend on tables created by earlier ones (e.g. Step 1 feeds Steps 2, 4, 8 and 9).
- **Update the hard-coded dates.** Snapshot and batch dates are written directly into the queries:

  | Data | Date(s) used |
  |------|--------------|
  | `master_dpk_bsi` snapshot | 2026-07-26 (clinics, hospitals, insurers, pharmacies, medical devices), 2026-08-30 (puskesmas), 2026-08-04 (payroll, facility view), 2026-08-31 (Part 2 transaction steps) |
  | Ministry of Health clinic scrape | 2026-06-08 |
  | Payroll snapshot | 2026-06-30 (Part 1), 2026-01-01 to 2026-08-31 (Part 2) |
  | Transactions (DMX / TFD) | 2026-01-01 to 2026-08-31 |

- **Tables are created with plain `CREATE TABLE`.** Drop existing tables first (the `-- drop table ...` comments are left above each statement) or the script will fail on re-run.
- **Known quirk:** the long name patterns in Steps 1, 2 and 4 contain line breaks. Pattern items that start right after a line break cannot match, so they have no effect. This is kept as-is so results do not change.

---

## Glossary

| Term | Meaning |
|------|---------|
| BSI | Bank Syariah Indonesia |
| DPK | Customer deposits (*Dana Pihak Ketiga*) |
| CIF / `nocif` | Customer identification number at the bank |
| Faskes | Healthcare facility (clinic, puskesmas, hospital) |
| Puskesmas | Government-run community health center |
| BPJS Kesehatan | National health insurance agency |
| PBF | Pharmaceutical wholesaler (*Pedagang Besar Farmasi*) |
| Alkes | Medical devices (*alat kesehatan*) |
| Nakes | Healthcare worker (*tenaga kesehatan*) |
| Masyarakat | The general public (retail customers, `individu = 'R'`) |
| Principal / Rekanan | The customer being analysed / its counterparty |
| TFD | Transaction Flow Diagram (Compre = Corporate/Commercial, Lite = other segments) |
| `flag_exc` | `1` = similarity match rejected after manual review |

Other Indonesian words in table and column names: *nasabah* (customer), *instansi* (institution), *nama* (name), *norek / norekening* (account number), *pengirim / penerima* (sender / receiver), *pegawai* (employee), *asuransi* (insurance), *apotek* (pharmacy), *farmasi* (pharmaceutical), *kemkes / kemenkes* (Ministry of Health), *rumahsakit* (hospital).

---

## Requirements

- Access to the BSI data lake schemas listed above (internal; not included in this repository).
- A SQL engine with `LEVENSHTEIN`, `REGEXP_REPLACE`, window functions and Parquet support (the syntax is Hive-style).

## Notes

This repository contains **query logic only**. It contains no customer data, and results should not be committed here.

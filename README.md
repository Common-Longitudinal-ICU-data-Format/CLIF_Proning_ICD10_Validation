# CLIF Proning ICD-10 Validation

CLIF Version: v2.1

## Objective
This project reuses the cohort from the parent study, [CLIF_Proning_Incidence_Severe_ARF](https://github.com/Common-Longitudinal-ICU-data-Format/CLIF_Proning_Incidence_Severe_ARF) (published as DOI: [10.1097/CCM.0000000000007148](https://doi.org/10.1097/CCM.0000000000007148)), which identifies patients with severe acute respiratory failure (ARF) who meet PROSEVA eligibility criteria for prone positioning.

Using that same cohort, this project runs a new, separate analysis: evaluating how well ICD-10-PCS procedural billing codes capture actual prone-positioning and invasive mechanical ventilation practice, using flowsheet-derived "ground truth" from CLIF data — comparing the flowsheet-based cohort against ICD-10-PCS proning (`5A09B5K`, `5A09C5K`, `5A09D5K`) and ventilation duration (`5A1935Z`, `5A1945Z`, `5A1955Z`) codes.

| Notebook | Role |
|---|---|
| [`00_local_CLIF_prone_incidence_cohort_identification_wCOVID.Rmd`](code/00_local_CLIF_prone_incidence_cohort_identification_wCOVID.Rmd) | Carried over unchanged from the parent study ([10.1097/CCM.0000000000007148](https://doi.org/10.1097/CCM.0000000000007148)). Identifies the local cohort of adults on invasive mechanical ventilation who meet PROSEVA eligibility criteria (severe P/F ratio confirmed on a second qualifying blood gas, or proned within 24 hours of first eligibility), applies exclusions (early tracheostomy, transfer-in on a vent, OR time before enrollment), and writes the analytic cohort and filtered CLIF tables used downstream. |
| [`01_local_prone_icd_validation_study.Rmd`](code/02_local_prone_icd_validation_study.Rmd) | New analysis for this project, built on top of the cohort from notebook 00. Pulls ICD-10-PCS procedure/diagnosis codes for the cohort, builds confusion matrices comparing ICD-documented proning/ventilation against flowsheet-documented proning/ventilation, computes sensitivity/specificity with Wilson 95% CIs, produces a Table 1 stratified by ICD-documentation status, and summarizes quarterly proning trends — with small-cell (n<10) suppression applied to every exported count for re-identification risk. |

## Required CLIF tables and fields
Refer to the [CLIF data dictionary](https://clif-icu.com/data-dictionary/data-dictionary-2.1.0) for details on required tables and fields.

1. **patient**: `patient_id`, `race_category`, `ethnicity_category`, `sex_category`
2. **hospitalization**: `patient_id`, `hospitalization_id`, `admission_dttm`, `discharge_dttm`, `age_at_admission`
3. **adt**: `patient_id`, `hospitalization_id`, `hospital_id`, `location_category`, `in_dttm`
4. **vitals**: `hospitalization_id`, `recorded_dttm`, `vital_category`, `vital_value` — categories: `height_cm`, `weight_kg`, `map`
5. **labs**: `hospitalization_id`, `lab_collect_dttm`, `lab_category`, `lab_value_numeric` — categories: `po2_arterial`, `creatinine`, `bilirubin_total`, `platelet_count`
6. **medication_admin_continuous**: `hospitalization_id`, `admin_dttm`, `med_name`, `med_category`, `med_group`, `med_dose`, `med_dose_unit` — vasoactive categories: `norepinephrine`, `epinephrine`, `phenylephrine`, `vasopressin`, `angiotensin`, `dopamine`, `dobutamine`, `milrinone`, `isoproterenol`
7. **respiratory_support**: `hospitalization_id`, `recorded_dttm`, `device_category`, `device_name`, `mode_category`, `tracheostomy`, `fio2_set`, `peep_set`, `tidal_volume_set`, `resp_rate_set`, `pressure_support_set`
8. **position**: `hospitalization_id`, `recorded_dttm`, `position_category`, `position_name`
9. **patient_assessments**: `hospitalization_id`, `recorded_dttm`, `assessment_category`, `numerical_value` — categories: `gcs_total`, `RASS`
10. **microbiology_nonculture**: `patient_id`, `hospitalization_id`, `organism_category`, `collect_dttm`, `result_category` — `sars_cov2` tests
11. **patient_procedures**: `hospitalization_id`, `procedure_code`, `procedure_code_format`, `procedure_billed_dttm`
12. **hospital_diagnosis**: `hospitalization_id`, `diagnosis_code`

## Cohort identification (notebook 00)
- Adults (age ≥ 18) admitted between 2018-01-01 and 2024-12-31, including linked/transferred encounters.
- First episode of invasive mechanical ventilation (IMV) lasting > 36 hours.
- PROSEVA-eligible: P/F ≤ 150 with PEEP ≥ 5 and FiO2 ≥ 0.6, confirmed on a second qualifying blood gas 12–24 hours later, **or** proned within 24 hours of the first qualifying blood gas.
- Excludes: tracheostomy within 24 hours of vent start, transfer-in already on a vent, OR time prior to study enrollment.
- One randomly selected eligible encounter per patient.
- Outputs cohort hospitalization IDs, filtered CLIF tables, and `prone_analytic_data.parquet` to `project_tables/`, plus consort/QC counts to `project_output/`.

## ICD-10 validation analysis (notebook 01)
- Restricts to patients enrolled in `cohort_year_cutoff` (2023) or later, when ICD-10-PCS procedural coding for proning is assumed more complete.
- Matches ICD-10-PCS codes from `clif_patient_procedures` (falling back to `clif_hospital_diagnosis` if needed) to the flowsheet-derived cohort.
- Computes confusion matrices and sensitivity/specificity (with Wilson 95% CIs) of ICD-10-PCS codes against flowsheet-documented proning and ventilation.
- Generates a Table 1 (continuous and categorical) stratified by ICD-documentation status.
- Summarizes proning counts by year-quarter.
- Applies a small-cell suppression threshold (n < 10 → masked) to every exported count, consistent with this being a multi-site pooled analysis.

## Running the project
1. Rename `config/config_template.yaml` to `config/config.yaml` and fill in your site's `clif_data_path`, `project_location`, `file_type`, `site`, and `site_time_zone`. See [`config/README.md`](config/README.md).
2. Run [`00_local_CLIF_prone_incidence_cohort_identification_wCOVID.Rmd`](code/00_local_CLIF_prone_incidence_cohort_identification_wCOVID.Rmd) to build the cohort and the filtered CLIF tables.
3. Run [`01_local_prone_icd_validation_study.Rmd`](code/02_local_prone_icd_validation_study.Rmd) to produce the ICD-10 validation results in `project_output/icd_revision_results_UPLOAD/`.
4. Upload `project_output/icd_revision_results_UPLOAD/` to box folder

## Source / acknowledgments
Cohort identification logic (notebook 00) is adapted from the original multi-center study repository: [CLIF_Proning_Incidence_Severe_ARF](https://github.com/Common-Longitudinal-ICU-data-Format/CLIF_Proning_Incidence_Severe_ARF).

Source owners:
- Chad Hochberg ([chochbe1](https://github.com/chochbe1))
- Anna Barker ([baanna23](https://github.com/baanna23))

## License
Apache License 2.0 — see [LICENSE](LICENSE).

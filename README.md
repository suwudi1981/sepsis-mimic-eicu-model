# sepsis-mimic-eicu-model
mimic eicu  sepsis code
-- ================================================================
-- MIMIC-IV DKA 6小时预测窗口 · 首次测量值 · 完整数据提取
-- ================================================================

WITH
-- 1. DKA 患者（首次ICU入院，按住院分组取最小stay_id）
-- 排除入院时/6小时内已有脓毒症诊断的患者
dka_stays AS (
    SELECT DISTINCT
        ie.stay_id,
        ie.hadm_id,
        ie.subject_id,
        ie.intime AS icu_intime,
        pt.anchor_age AS age,
        pt.gender,
        ad.race AS ethnicity,
        ad.admission_type,
        ad.insurance,
        ad.marital_status,
        ad.language
    FROM mimiciv_icu.icustays ie
    INNER JOIN mimiciv_hosp.admissions ad ON ie.hadm_id = ad.hadm_id
    INNER JOIN mimiciv_hosp.patients pt ON ie.subject_id = pt.subject_id
    INNER JOIN mimiciv_hosp.diagnoses_icd dx ON ie.hadm_id = dx.hadm_id
    WHERE ie.stay_id IN (
        SELECT MIN(stay_id)
        FROM mimiciv_icu.icustays
        GROUP BY hadm_id
    )
    AND pt.anchor_age >= 18
    AND (
        (dx.icd_version = 9 AND dx.icd_code IN ('25010','25011','25012','25013'))
        OR
        (dx.icd_version = 10 AND dx.icd_code IN ('E101','E1010','E1011','E111','E1110','E1111'))
    )
    AND dx.seq_num = 1
    -- ========== 排除入院时/6小时内已有脓毒症的患者 ==========
    AND NOT EXISTS (
        SELECT 1 FROM mimiciv_hosp.diagnoses_icd dx_sepsis
        WHERE dx_sepsis.hadm_id = ie.hadm_id
        AND (
            (dx_sepsis.icd_version = 9 AND (
                dx_sepsis.icd_code LIKE '038%'
                OR dx_sepsis.icd_code = '99591'
                OR dx_sepsis.icd_code = '99592'
                OR dx_sepsis.icd_code = '78552'
            ))
            OR
            (dx_sepsis.icd_version = 10 AND (
                dx_sepsis.icd_code LIKE 'A41%'
                OR dx_sepsis.icd_code = 'R652'
                OR dx_sepsis.icd_code = 'R651'
            ))
        )
        AND dx_sepsis.seq_num <= 2
    )
    -- ========== 排除结束 ==========
),

-- 2. 生命体征（6小时内首次值）
vitals_batch AS (
    SELECT
        c.stay_id,
        di.label,
        MIN(c.valuenum) AS first_value
    FROM mimiciv_icu.chartevents c
    INNER JOIN mimiciv_icu.d_items di ON c.itemid = di.itemid
    INNER JOIN dka_stays d ON c.stay_id = d.stay_id
    WHERE EXTRACT(EPOCH FROM (c.charttime - d.icu_intime)) / 60 BETWEEN 0 AND 360
      AND c.valuenum IS NOT NULL
      AND di.label IN (
          'Heart Rate',
          'Respiratory Rate',
          'Non Invasive Blood Pressure systolic',
          'Non Invasive Blood Pressure mean',
          'Temperature Fahrenheit',
          'O2 saturation pulseoxymetry'
      )
    GROUP BY c.stay_id, di.label
),

-- 3. 实验室指标（6小时内首次值）
labs_batch AS (
    SELECT
        l.subject_id,
        l.hadm_id,
        dli.label,
        MIN(l.valuenum) AS first_value
    FROM mimiciv_hosp.labevents l
    INNER JOIN mimiciv_hosp.d_labitems dli ON l.itemid = dli.itemid
    INNER JOIN dka_stays d ON l.subject_id = d.subject_id AND l.hadm_id = d.hadm_id
    WHERE EXTRACT(EPOCH FROM (l.charttime - d.icu_intime)) / 60 BETWEEN 0 AND 360
      AND l.valuenum IS NOT NULL
      AND dli.label IN (
          'White Blood Cells',
          'Hemoglobin',
          'Hematocrit',
          'Platelet Count',
          'RDW',
          'Creatinine',
          'Urea Nitrogen',
          'Lactate',
          'Anion Gap',
          'Potassium',
          'Bicarbonate',
          'Glucose',
          'pH'
      )
    GROUP BY l.subject_id, l.hadm_id, dli.label
),

-- 4. 合并症（基于ICD代码）
comorbidities AS (
    SELECT
        d.stay_id,
        d.hadm_id,
        MAX(CASE WHEN EXISTS (
            SELECT 1 FROM mimiciv_hosp.diagnoses_icd dx
            WHERE dx.hadm_id = d.hadm_id
                AND (
                    (dx.icd_version = 9 AND dx.icd_code LIKE '250.7%')
                    OR (dx.icd_version = 10 AND dx.icd_code LIKE 'E10.5%')
                    OR (dx.icd_version = 10 AND dx.icd_code LIKE 'E11.5%')
                )
        ) THEN 1 ELSE 0 END) AS dpvd,
        MAX(CASE WHEN EXISTS (
            SELECT 1 FROM mimiciv_hosp.diagnoses_icd dx
            WHERE dx.hadm_id = d.hadm_id
                AND (
                    (dx.icd_version = 10 AND dx.icd_code LIKE 'I10%')
                    OR (dx.icd_version = 9 AND dx.icd_code LIKE '401%')
                )
        ) THEN 1 ELSE 0 END) AS hypertension,
        MAX(CASE WHEN EXISTS (
            SELECT 1 FROM mimiciv_hosp.diagnoses_icd dx
            WHERE dx.hadm_id = d.hadm_id
                AND dx.icd_code LIKE 'A%'
                AND dx.icd_code NOT IN ('A00','A01','A02','A03','A04','A05','A06','A07','A08','A09')
        ) THEN 1 ELSE 0 END) AS infection_history,
        MAX(CASE WHEN EXISTS (
            SELECT 1 FROM mimiciv_hosp.diagnoses_icd dx
            WHERE dx.hadm_id = d.hadm_id
                AND (
                    (dx.icd_version = 9 AND dx.icd_code IN ('25011','25013'))
                    OR (dx.icd_version = 10 AND dx.icd_code IN ('E101','E1010','E1011'))
                )
        ) THEN 1 ELSE 0 END) AS t1dm,
        MAX(CASE WHEN EXISTS (
            SELECT 1 FROM mimiciv_hosp.diagnoses_icd dx
            WHERE dx.hadm_id = d.hadm_id
                AND (
                    (dx.icd_version = 9 AND dx.icd_code IN ('25010','25012'))
                    OR (dx.icd_version = 10 AND dx.icd_code IN ('E111','E1110','E1111'))
                )
        ) THEN 1 ELSE 0 END) AS t2dm
    FROM dka_stays d
    GROUP BY d.stay_id, d.hadm_id
),

-- 5. GCS（最小GCS）
gcs_min AS (
    SELECT stay_id, MIN(gcs) AS gcs_min
    FROM mimiciv_derived.gcs
    WHERE stay_id IN (SELECT stay_id FROM dka_stays)
    GROUP BY stay_id
),

-- 6. APS III 评分
apsiii_score AS (
    SELECT stay_id, apsiii AS apsiii_score
    FROM mimiciv_derived.apsiii
    WHERE stay_id IN (SELECT stay_id FROM dka_stays)
),

-- 7. Charlson 指数
charlson_score AS (
    SELECT hadm_id, charlson_comorbidity_index AS charlson_index
    FROM mimiciv_derived.charlson
    WHERE hadm_id IN (SELECT hadm_id FROM dka_stays)
),

-- 8. SOFA 24小时最差值（用于结局定义）
sofa_24h AS (
    SELECT stay_id, MAX(sofa_24hours) AS sofa_24h_max
    FROM mimiciv_derived.sofa
    WHERE stay_id IN (SELECT stay_id FROM dka_stays)
    GROUP BY stay_id
),

-- 9. 脓毒症结局（使用ICD代码定义，与论文一致）
sepsis_label AS (
    SELECT DISTINCT 
        d.stay_id,
        CASE WHEN EXISTS (
            SELECT 1 FROM mimiciv_hosp.diagnoses_icd dx
            WHERE dx.hadm_id = d.hadm_id
            AND (
                (dx.icd_version = 9 AND (
                    dx.icd_code LIKE '038%'
                    OR dx.icd_code = '99591'
                    OR dx.icd_code = '99592'
                    OR dx.icd_code = '78552'
                ))
                OR
                (dx.icd_version = 10 AND (
                    dx.icd_code LIKE 'A41%'
                    OR dx.icd_code = 'R652'
                    OR dx.icd_code = 'R651'
                ))
            )
        ) THEN 1 ELSE 0 END AS sepsis_flag
    FROM dka_stays d
),

-- 10. 血管加压药使用（6小时内）
vasopressor_use AS (
    SELECT ie.stay_id, 1 AS vasopressor
    FROM mimiciv_icu.inputevents ie
    INNER JOIN mimiciv_icu.d_items di ON ie.itemid = di.itemid
    WHERE ie.stay_id IN (SELECT stay_id FROM dka_stays)
        AND EXTRACT(EPOCH FROM (ie.starttime - (SELECT icu_intime FROM dka_stays d WHERE d.stay_id = ie.stay_id))) / 60 BETWEEN 0 AND 360
        AND (
            LOWER(di.label) LIKE '%norepineph%'
            OR LOWER(di.label) LIKE '%epineph%'
            OR LOWER(di.label) LIKE '%dopamin%'
            OR LOWER(di.label) LIKE '%dobutamin%'
            OR LOWER(di.label) LIKE '%vasopress%'
            OR LOWER(di.label) LIKE '%phenyleph%'
        )
    GROUP BY ie.stay_id
),

-- 11. 机械通气（6小时内）
mech_vent AS (
    SELECT DISTINCT stay_id, 1 AS mech_vent
    FROM mimiciv_derived.ventilation
    WHERE stay_id IN (SELECT stay_id FROM dka_stays)
        AND EXTRACT(EPOCH FROM (starttime - (SELECT icu_intime FROM dka_stays d WHERE d.stay_id = ventilation.stay_id))) / 60 BETWEEN 0 AND 360
),

-- 12. 抗生素（6小时内）
antibiotics AS (
    SELECT DISTINCT stay_id, 1 AS antibiotic_6h
    FROM mimiciv_icu.inputevents ie
    INNER JOIN mimiciv_icu.d_items di ON ie.itemid = di.itemid
    WHERE ie.stay_id IN (SELECT stay_id FROM dka_stays)
        AND EXTRACT(EPOCH FROM (ie.starttime - (SELECT icu_intime FROM dka_stays d WHERE d.stay_id = ie.stay_id))) / 60 BETWEEN 0 AND 360
        AND LOWER(di.label) ~ '(cef|piperacillin|vancomycin|meropenem|ertapenem|aztreonam|clindamycin|metronidazole|levofloxacin|ciprofloxacin|gentamicin|tobramycin|linezolid|ampicillin|amoxicillin|sulbactam|tazobactam|clavulanic|ceftriaxone|cefepime|ceftazidime|cefuroxime|azithromycin|clarithromycin|erythromycin|doxycycline|tigecycline|polymyxin|colistin|sulfamethoxazole|trimethoprim)'
),

-- 13. 最终数据组装
final_data AS (
    SELECT
        d.stay_id,
        d.hadm_id,
        d.subject_id,
        d.icu_intime,
        d.age,
        d.gender,
        d.ethnicity,
        d.admission_type,
        d.insurance,
        d.marital_status,
        d.language,
        co.t1dm,
        co.t2dm,
        co.dpvd,
        co.hypertension,
        co.infection_history,
        -- 生命体征
        (SELECT first_value FROM vitals_batch vb WHERE vb.stay_id = d.stay_id AND vb.label = 'Heart Rate') AS heart_rate,
        (SELECT first_value FROM vitals_batch vb WHERE vb.stay_id = d.stay_id AND vb.label = 'Respiratory Rate') AS resp_rate,
        (SELECT first_value FROM vitals_batch vb WHERE vb.stay_id = d.stay_id AND vb.label = 'Non Invasive Blood Pressure systolic') AS sbp,
        (SELECT first_value FROM vitals_batch vb WHERE vb.stay_id = d.stay_id AND vb.label = 'Non Invasive Blood Pressure mean') AS map,
        (SELECT first_value FROM vitals_batch vb WHERE vb.stay_id = d.stay_id AND vb.label = 'Temperature Fahrenheit') AS temperature,
        (SELECT first_value FROM vitals_batch vb WHERE vb.stay_id = d.stay_id AND vb.label = 'O2 saturation pulseoxymetry') AS spo2,
        -- 实验室
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'White Blood Cells') AS wbc,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'Hemoglobin') AS hemoglobin,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'Hematocrit') AS hematocrit,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'Platelet Count') AS platelet,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'RDW') AS rdw,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'Creatinine') AS scr,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'Urea Nitrogen') AS bun,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'Lactate') AS lactate,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'Anion Gap') AS anion_gap,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'Potassium') AS potassium,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'Bicarbonate') AS bicarbonate,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'Glucose') AS glucose,
        (SELECT first_value FROM labs_batch lb WHERE lb.subject_id = d.subject_id AND lb.hadm_id = d.hadm_id AND lb.label = 'pH') AS ph,
        -- 评分
        c.charlson_index,
        s.sofa_24h_max,
        a.apsiii_score,
        g.gcs_min,
        CASE WHEN g.gcs_min < 13 THEN 1 ELSE 0 END AS consciousness_disturbance,
        COALESCE(v.vasopressor, 0) AS vasopressor,
        COALESCE(mv.mech_vent, 0) AS mech_vent,
        COALESCE(abx.antibiotic_6h, 0) AS antibiotic_6h,
        sl.sepsis_flag AS sepsis_icd
    FROM dka_stays d
    LEFT JOIN comorbidities co ON d.stay_id = co.stay_id
    LEFT JOIN charlson_score c ON d.hadm_id = c.hadm_id
    LEFT JOIN sofa_24h s ON d.stay_id = s.stay_id
    LEFT JOIN apsiii_score a ON d.stay_id = a.stay_id
    LEFT JOIN gcs_min g ON d.stay_id = g.stay_id
    LEFT JOIN vasopressor_use v ON d.stay_id = v.stay_id
    LEFT JOIN mech_vent mv ON d.stay_id = mv.stay_id
    LEFT JOIN antibiotics abx ON d.stay_id = abx.stay_id
    LEFT JOIN sepsis_label sl ON d.stay_id = sl.stay_id
)

-- 14. 排除缺失核心变量超过2个的患者（最终输出）
SELECT *
FROM final_data
WHERE (
    (CASE WHEN spo2 IS NULL THEN 1 ELSE 0 END +
     CASE WHEN sbp IS NULL THEN 1 ELSE 0 END +
     CASE WHEN scr IS NULL THEN 1 ELSE 0 END +
     CASE WHEN wbc IS NULL THEN 1 ELSE 0 END +
     CASE WHEN hemoglobin IS NULL THEN 1 ELSE 0 END +
     CASE WHEN bun IS NULL THEN 1 ELSE 0 END +
     CASE WHEN anion_gap IS NULL THEN 1 ELSE 0 END +
     CASE WHEN potassium IS NULL THEN 1 ELSE 0 END +
     CASE WHEN bicarbonate IS NULL THEN 1 ELSE 0 END +
     CASE WHEN glucose IS NULL THEN 1 ELSE 0 END +
     CASE WHEN ph IS NULL THEN 1 ELSE 0 END +
     CASE WHEN t1dm IS NULL AND t2dm IS NULL THEN 1 ELSE 0 END +
     CASE WHEN dpvd IS NULL THEN 1 ELSE 0 END +
     CASE WHEN ethnicity IS NULL THEN 1 ELSE 0 END
    ) <= 2
)
ORDER BY hadm_id, icu_intime;

EICU脓毒血症提取代码
-- ============================================================ 
-- eICU-CRD DKA 6小时首次值提取（完整版，与MIMIC对齐）
-- 修正：排除入院时/6小时内已有脓毒症的患者
-- 血压多源补充、温度转摄氏度、合并症输出、gcs列名统一
-- ============================================================ 
SET search_path TO eicu, public;

-- 0. 创建DKA队列临时表（排除入院时/6小时内已有脓毒症的患者）
DROP TABLE IF EXISTS tmp_dka;
CREATE TEMP TABLE tmp_dka AS
SELECT
    p.patientunitstayid,
    p.uniquepid,
    CASE 
        WHEN p.age = '> 89' THEN 90
        WHEN p.age ~ '^\d+$' THEN CAST(p.age AS INTEGER)
        ELSE NULL
    END AS age,
    p.gender,
    p.ethnicity,
    p.unitadmitsource
FROM patient p
WHERE p.unitvisitnumber = 1
  AND EXISTS (
      SELECT 1 FROM diagnosis d
      WHERE d.patientunitstayid = p.patientunitstayid
        AND d.diagnosispriority IN ('Primary', 'Major')
        AND (
            d.icd9code LIKE '250.1%' OR d.icd9code LIKE '250.3%'
            OR LOWER(d.diagnosisstring) LIKE '%dka%'
            OR LOWER(d.diagnosisstring) LIKE '%ketoacidosis%'
        )
  )
  AND CASE 
        WHEN p.age = '> 89' THEN 90
        WHEN p.age ~ '^\d+$' THEN CAST(p.age AS INTEGER)
        ELSE NULL
      END >= 18
  -- ========== 排除入院时/6小时内已有脓毒症的患者 ==========
  AND NOT EXISTS (
      SELECT 1 FROM diagnosis d_sepsis
      WHERE d_sepsis.patientunitstayid = p.patientunitstayid
        AND d_sepsis.diagnosispriority IN ('Primary', 'Major')
        AND (
            d_sepsis.icd9code LIKE '038%'
            OR d_sepsis.icd9code = '99591'
            OR d_sepsis.icd9code = '99592'
            OR d_sepsis.icd9code = '78552'
            OR d_sepsis.icd9code = '7907'
            OR LOWER(d_sepsis.diagnosisstring) LIKE '%sepsis%'
            OR LOWER(d_sepsis.diagnosisstring) LIKE '%septic shock%'
            OR LOWER(d_sepsis.diagnosisstring) LIKE '%bacteremia%'
        )
        AND COALESCE(d_sepsis.diagnosisoffset, 0) <= 360
  )
  -- ========== 排除结束 ==========
;
CREATE INDEX ON tmp_dka(patientunitstayid);

-- 1. 生命体征（0-6h首次值，多源融合）
-- 1.1 心率、呼吸、体温（转摄氏度）、SpO2
DROP TABLE IF EXISTS tmp_vital_raw;
CREATE TEMP TABLE tmp_vital_raw AS
SELECT DISTINCT ON (d.patientunitstayid, label)
    d.patientunitstayid,
    label,
    val,
    observationoffset
FROM (
    SELECT patientunitstayid, 'heart_rate' AS label, heartrate AS val, observationoffset
    FROM vitalperiodic WHERE observationoffset BETWEEN 0 AND 360 AND heartrate IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'respiratory_rate', respiration, observationoffset
    FROM vitalperiodic WHERE observationoffset BETWEEN 0 AND 360 AND respiration IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'spo2', sao2, observationoffset
    FROM vitalperiodic WHERE observationoffset BETWEEN 0 AND 360 AND sao2 IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'temperature',
           CASE WHEN temperature > 45 THEN (temperature - 32) * 5.0/9.0 ELSE temperature END AS val,
           observationoffset
    FROM vitalperiodic WHERE observationoffset BETWEEN 0 AND 360 AND temperature IS NOT NULL
) t
JOIN tmp_dka d ON t.patientunitstayid = d.patientunitstayid
ORDER BY d.patientunitstayid, label, observationoffset;

-- 1.2 血压多源提取（第一次值）
-- 源1：vitalperiodic
DROP TABLE IF EXISTS tmp_bp_vital;
CREATE TEMP TABLE tmp_bp_vital AS
SELECT DISTINCT ON (patientunitstayid)
    patientunitstayid,
    systemicsystolic AS sbp,
    systemicdiastolic AS dbp,
    systemicmean AS map,
    observationoffset
FROM vitalperiodic
WHERE observationoffset BETWEEN 0 AND 360
  AND (systemicsystolic IS NOT NULL OR systemicmean IS NOT NULL)
ORDER BY patientunitstayid, observationoffset;

-- 源2：nursecharting 无创血压
DROP TABLE IF EXISTS tmp_bp_nurse;
CREATE TEMP TABLE tmp_bp_nurse AS
SELECT DISTINCT ON (patientunitstayid)
    patientunitstayid,
    MAX(CASE WHEN nursingchartcelltypevalname IN ('Systolic Blood Pressure','NIBP Systolic','BP Systolic')
             THEN CAST(nursingchartvalue AS NUMERIC) END) AS sbp_nurse,
    MAX(CASE WHEN nursingchartcelltypevalname IN ('Diastolic Blood Pressure','NIBP Diastolic','BP Diastolic')
             THEN CAST(nursingchartvalue AS NUMERIC) END) AS dbp_nurse,
    MAX(CASE WHEN nursingchartcelltypevalname IN ('Mean Arterial Pressure','NIBP Mean','BP Mean')
             THEN CAST(nursingchartvalue AS NUMERIC) END) AS map_nurse,
    MIN(nursingchartoffset) AS offset
FROM nursecharting
WHERE nursingchartoffset BETWEEN 0 AND 360
  AND nursingchartvalue ~ '^[0-9]+\.?[0-9]*$'
  AND nursingchartcelltypevalname IN (
      'Systolic Blood Pressure','NIBP Systolic','BP Systolic',
      'Diastolic Blood Pressure','NIBP Diastolic','BP Diastolic',
      'Mean Arterial Pressure','NIBP Mean','BP Mean'
  )
GROUP BY patientunitstayid
ORDER BY patientunitstayid, MIN(nursingchartoffset);

-- 源3：apacheapsvar 兜底
DROP TABLE IF EXISTS tmp_bp_apache;
CREATE TEMP TABLE tmp_bp_apache AS
SELECT DISTINCT ON (d.patientunitstayid)
    d.patientunitstayid,
    a.meanbp AS map_apache,
    NULL::numeric AS sbp_apache,
    NULL::numeric AS dbp_apache
FROM tmp_dka d
JOIN apacheapsvar a ON d.patientunitstayid = a.patientunitstayid
WHERE a.meanbp IS NOT NULL
ORDER BY d.patientunitstayid, a.apacheapsvarid;

-- 源4：vitalaperiodic 非周期性血压
DROP TABLE IF EXISTS tmp_bp_aperiodic;
CREATE TEMP TABLE tmp_bp_aperiodic AS
SELECT DISTINCT ON (patientunitstayid)
    patientunitstayid,
    noninvasivesystolic AS sbp_a,
    noninvasivediastolic AS dbp_a,
    noninvasivemean AS map_a,
    observationoffset
FROM vitalaperiodic
WHERE observationoffset BETWEEN 0 AND 360
  AND (noninvasivesystolic IS NOT NULL OR noninvasivemean IS NOT NULL)
ORDER BY patientunitstayid, observationoffset;

-- 1.3 合并血压，优先级：vitalperiodic > nursecharting > apacheapsvar > vitalaperiodic
DROP TABLE IF EXISTS tmp_bp_final;
CREATE TEMP TABLE tmp_bp_final AS
SELECT 
    d.patientunitstayid,
    COALESCE(bv.sbp, bn.sbp_nurse, ba.sbp_apache, bap.sbp_a) AS sbp,
    COALESCE(bv.dbp, bn.dbp_nurse, ba.dbp_apache, bap.dbp_a) AS dbp,
    COALESCE(bv.map, bn.map_nurse, ba.map_apache, bap.map_a) AS map
FROM tmp_dka d
LEFT JOIN tmp_bp_vital bv ON d.patientunitstayid = bv.patientunitstayid
LEFT JOIN tmp_bp_nurse bn ON d.patientunitstayid = bn.patientunitstayid
LEFT JOIN tmp_bp_apache ba ON d.patientunitstayid = ba.patientunitstayid
LEFT JOIN tmp_bp_aperiodic bap ON d.patientunitstayid = bap.patientunitstayid;

-- 1.4 生命体征宽表
DROP TABLE IF EXISTS tmp_vital_wide;
CREATE TEMP TABLE tmp_vital_wide AS
SELECT 
    d.patientunitstayid,
    MAX(CASE WHEN v.label = 'heart_rate' THEN v.val END) AS heart_rate,
    MAX(CASE WHEN v.label = 'respiratory_rate' THEN v.val END) AS respiratory_rate,
    MAX(CASE WHEN v.label = 'spo2' THEN v.val END) AS spo2,
    MAX(CASE WHEN v.label = 'temperature' THEN v.val END) AS temperature,
    bp.sbp,
    bp.dbp,
    bp.map
FROM tmp_dka d
LEFT JOIN tmp_vital_raw v ON d.patientunitstayid = v.patientunitstayid
LEFT JOIN tmp_bp_final bp ON d.patientunitstayid = bp.patientunitstayid
GROUP BY d.patientunitstayid, bp.sbp, bp.dbp, bp.map;

-- 2. 实验室（0-6h首次值）
DROP TABLE IF EXISTS tmp_lab;
CREATE TEMP TABLE tmp_lab AS
SELECT DISTINCT ON (d.patientunitstayid, label)
    d.patientunitstayid,
    label,
    labresult,
    labresultoffset
FROM (
    SELECT patientunitstayid, 'wbc' AS label, labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'WBC x 1000' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'hemoglobin', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'Hgb' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'hematocrit', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'Hct' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'platelet', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'platelets x 1000' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'rdw', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'RDW' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'creatinine', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'creatinine' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'bun', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'BUN' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'lactate', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'lactate' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'anion_gap', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'anion gap' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'potassium', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'potassium' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'bicarbonate', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND (labname = 'bicarbonate' OR labname = 'HCO3') AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'glucose', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'glucose' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'ph', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = 'pH' AND labresult IS NOT NULL
    UNION ALL
    SELECT patientunitstayid, 'neutrophil_pct', labresult, labresultoffset
    FROM lab WHERE labresultoffset BETWEEN 0 AND 360 AND labname = '-polys' AND labresult IS NOT NULL
) t
JOIN tmp_dka d ON t.patientunitstayid = d.patientunitstayid
ORDER BY d.patientunitstayid, label, labresultoffset;

DROP TABLE IF EXISTS tmp_lab_wide;
CREATE TEMP TABLE tmp_lab_wide AS
SELECT patientunitstayid,
    MAX(CASE WHEN label='wbc' THEN labresult END) AS wbc,
    MAX(CASE WHEN label='hemoglobin' THEN labresult END) AS hemoglobin,
    MAX(CASE WHEN label='hematocrit' THEN labresult END) AS hematocrit,
    MAX(CASE WHEN label='platelet' THEN labresult END) AS platelet,
    MAX(CASE WHEN label='rdw' THEN labresult END) AS rdw,
    MAX(CASE WHEN label='creatinine' THEN labresult END) AS creatinine,
    MAX(CASE WHEN label='bun' THEN labresult END) AS bun,
    MAX(CASE WHEN label='lactate' THEN labresult END) AS lactate,
    MAX(CASE WHEN label='anion_gap' THEN labresult END) AS anion_gap,
    MAX(CASE WHEN label='potassium' THEN labresult END) AS potassium,
    MAX(CASE WHEN label='bicarbonate' THEN labresult END) AS bicarbonate,
    MAX(CASE WHEN label='glucose' THEN labresult END) AS glucose,
    MAX(CASE WHEN label='ph' THEN labresult END) AS ph,
    MAX(CASE WHEN label='neutrophil_pct' THEN labresult END) AS neutrophil_pct
FROM tmp_lab
GROUP BY patientunitstayid;

-- 3. GCS 和意识障碍
DROP TABLE IF EXISTS tmp_gcs;
CREATE TEMP TABLE tmp_gcs AS
SELECT DISTINCT ON (d.patientunitstayid)
    d.patientunitstayid,
    (COALESCE(a.eyes,0)+COALESCE(a.motor,0)+COALESCE(a.verbal,0)) AS gcs,
    CASE WHEN (COALESCE(a.eyes,0)+COALESCE(a.motor,0)+COALESCE(a.verbal,0)) < 13 THEN 1 ELSE 0 END AS consciousness_disturbance
FROM tmp_dka d
JOIN apacheapsvar a ON d.patientunitstayid = a.patientunitstayid
WHERE a.eyes IS NOT NULL AND a.motor IS NOT NULL AND a.verbal IS NOT NULL
ORDER BY d.patientunitstayid, a.apacheapsvarid;

-- 4. 合并症
DROP TABLE IF EXISTS tmp_comorb;
CREATE TEMP TABLE tmp_comorb AS
SELECT d.patientunitstayid,
    MAX(CASE WHEN dx.icd9code LIKE '250.7%' THEN 1 ELSE 0 END) AS dpvd,
    MAX(CASE WHEN dx.icd9code LIKE '401%' THEN 1 ELSE 0 END) AS hypertension,
    MAX(CASE WHEN dx.icd9code IN ('0380','0381','0382','0383','0384','0388','0389',
                    '03810','03811','03812','03819','03840','03841',
                    '03842','03843','03844','03849','7907','99591','99592')
               OR dx.icd9code ~ '^003\.?1' THEN 1 ELSE 0 END) AS infection_history,
    MAX(CASE WHEN dx.icd9code IN ('25011','25013') THEN 1 ELSE 0 END) AS t1dm,
    MAX(CASE WHEN dx.icd9code IN ('25010','25012') THEN 1 ELSE 0 END) AS t2dm
FROM tmp_dka d
LEFT JOIN diagnosis dx ON d.patientunitstayid = dx.patientunitstayid
GROUP BY d.patientunitstayid;

-- 5. 器官支持与抗生素
DROP TABLE IF EXISTS tmp_vaso;
CREATE TEMP TABLE tmp_vaso AS
SELECT DISTINCT d.patientunitstayid, 1 AS vasopressor
FROM tmp_dka d
JOIN infusiondrug i ON d.patientunitstayid = i.patientunitstayid
WHERE i.infusionoffset BETWEEN 0 AND 360
  AND LOWER(i.drugname) ~ '(norepineph|epineph|dopamin|dobutamin|vasopressin|phenyleph)';

DROP TABLE IF EXISTS tmp_vent;
CREATE TEMP TABLE tmp_vent AS
SELECT DISTINCT d.patientunitstayid, 1 AS mech_vent
FROM tmp_dka d
JOIN ventilation_events v ON d.patientunitstayid = v.patientunitstayid
WHERE v.event = 'vent' AND v.hrs <= 6;

DROP TABLE IF EXISTS tmp_abx;
CREATE TEMP TABLE tmp_abx AS
SELECT DISTINCT d.patientunitstayid, 1 AS antibiotic_6h
FROM tmp_dka d
JOIN medication m ON d.patientunitstayid = m.patientunitstayid
WHERE (m.drugstartoffset BETWEEN 0 AND 360 OR m.drugorderoffset BETWEEN 0 AND 360)
  AND LOWER(m.drugname) ~ '(cef|piperacillin|vancomycin|meropenem|ertapenem|aztreonam|clindamycin|metronidazole|levofloxacin|ciprofloxacin|gentamicin|tobramycin|linezolid|ampicillin|amoxicillin|sulbactam|tazobactam|clavulanic|ceftriaxone|cefepime|ceftazidime|cefuroxime|azithromycin|clarithromycin|erythromycin|doxycycline|tigecycline|polymyxin|colistin|sulfamethoxazole|trimethoprim)';

-- 6. 评分
DROP TABLE IF EXISTS tmp_aps;
CREATE TEMP TABLE tmp_aps AS
SELECT a.patientunitstayid, a.acutephysiologyscore AS apsiii_score
FROM apachepatientresult a JOIN tmp_dka d ON a.patientunitstayid = d.patientunitstayid
WHERE a.acutephysiologyscore IS NOT NULL;

DROP TABLE IF EXISTS tmp_sofa;
CREATE TEMP TABLE tmp_sofa AS
SELECT patientunitstayid,
    MAX(CASE WHEN pao2/fio2 < 100 THEN 4 WHEN pao2/fio2 < 200 THEN 3
             WHEN pao2/fio2 < 300 THEN 2 WHEN pao2/fio2 < 400 THEN 1 ELSE 0 END) AS sofa_resp,
    MAX(CASE WHEN platelet < 20 THEN 4 WHEN platelet < 50 THEN 3 WHEN platelet < 100 THEN 2 WHEN platelet < 150 THEN 1 ELSE 0 END) AS sofa_coag,
    MAX(CASE WHEN bilirubin > 12 THEN 4 WHEN bilirubin > 6 THEN 3 WHEN bilirubin > 2 THEN 2 WHEN bilirubin > 1.2 THEN 1 ELSE 0 END) AS sofa_liver,
    MAX(CASE WHEN meanbp < 70 THEN 1 ELSE 0 END) AS sofa_cardio,
    MAX(CASE WHEN gcs < 6 THEN 4 WHEN gcs < 10 THEN 3 WHEN gcs < 13 THEN 2 WHEN gcs < 15 THEN 1 ELSE 0 END) AS sofa_neuro,
    MAX(CASE WHEN creatinine > 5 THEN 4 WHEN creatinine > 3.5 THEN 3 WHEN creatinine > 2 THEN 2 WHEN creatinine > 1.2 THEN 1 ELSE 0 END) AS sofa_renal
FROM (
    SELECT a.patientunitstayid,
        a.pao2, a.fio2, a.bilirubin, a.meanbp, a.creatinine,
        (COALESCE(a.eyes,0)+COALESCE(a.motor,0)+COALESCE(a.verbal,0)) AS gcs,
        lp.platelet
    FROM apacheapsvar a
    JOIN tmp_dka d ON a.patientunitstayid = d.patientunitstayid
    LEFT JOIN tmp_lab_wide lp ON a.patientunitstayid = lp.patientunitstayid
) sub
GROUP BY patientunitstayid;

DROP TABLE IF EXISTS tmp_sofa_total;
CREATE TEMP TABLE tmp_sofa_total AS
SELECT patientunitstayid,
    COALESCE(sofa_resp,0)+COALESCE(sofa_coag,0)+COALESCE(sofa_liver,0)+
    COALESCE(sofa_cardio,0)+COALESCE(sofa_neuro,0)+COALESCE(sofa_renal,0) AS sofa_day1
FROM tmp_sofa;

DROP TABLE IF EXISTS tmp_charlson;
CREATE TEMP TABLE tmp_charlson AS
SELECT d.patientunitstayid,
    (CASE WHEN d.age >= 50 THEN 1 ELSE 0 END) +
    (CASE WHEN EXISTS (SELECT 1 FROM diagnosis dx WHERE dx.patientunitstayid = d.patientunitstayid AND dx.icd9code LIKE '410%') THEN 1 ELSE 0 END) +
    (CASE WHEN EXISTS (SELECT 1 FROM diagnosis dx WHERE dx.patientunitstayid = d.patientunitstayid AND dx.icd9code LIKE '428%') THEN 1 ELSE 0 END) +
    (CASE WHEN EXISTS (SELECT 1 FROM diagnosis dx WHERE dx.patientunitstayid = d.patientunitstayid AND dx.icd9code LIKE '250.7%') THEN 1 ELSE 0 END) +
    (CASE WHEN EXISTS (SELECT 1 FROM diagnosis dx WHERE dx.patientunitstayid = d.patientunitstayid AND dx.icd9code LIKE '585%') THEN 1 ELSE 0 END) AS charlson_index
FROM tmp_dka d;

-- 7. 双结局（ICD和Sepsis-3，与论文一致）
DROP TABLE IF EXISTS tmp_sepsis_icd;
CREATE TEMP TABLE tmp_sepsis_icd AS
SELECT DISTINCT d.patientunitstayid, 1 AS sepsis_icd
FROM tmp_dka d
JOIN diagnosis dx ON d.patientunitstayid = dx.patientunitstayid
WHERE dx.diagnosispriority IN ('Primary', 'Major')
  AND (
      dx.icd9code ~ '^995\.?9[12]' OR dx.icd9code ~ '^785\.?52' OR dx.icd9code ~ '^038\.?[0-9]'
      OR dx.icd9code = '7907'
      OR LOWER(dx.diagnosisstring) LIKE '%sepsis%'
      OR LOWER(dx.diagnosisstring) LIKE '%septic shock%'
  );

DROP TABLE IF EXISTS tmp_sepsis3;
CREATE TEMP TABLE tmp_sepsis3 AS
SELECT DISTINCT d.patientunitstayid, 1 AS sepsis_sepsis3
FROM tmp_dka d
JOIN diagnosis dx ON d.patientunitstayid = dx.patientunitstayid
JOIN tmp_sofa_total s ON d.patientunitstayid = s.patientunitstayid
WHERE dx.diagnosisoffset BETWEEN 360 AND 4320
  AND dx.icd9code ~ '^0[0-9]{2}|^1[0-2][0-9]|^13[0-9]'
  AND s.sofa_day1 >= 2;

DROP TABLE IF EXISTS tmp_outcome;
CREATE TEMP TABLE tmp_outcome AS
SELECT d.patientunitstayid,
    COALESCE(si.sepsis_icd, 0) AS sepsis_icd,
    COALESCE(ss.sepsis_sepsis3, 0) AS sepsis_sepsis3
FROM tmp_dka d
LEFT JOIN tmp_sepsis_icd si ON d.patientunitstayid = si.patientunitstayid
LEFT JOIN tmp_sepsis3 ss ON d.patientunitstayid = ss.patientunitstayid;

-- 8. 最终输出（列与MIMIC对齐）
SELECT
    d.patientunitstayid,
    d.uniquepid,
    d.age,
    d.gender,
    d.ethnicity,
    d.unitadmitsource,
    v.heart_rate,
    v.respiratory_rate,
    v.sbp,
    v.dbp,
    v.map,
    v.spo2,
    v.temperature,
    l.wbc,
    l.hemoglobin,
    l.hematocrit,
    l.platelet,
    l.rdw,
    l.creatinine,
    l.bun,
    l.lactate,
    l.anion_gap,
    l.potassium,
    l.bicarbonate,
    l.glucose,
    l.ph,
    l.neutrophil_pct,
    g.gcs AS gcs_first,
    g.consciousness_disturbance,
    COALESCE(vp.vasopressor, 0) AS vasopressor,
    COALESCE(vent.mech_vent, 0) AS mech_vent,
    COALESCE(abx.antibiotic_6h, 0) AS antibiotic_6h,
    aps.apsiii_score,
    ch.charlson_index,
    s.sofa_day1,
    com.t1dm,
    com.t2dm,
    com.dpvd,
    com.hypertension,
    com.infection_history,
    sep.sepsis_icd,
    sep.sepsis_sepsis3
FROM tmp_dka d
LEFT JOIN tmp_vital_wide v ON d.patientunitstayid = v.patientunitstayid
LEFT JOIN tmp_lab_wide l ON d.patientunitstayid = l.patientunitstayid
LEFT JOIN tmp_gcs g ON d.patientunitstayid = g.patientunitstayid
LEFT JOIN tmp_comorb com ON d.patientunitstayid = com.patientunitstayid
LEFT JOIN tmp_vaso vp ON d.patientunitstayid = vp.patientunitstayid
LEFT JOIN tmp_vent vent ON d.patientunitstayid = vent.patientunitstayid
LEFT JOIN tmp_abx abx ON d.patientunitstayid = abx.patientunitstayid
LEFT JOIN tmp_aps aps ON d.patientunitstayid = aps.patientunitstayid
LEFT JOIN tmp_charlson ch ON d.patientunitstayid = ch.patientunitstayid
LEFT JOIN tmp_sofa_total s ON d.patientunitstayid = s.patientunitstayid
LEFT JOIN tmp_outcome sep ON d.patientunitstayid = sep.patientunitstayid
ORDER BY d.patientunitstayid;

```sql
WITH battle_stats AS (
  SELECT
    sb.squad_id,
    COUNT(*) AS total_battles,
    COUNT(*) FILTER (WHERE sb.result = 'victory') AS victories,
    COUNT(*) FILTER (WHERE sb.result = 'defeat') AS defeats,
    SUM(sb.squad_losses) AS casualties,
    SUM(sb.enemy_losses) AS enemy_casualties,
    ARRAY_AGG(sb.battle_id) AS battle_report_ids
  FROM SQUAD_BATTLES sb
  GROUP BY sb.squad_id
),
member_stats AS (
  SELECT
    sm.squad_id,
    COUNT(*) FILTER (WHERE sm.exit_date IS NULL) AS current_members,
    COUNT(*) AS total_members_ever,
    ROUND(COUNT(*) FILTER (WHERE sm.exit_date IS NULL) * 100.0 / NULLIF(COUNT(*),0), 2) AS retention_rate,
    ARRAY_AGG(sm.dwarf_id) FILTER (WHERE sm.exit_date IS NULL) AS member_ids
  FROM SQUAD_MEMBERS sm
  GROUP BY sm.squad_id
),
equipment_stats AS (
  SELECT
    sm.squad_id,
    ROUND(AVG(de.quality), 2) AS avg_equipment_quality,
    ARRAY_AGG(DISTINCT de.equipment_id) AS equipment_ids
  FROM SQUAD_MEMBERS sm
  JOIN DWARF_EQUIPMENT de ON sm.dwarf_id = de.dwarf_id
  WHERE sm.exit_date IS NULL
  GROUP BY sm.squad_id
),
formation_type AS (
  SELECT
    sm.squad_id,
    MAX(sm.role) AS formation_type
  FROM SQUAD_MEMBERS sm
  GROUP BY sm.squad_id
),
squad_leader AS (
  SELECT
    sm.squad_id,
    d.name AS leader_name
  FROM SQUAD_MEMBERS sm
  JOIN DWARVES d ON sm.dwarf_id = d.dwarf_id
  WHERE sm.role = 'Leader' AND sm.exit_date IS NULL
),
skill_progress AS (
  SELECT
    sm.squad_id,
    ROUND(AVG(ds.level) - MIN(ds.level), 2) AS avg_combat_skill_improvement
  FROM SQUAD_MEMBERS sm
  JOIN DWARF_SKILLS ds ON sm.dwarf_id = ds.dwarf_id
  WHERE ds.category = 'Combat'
  GROUP BY sm.squad_id
),
trainings AS (
  SELECT
    st.squad_id,
    COUNT(*) AS total_training_sessions,
    ROUND(AVG(st.effectiveness), 2) AS avg_training_effectiveness,
    ARRAY_AGG(st.training_id) AS training_ids
  FROM SQUAD_TRAININGS st
  GROUP BY st.squad_id
),
training_correlation AS (
  SELECT
    sb.squad_id,
    ROUND(CORR(st.effectiveness, CASE WHEN sb.result = 'victory' THEN 1 ELSE 0 END),2) AS training_battle_correlation
  FROM SQUAD_BATTLES sb
  JOIN SQUAD_TRAININGS st ON st.squad_id = sb.squad_id AND st.date <= sb.date
  GROUP BY sb.squad_id
)
SELECT
  b.squad_id,
  'SQUAD NAME PLACEHOLDER' AS squad_name,
  f.formation_type,
  sl.leader_name,
  b.total_battles,
  b.victories,
  ROUND(b.victories * 100.0 / NULLIF(b.total_battles,0), 2) AS victory_percentage,
  ROUND(b.casualties * 100.0 / NULLIF(b.total_battles,0), 2) AS casualty_rate,
  ROUND(b.enemy_casualties * 1.0 / NULLIF(b.casualties,1), 2) AS casualty_exchange_ratio,
  m.current_members,
  m.total_members_ever,
  m.retention_rate,
  e.avg_equipment_quality,
  t.total_training_sessions,
  t.avg_training_effectiveness,
  tc.training_battle_correlation,
  sp.avg_combat_skill_improvement,
  (
    (ROUND(b.victories * 100.0 / NULLIF(b.total_battles,0),2) * 0.3)
    + (COALESCE(e.avg_equipment_quality,0) * 0.15)
    + (COALESCE(t.avg_training_effectiveness,0) * 0.15)
    + (COALESCE(tc.training_battle_correlation,0) * 0.1)
    + (COALESCE(sp.avg_combat_skill_improvement,0) * 0.1)
    + (COALESCE(m.retention_rate,0) * 0.2)
  ) / 100.0 AS overall_effectiveness_score,
  json_build_object(
    'member_ids', m.member_ids,
    'equipment_ids', e.equipment_ids,
    'battle_report_ids', b.battle_report_ids,
    'training_ids', t.training_ids
  ) AS related_entities
FROM battle_stats b
LEFT JOIN member_stats m ON b.squad_id = m.squad_id
LEFT JOIN equipment_stats e ON b.squad_id = e.squad_id
LEFT JOIN formation_type f ON b.squad_id = f.squad_id
LEFT JOIN squad_leader sl ON b.squad_id = sl.squad_id
LEFT JOIN skill_progress sp ON b.squad_id = sp.squad_id
LEFT JOIN trainings t ON b.squad_id = t.squad_id
LEFT JOIN training_correlation tc ON b.squad_id = tc.squad_id
ORDER BY overall_effectiveness_score DESC
```

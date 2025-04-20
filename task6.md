```sql
WITH Efficiency AS (
  SELECT 
    w.workshop_id,
    w.name AS workshop_name,
    w.workshop_type,
    COUNT(DISTINCT c.craftsdwarf_id) AS num_craftsdwarves,
    COUNT(p.product_id) AS total_quantity_produced,
    SUM(p.value) AS total_production_value,
    
    COUNT(p.product_id) / NULLIF(
      EXTRACT(DAY FROM (MAX(p.production_date) - MIN(p.production_date))), 0
    ) AS daily_production_rate,
    
    SUM(p.value) / NULLIF(SUM(m.quantity_used), 0) AS value_per_material_unit,
    
    (SUM(EXTRACT(EPOCH FROM (p.completion_time - p.start_time))) /
     NULLIF(
       EXTRACT(EPOCH FROM (MAX(p.production_date) - MIN(p.production_date))) * 24 * 3600,
       0
     ) * 100) AS workshop_utilization_percent,
    
    SUM(p.quantity_produced) / NULLIF(SUM(m.quantity_used), 0) AS material_conversion_ratio,
    
    AVG(c.skill_level) AS average_craftsdwarf_skill,
    
    CORR(c.skill_level, p.quality_rating) AS skill_quality_correlation,
    
    JSON_AGG(DISTINCT c.craftsdwarf_id) AS craftsdwarf_ids,
    JSON_AGG(DISTINCT p.product_id) AS product_ids,
    JSON_AGG(DISTINCT m.material_id) AS material_ids,
    JSON_AGG(DISTINCT p.project_id) AS project_ids
  FROM 
    workshops w
    LEFT JOIN craftsdwarves c ON c.workshop_id = w.workshop_id
    LEFT JOIN products p ON p.workshop_id = w.workshop_id
    LEFT JOIN materials m ON m.product_id = p.product_id
  GROUP BY 
    w.workshop_id,
    w.name,
    w.workshop_type
)
SELECT 
  workshop_id,
  workshop_name,
  workshop_type,
  num_craftsdwarves,
  total_quantity_produced,
  total_production_value,
  ROUND(daily_production_rate::numeric, 2) AS daily_production_rate,
  ROUND(value_per_material_unit::numeric, 2) AS value_per_material_unit,
  ROUND(workshop_utilization_percent::numeric, 2) AS workshop_utilization_percent,
  ROUND(material_conversion_ratio::numeric, 2) AS material_conversion_ratio,
  ROUND(average_craftsdwarf_skill::numeric, 2) AS average_craftsdwarf_skill,
  ROUND(skill_quality_correlation::numeric, 2) AS skill_quality_correlation,
  JSON_OBJECT(
    'craftsdwarf_ids', craftsdwarf_ids,
    'product_ids', product_ids,
    'material_ids', material_ids,
    'project_ids', project_ids
  ) AS related_entities
FROM 
  Efficiency
ORDER BY 
  workshop_id;
```

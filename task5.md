```sql
SELECT 
    e.expedition_id,
    e.destination,
    e.status,
    -- процент выживших
    ROUND(
        (SUM(CASE WHEN em.survived = true THEN 1 ELSE 0 END) * 100.0 / COUNT(DISTINCT em.dwarf_id)),
        2
    ) as survival_rate,
    
    -- ценность артефактов
    COALESCE(
        (SELECT SUM(value) 
         FROM EXPEDITION_ARTIFACTS ea 
         WHERE ea.expedition_id = e.expedition_id),
        0
    ) as artifacts_value,
    
    -- расчет обнаруженных мест
    (SELECT COUNT(*) 
     FROM EXPEDITION_SITES es 
     WHERE es.expedition_id = e.expedition_id) as discovered_sites,
    
    -- расчет успешности встреч с существами
    ROUND(
        COALESCE(
            (SELECT (SUM(CASE WHEN outcome = 'Favorable' THEN 1 ELSE 0 END) * 100.0 / COUNT(*))
             FROM EXPEDITION_CREATURES ec 
             WHERE ec.expedition_id = e.expedition_id),
            0
        ),
        2
    ) as encounter_success_rate,
    
    -- подсчет длительности экспедиции
    DATEDIFF(e.return_date, e.departure_date) as expedition_duration,
    
    -- обший показатель успешности
    ROUND(
        (
            (SUM(CASE WHEN em.survived = true THEN 1 ELSE 0 END) * 100.0 / COUNT(DISTINCT em.dwarf_id)) / 100 * 0.3 + 
            (COALESCE(
                (SELECT SUM(value) 
                 FROM EXPEDITION_ARTIFACTS ea 
                 WHERE ea.expedition_id = e.expedition_id),
                0
            ) / 100000) * 0.25 +
            ((SELECT COUNT(*) 
              FROM EXPEDITION_SITES es 
              WHERE es.expedition_id = e.expedition_id) / 5) * 0.2 +
            (COALESCE(
                (SELECT (SUM(CASE WHEN outcome = 'Favorable' THEN 1 ELSE 0 END) * 100.0 / COUNT(*))
                 FROM EXPEDITION_CREATURES ec 
                 WHERE ec.expedition_id = e.expedition_id),
                0
            ) / 100) * 0.25
        ),
        2
    ) as overall_success_score,
    
    JSON_OBJECT(
        'member_ids', (SELECT JSON_ARRAYAGG(dwarf_id) 
                      FROM EXPEDITION_MEMBERS em2 
                      WHERE em2.expedition_id = e.expedition_id),
        'artifact_ids', (SELECT JSON_ARRAYAGG(artifact_id) 
                        FROM EXPEDITION_ARTIFACTS ea2 
                        WHERE ea2.expedition_id = e.expedition_id),
        'site_ids', (SELECT JSON_ARRAYAGG(site_id) 
                    FROM EXPEDITION_SITES es2 
                    WHERE es2.expedition_id = e.expedition_id)
    ) as related_entities

FROM EXPEDITIONS e
LEFT JOIN EXPEDITION_MEMBERS em ON e.expedition_id = em.expedition_id
WHERE e.status = 'Completed'
GROUP BY 
    e.expedition_id,
    e.destination,
    e.status,
    e.departure_date,
    e.return_date
ORDER BY overall_success_score DESC
```

```sql
WITH attack_data AS (
    SELECT 
        ca.attack_id,
        ca.creature_id,
        cr.name AS creature_name,
        cr.type AS creature_type,
        cr.threat_level,
        ca.date AS attack_date,
        EXTRACT(YEAR FROM ca.date) AS attack_year,
        EXTRACT(QUARTER FROM ca.date) AS attack_quarter,
        ca.casualties,
        ca.enemy_casualties,
        ca.outcome,
        ca.location_id,
        l.zone_id,
        l.zone_type,
        l.fortification_level,
        l.wall_integrity,
        l.trap_density,
        l.choke_points
    FROM creature_attacks ca
    JOIN creatures cr ON ca.creature_id = cr.creature_id
    LEFT JOIN locations l ON ca.location_id = l.location_id
    WHERE cr.active = true
),

active_threats AS (
    SELECT 
        cr.creature_type,
        cr.threat_level,
        MAX(cs.date) AS last_sighting_date,
        COUNT(DISTINCT cs.sighting_id) AS sighting_count,
        ARRAY_AGG(DISTINCT cr.creature_id) AS creature_ids,
        AVG(ct.distance_to_fortress) AS avg_distance
    FROM creatures cr
    JOIN creature_sightings cs ON cr.creature_id = cs.creature_id
    LEFT JOIN creature_territories ct ON cr.creature_id = ct.creature_id
    WHERE cs.date > CURRENT_DATE - INTERVAL '90 days'
    GROUP BY cr.creature_type, cr.threat_level
    HAVING COUNT(DISTINCT cs.sighting_id) > 0
),

-- Zone vulnerability analysis
zone_vulnerability AS (
    SELECT 
        l.zone_id,
        l.zone_type,
        l.fortification_level,
        COALESCE(COUNT(DISTINCT a.attack_id), 0) AS attack_count,
        COALESCE(SUM(CASE WHEN a.outcome = 'Defeated' THEN 1 ELSE 0 END) / 
                NULLIF(COUNT(DISTINCT a.attack_id), 0), 1.0) AS defense_success_rate,
        AVG(a.casualties) AS avg_casualties,
        AVG(a.enemy_casualties) AS avg_enemy_casualties,
        ARRAY(
            SELECT DISTINCT sm.squad_id 
            FROM squad_members sm
            JOIN squad_operations so ON sm.squad_id = so.squad_id
            WHERE so.location_id = l.zone_id
            AND so.status = 'Active'
        ) AS assigned_squads,
        ARRAY(
            SELECT DISTINCT d.structure_id 
            FROM defenses d
            WHERE d.zone_id = l.zone_id
        ) AS defense_structures
    FROM locations l
    LEFT JOIN attack_data a ON l.zone_id = a.zone_id
    GROUP BY l.zone_id, l.zone_type, l.fortification_level
),

military_readiness AS (
    SELECT 
        ms.squad_id,
        ms.name AS squad_name,
        COUNT(DISTINCT sm.dwarf_id) AS active_members,
        AVG(ds.level) FILTER (WHERE ds.skill_type = 'Combat') AS avg_combat_skill,
        COUNT(DISTINCT st.schedule_id) AS active_trainings,
        ARRAY_AGG(DISTINCT so.operation_type) AS recent_operations
    FROM military_squads ms
    LEFT JOIN squad_members sm ON ms.squad_id = sm.squad_id AND sm.exit_date IS NULL
    LEFT JOIN dwarf_skills ds ON sm.dwarf_id = ds.dwarf_id
    LEFT JOIN squad_training st ON ms.squad_id = st.squad_id AND st.date > CURRENT_DATE - INTERVAL '30 days'
    LEFT JOIN squad_operations so ON ms.squad_id = so.squad_id AND so.status = 'Active'
    GROUP BY ms.squad_id, ms.name
),

defense_effectiveness AS (
    SELECT 
        d.type AS defense_type,
        d.defense_id,
        COUNT(DISTINCT a.attack_id) AS total_encounters,
        SUM(CASE WHEN a.outcome = 'Defeated' THEN 1 ELSE 0) AS successful_defenses,
        AVG(a.enemy_casualties) AS avg_enemy_casualties,
        AVG(a.casualties) AS avg_friendly_casualties
    FROM defenses d
    LEFT JOIN attack_defenses ad ON d.defense_id = ad.defense_id
    LEFT JOIN creature_attacks ca ON ad.attack_id = ca.attack_id
    LEFT JOIN attack_data a ON ca.attack_id = a.attack_id
    GROUP BY d.type, d.defense_id
)

SELECT JSON_OBJECT(
    'total_recorded_attacks', (SELECT COUNT(*) FROM attack_data),
    'unique_attackers', (SELECT COUNT(DISTINCT creature_id) FROM attack_data),
    'overall_defense_success_rate', 
        (SELECT ROUND(100.0 * 
            SUM(CASE WHEN outcome = 'Defeated' THEN 1 ELSE 0) / 
            NULLIF(COUNT(*), 0), 2)
         FROM attack_data),
    'security_analysis', JSON_OBJECT(
        'threat_assessment', (
            SELECT JSON_OBJECT(
                'current_threat_level', 
                CASE 
                    WHEN EXISTS (SELECT 1 FROM active_threats WHERE threat_level >= 4) THEN 'High'
                    WHEN EXISTS (SELECT 1 FROM active_threats WHERE threat_level >= 2) THEN 'Moderate'
                    ELSE 'Low'
                END,
                'active_threats', COALESCE((
                    SELECT JSON_ARRAY_AGG(JSON_OBJECT(
                        'creature_type', at.creature_type,
                        'threat_level', at.threat_level,
                        'last_sighting_date', at.last_sighting_date,
                        'territory_proximity', ROUND(at.avg_distance, 1),
                        'estimated_numbers', at.sighting_count * 5,
                        'creature_ids', at.creature_ids
                    ))
                    FROM active_threats at
                ), '[]')
            )
        ),
        'vulnerability_analysis', (
            SELECT COALESCE(JSON_ARRAY_AGG(JSON_OBJECT(
                'zone_id', zv.zone_id,
                'zone_name', zv.zone_type,
                'vulnerability_score', ROUND((1 - zv.defense_success_rate) * 100, 2),
                'historical_breaches', zv.attack_count,
                'fortification_level', zv.fortification_level,
                'military_response_time', (
                    SELECT AVG(so.response_time)
                    FROM squad_operations so
                    WHERE so.zone_id = zv.zone_id
                    AND so.operation_type = 'Defense'
                ),
                'defense_coverage', JSON_OBJECT(
                    'structure_ids', zv.defense_structures,
                    'squad_ids', zv.assigned_squads
                )
            )), '[]')
            FROM zone_vulnerability zv
            WHERE zv.attack_count > 0
        ),
        'defense_effectiveness', (
            SELECT COALESCE(JSON_ARRAY_AGG(JSON_OBJECT(
                'defense_type', de.defense_type,
                'effectiveness_rate', ROUND(100.0 * de.successful_defenses / NULLIF(de.total_encounters, 0), 2),
                'avg_enemy_casualties', ROUND(de.avg_enemy_casualties, 1),
                'structure_ids', ARRAY_AGG(de.defense_id)
            )), '[]')
            FROM defense_effectiveness de
            WHERE de.total_encounters > 0
            GROUP BY de.defense_type
        ),
        'military_readiness_assessment', (
            SELECT COALESCE(JSON_ARRAY_AGG(JSON_OBJECT(
                'squad_id', mr.squad_id,
                'squad_name', mr.squad_name,
                'readiness_score', ROUND(
                    (mr.avg_combat_skill / 10.0 * 0.5) + 
                    (LEAST(mr.active_members / 10.0, 1) * 0.3) + 
                    (LEAST(mr.active_trainings / 4.0, 1) * 0.2), 
                    2
                ),
                'active_members', mr.active_members,
                'avg_combat_skill', ROUND(mr.avg_combat_skill, 1),
                'combat_effectiveness', ROUND(mr.avg_combat_skill / 10.0, 2),
                'response_coverage', (
                    SELECT COALESCE(JSON_ARRAY_AGG(JSON_OBJECT(
                        'zone_id', so.zone_id,
                        'response_time', so.avg_response_time
                    )), '[]')
                    FROM (
                        SELECT 
                            so.zone_id,
                            AVG(so.response_time) AS avg_response_time
                        FROM squad_operations so
                        WHERE so.squad_id = mr.squad_id
                        GROUP BY so.zone_id
                    ) so
                )
            )), '[]')
            FROM military_readiness mr
        ),
        'security_evolution', (
            WITH yearly_stats AS (
                SELECT 
                    EXTRACT(YEAR FROM date) AS year,
                    COUNT(*) AS total_attacks,
                    SUM(CASE WHEN outcome = 'Defeated' THEN 1 ELSE 0) AS successful_defenses,
                    SUM(casualties) AS total_casualties,
                    LAG(SUM(CASE WHEN outcome = 'Defeated' THEN 1.0 ELSE 0.0) / COUNT(*), 1) 
                        OVER (ORDER BY EXTRACT(YEAR FROM date)) AS prev_year_success_rate
                FROM creature_attacks
                GROUP BY EXTRACT(YEAR FROM date)
            )
            SELECT COALESCE(JSON_ARRAY_AGG(JSON_OBJECT(
                'year', ys.year,
                'defense_success_rate', ROUND(100.0 * ys.successful_defenses / NULLIF(ys.total_attacks, 0), 2),
                'total_attacks', ys.total_attacks,
                'casualties', ys.total_casualties,
                'year_over_year_improvement', ROUND(
                    ((ys.successful_defenses / NULLIF(ys.total_attacks, 0)) - 
                     COALESCE(ys.prev_year_success_rate, 0)) * 100, 
                    2
                )
            )), '[]')
            FROM yearly_stats ys
            ORDER BY ys.year
        )
    )
) AS security_analysis_report;
```

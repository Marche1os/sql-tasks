### 1 

```sql 
SELECT JSON_OBJECT(
  'dwarf_id', d.dwarf_id,
  'name', d.name,
  'age', d.age,
  'profession', d.profession,
  'related_entities', JSON_OBJECT(
    'skill_ids', (
      SELECT JSON_ARRAYAGG(ds.skill_id)
      FROM dwarf_skills ds
      WHERE ds.dwarf_id = d.dwarf_id
    ),
    'assignment_ids', (
      SELECT JSON_ARRAYAGG(da.assignment_id)
      FROM dwarf_assignments da
      WHERE da.dwarf_id = d.dwarf_id
      AND today() < end_date
    ),
    'squad_ids', (
      SELECT JSON_ARRAYAGG(sm.squad_id)
      FROM squad_members sm
      WHERE sm.dwarf_id = d.dwarf_id
    ),
    'equipment_ids', (
      SELECT JSON_ARRAYAGG(de.equipment_id)
      FROM dwarf_equipment de
      WHERE de.dwarf_id = d.dwarf_id
    )
  )
) as related_entities
FROM dwarves d
```

### 2

```sql
SELECT JSON_OBJECT(
  'workshop_id', w.workshop_id,
  'name', w.name,
  'type', w.type,
  'quality', w.quality,
  'related_entities', JSON_OBJECT(
    'craftsdwarf_ids', (
      SELECT JSON_ARRAYAGG(wc.dwarf_id)
      FROM workshop_craftsdwarves wc
      WHERE wc.workshop_id = w.workshop_id
    ),
    'project_ids', (
      SELECT JSON_ARRAYAGG(p.project_id)
      FROM projects p
      WHERE p.workshop_id = w.workshop_id
    ),
    'input_material_ids', (
      SELECT JSON_ARRAYAGG(wm.material_id)
      FROM workshop_materials wm
      WHERE wm.workshop_id = w.workshop_id
    ),
    'output_product_ids', (
      SELECT JSON_ARRAYAGG(wp.product_id)
      FROM workshop_products wp
      WHERE wp.workshop_id = w.workshop_id
    )
  )
) as related_entities
FROM workshops w
```

### 3

```sql
SELECT JSON_OBJECT(
  'squad_id', ms.squad_id,
  'name', ms.name,
  'formation_type', ms.formation_type,
  'leader_id', ms.leader_id,
  'related_entities', JSON_OBJECT(
    'member_ids', (
      SELECT JSON_ARRAYAGG(sm.dwarf_id)
      FROM squad_members sm
      WHERE sm.squad_id = ms.squad_id
      AND sm.active = true
    ),
    'equipment_ids', (
      SELECT JSON_ARRAYAGG(se.equipment_id)
      FROM squad_equipment se
      WHERE se.squad_id = ms.squad_id
    ),
    'operation_ids', (
      SELECT JSON_ARRAYAGG(so.operation_id)
      FROM squad_operations so
      WHERE so.squad_id = ms.squad_id
    ),
    'training_schedule_ids', (
      SELECT JSON_ARRAYAGG(st.training_id)
      FROM squad_training st
      WHERE st.squad_id = ms.squad_id
    ),
    'battle_report_ids', (
      SELECT JSON_ARRAYAGG(sb.battle_id)
      FROM squad_battles sb
      WHERE sb.squad_id = ms.squad_id
    )
  )
) as related_entities
FROM military_squads ms
```


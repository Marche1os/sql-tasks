1. SELECT * FROM squads, dwarfs WHERE dwarfs.squad_id = squads.squad_id;

2. SELECT * FROM dwarfs WHERE dwarfs.profession = 'miner' COLLATE NOCASE AND squad_id ISNULL;

3. SELECT description, MAX(priority) as priority, assigned_to, status FROM tasks WHERE status = 'pending' COLLATE NOCASE;

4. SELECT dwarf_id, dwarfs.name, age, profession, COUNT(*) AS items_count FROM dwarfs, items
WHERE dwarfs.dwarf_id = items.owner_id
GROUP BY dwarfs.dwarf_id
HAVING COUNT(*) > 0

5. SELECT squads.name, squads.mission, COUNT(dwarfs.dwarf_id) AS dwarf_count
FROM squads
LEFT JOIN dwarfs ON dwarfs.squad_id = squads.squad_id
GROUP BY squads.name;

6. SELECT dwarfs.profession, COUNT(*) AS task_count FROM dwarfs
JOIN tasks ON tasks.assigned_to = dwarfs.dwarf_id
WHERE tasks.status IN ('in_progress', 'pending')
GROUP BY dwarfs.profession
ORDER BY task_count DESC;

7. SELECT items.type, AVG(dwarfs.age) FROM dwarfs, items
WHERE dwarfs.dwarf_id = items.owner_id
GROUP BY items.type
HAVING COUNT(*);

8. SELECT dwarfs.dwarf_id, dwarfs.name, dwarfs.age, dwarfs.profession, dwarfs.squad_id FROM dwarfs
LEFT JOIN items ON dwarfs.dwarf_id = items.owner_id
WHERE dwarfs.age > (SELECT AVG(age) FROM dwarfs)
AND items.owner_id IS NULL;
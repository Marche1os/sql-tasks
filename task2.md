1. 
```sql
SELECT S.name
FROM Squads S
WHERE S.leader_id IS NULL or LENGTH(S.leader_id) < 1
```

2. 

```sql
SELECT D.name, D.age, D.profession
FROM Dwarves D
WHERE D.age > 150 AND D.profession = 'Warrior'
```

3.

```sql
SELECT D.name
FROM Dwarves D
INNER JOIN Items I
ON I.owner_id = D.dwarf_id AND I.type = 'weapon'
```

4.

```sql
SELECT 
    D.dwarf_id,
    D.name,
    T.status,
    COUNT(T.task_id) as task_count
FROM 
    Dwarves D
LEFT JOIN Tasks T 
ON D.dwarf_id = T.assigned_to
GROUP BY T.status
ORDER BY T.status;
```

5. 

```sql
SELECT DISTINCT T.description, T.status
FROM Tasks T
JOIN Dwarves D ON T.assigned_to = D.dwarf_id
JOIN Squads S ON D.squad_id = S.squad_id
WHERE S.name = 'Guardians';
```

6.

```sql
SELECT 
    D1.name AS dwarf_name,
    D2.name AS relative_name,
    R.relationship AS relationship_type
FROM Relationships R
JOIN Dwarves D1 ON R.dwarf_id = D1.dwarf_id
JOIN Dwarves D2 ON R.related_to = D2.dwarf_id;
```

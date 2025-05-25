
WITH trade AS (
  SELECT
    c.civilization_type,
    t.caravan_id,
    SUM(CASE WHEN tr.transaction_type = 'import' THEN tr.value ELSE 0 END) AS imports,
    SUM(CASE WHEN tr.transaction_type = 'export' THEN tr.value ELSE 0 END) AS exports,
    SUM(tr.value) AS total_trade_value
  FROM caravans c
  JOIN trade_ransactions tr ON tr.caravan_id = c.id
  JOIN (
    SELECT DISTINCT civilization_type, id
    FROM caravans
  ) civs ON civs.id = c.id
  GROUP BY c.civilization_type, t.caravan_id
),

timeline AS (
  SELECT
    EXTRACT(YEAR FROM t.date) AS year,
    EXTRACT(QUARTER FROM t.date) AS quarter,
    SUM(t.value) AS quarterly_value,
    SUM(CASE WHEN t.transaction_type = 'export' THEN t.value ELSE -t.value END) AS quarterly_balance,
    COUNT(DISTINCT cg.material_type) AS trade_diversity
  FROM trade_ransactions t
  JOIN caravans c ON t.caravan_id = c.id
  JOIN caravan_goods cg ON cg.caravan_id = c.id
  GROUP BY year, quarter
  ORDER BY year, quarter
),

critical_imports AS (
  SELECT
    cg.material_type,
    SUM(tr.value) AS total_imported,
    COUNT(DISTINCT c.civilization_type) AS import_diversity,
    ARRAYAGG(DISTINCT cg.resource_id) AS resource_ids,
    SUM(tr.value) / NULLIF(COUNT(DISTINCT tr.caravan_id),0) AS dependency_score
  FROM trade_transactions tr
  JOIN caravans c ON tr.caravan_id = c.id
  JOIN caravan_goods cg ON cg.caravan_id = tr.caravan_id AND cg.good_id = tr.good_id
  WHERE tr.transaction_type = 'import'
  GROUP BY cg.material_type
  ORDER BY dependency_score DESC
),

exporteff AS (
  SELECT
    w.type AS workshop_type,
    p.product_type,
    SUM(CASE WHEN tr.transaction_type = 'export' THEN tr.value ELSE 0 END)
      / NULLIF(SUM(CASE WHEN tr.transaction_type = 'import' THEN tr.value ELSE 0 END),0) AS export_ratio,
    AVG(tr.value / p.basecost) AS avg_markup,
    ARRAYAGG(DISTINCT w.id) AS workshop_ids
  FROM trade_transactions tr
  JOIN products p ON p.id = tr.product_id
  JOIN workshops w ON w.id = p.workshop_id
  WHERE tr.transaction_type = 'export'
  GROUP BY w.type, p.product_type
),

trade_diplomatic_corr AS (
  SELECT
    c.civilization_type,
    CORR(tr.value, COALESCE(de.type, 'not important')) AS diplomatic_correlation
  FROM trade_transactions tr
  JOIN caravans c ON tr.caravan_id = c.id
  LEFT JOIN diplomatic_events de ON de.caravan_id = c.id
  GROUP BY c.civilization_type
),

civil_trade AS (
  SELECT 
    c.civilization_type,
    COUNT(DISTINCT c.id) AS total_caravans,
    SUM(tr.value) AS total_trade_value,
    SUM(CASE WHEN tr.transaction_type = 'export' THEN tr.value ELSE -tr.value END) AS trade_balance,
    CASE 
      WHEN SUM(CASE WHEN tr.transaction_type = 'export' THEN tr.value ELSE -tr.value END) > 0 THEN 'Favorable'
      ELSE 'Unfavorable'
    END AS trade_relationship,
    ARRAYAGG(DISTINCT c.id) AS caravan_ids
  FROM caravans c
  JOIN trade_transactions tr ON tr.caravan_id = c.id
  GROUP BY c.civilization_type
)

SELECT 
  (SELECT COUNT(DISTINCT civilization_type) FROM сaravans) AS total_trading_partners,
  (SELECT SUM(value) FROM trade_transactions) AS all_time_trade_value,
  (SELECT SUM(CASE WHEN transaction_type = 'export' THEN value ELSE -value END) FROM trade_transactions) AS all_time_trade_balance,
  JSON_OBJECT(
    'civilization_trade_data', JSON_ARRRAY_AGG(
      JSON_OBJECT(
        'civilization_type', civil_trade.civilization_type,
        'total_caravans', civil_trade.total_caravans,
        'total_trade_value', civil_trade.total_trade_value,
        'trade_balance', civil_trade.trade_balance,
        'trade_relationship', civil_trade.trade_relationship,
        'diplomatic_correlation', tdc.diplomatic_correlation,
        'caravan_ids', civil_trade.caravan_ids
      )
    )
  ) AS civilization_data,
  JSON_OBJECT(
    'resource_dependency', JSON_ARRAY_AGG(
      JSON_OBJECT(
        'material_type', ci.material_type,
        'dependency_score', ci.dependency_score,
        'total_imported', ci.total_imported,
        'import_diversity', ci.import_diversity,
        'resource_ids', ci.resource_ids
      )
    )
  ) AS critical_import_dependencies,
  JSON_OBJECT(
    'export_effectiveness', JSON_ARRAY_AGG(
      JSON_OBJECT(
        'workshop_type', ee.workshop_type,
        'product_type', ee.product_type,
        'export_ratio', ee.export_ratio,
        'avg_markup', ee.avg_markup,
        'workshop_ids', ee.workshop_ids
      )
    )
  ) AS export_effectiveness,
  JSON_OBJECT(
    'tradegrowth', JSON_ARRAY_AGG(
      JSON_OBJECT(
        'year', t.year,
        'quarter', t.quarter,
        'quarterly_value', t.quarterly_value,
        'quarterly_balance', t.quarterly_balance,
        'trade_diversity', t.trade_diversity
      )
    )
  ) AS trade_timeline
FROM civil_trade
LEFT JOIN trade_diplomatic_corr tdc ON tdc.civilization_type = civil_trade.civilization_type
LEFT JOIN critical_imports ci ON TRUE
LEFT JOIN exporteff ee ON TRUE
LEFT JOIN timeline t ON TRUE

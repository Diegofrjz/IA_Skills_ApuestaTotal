# Contexto del Agente — Skill Databricks / Apuesta Total

> Este archivo es para uso interno del agente Claude. Documenta el progreso, decisiones tomadas, hallazgos y pendientes del proyecto de skill de datos para Apuesta Total.

---

## Proyecto

**Empresa**: Apuesta Total (betting company, Perú)
**Objetivo**: Construir una skill `/datos` para Claude Code que permita generar queries SQL correctas en Databricks para KPIs del negocio, organizada por canal (Digital, Retail, Teleservicio).
**Archivo principal**: `.claude/commands/datos.md`

---

## Estado actual — 2026-06-15

### ✅ Completado

| Área | Estado |
|------|--------|
| Arquitectura de catálogos (Silver, Copper, Gold) | ✅ Documentado |
| Canal Digital — query KPIs diarios | ✅ Validado vs reporte oficial |
| Canal Retail — query KPIs diarios (copper) | ✅ Validado vs reporte oficial |
| Canal Retail — depósitos granulares desde `tm_d_operations` | ✅ En skill, validado |
| Tablas calimaco Silver (7 tablas + 3 catálogos de máquinas) | ✅ Documentado con columnas |
| Tablas BetConstruct Silver (4 tablas) | ✅ Documentado con joins |
| Tablas db_copper Retail (3 tablas) | ✅ Documentado |
| Regla fundamental de joins calimaco (`db + company + operation + user`) | ✅ Documentado |
| Centavos vs Soles: calimaco=centavos, BetConstruct/copper=soles | ✅ Documentado |
| Player ID retail: `CONCAT(document_type, '_', document_id)` | ✅ Corregido (con guion bajo) |
| Bug `[MISSING_AGGREGATION]` en deposit_mode CTE | ✅ Corregido (3 niveles de anidamiento) |

---

## Hallazgos y Decisiones Clave

### Arquitectura de queries Retail — Enfoque Híbrido
**Decisión**: No ir 100% granular (tablas raw por proveedor) porque las copper tables ya cuadran con el reporte oficial. La estrategia es:
- **Copper**: wager, GGR, TX, jugadores, tiendas → `user_summary_retail_daily` y `user_store_summary_retail_daily`
- **`tm_d_operations`**: depósitos y retiros → fuente granular calimaco

**Por qué**: Ir completamente granular requeriría unir BetConstruct + FIRST + GoldenRace + Simulcast manualmente. Las tablas copper ya lo consolidan y están validadas.

---

### Depósitos Retail
- **Tabla**: `dlh_silver.calimaco.tm_d_operations`
- **Filtros confirmados**: `type = 'DEPOSIT'`, `method = 'ATPAYMENTSERVICE'`, `status = 'SUCCESS'`
- **Fecha**: `DATE_FORMAT(FROM_UTC_TIMESTAMP(updated_date, 'America/Lima'), 'yyyy-MM-dd')`
- **Tienda**: `RIGHT(extra, 4)` — los últimos 4 caracteres del campo `extra` contienen el ID de tienda
- **Monto**: `amount / 100.0` (está en centavos)

---

### Retiros Retail — PENDIENTE CONFIRMAR
- **Candidato**: `type = 'REDEEM'` con `status = 'SUCCESS'` (~2M registros)
- **Estado**: ⚠️ No confirmado — pendiente comparar con reporte oficial
- **Alternativa investigada**: `MANUAL` (~2.8M registros) — descartado por semántica
- **No existe** `type = 'PAYOUT'` ni `type = 'WITHDRAWAL'` en la tabla

---

### Bug Histórico — `[MISSING_AGGREGATION]`
**Patrón incorrecto** (causa error en Databricks):
```sql
ROW_NUMBER() OVER (PARTITION BY x ORDER BY COUNT(*) DESC) AS rn
FROM tabla
GROUP BY x, y
```
**Patrón correcto** (3 niveles):
```sql
-- Nivel 1: agrupar y contar
SELECT x AS fecha, y AS valor, COUNT(*) AS freq FROM tabla GROUP BY x, y
-- Nivel 2: aplicar ROW_NUMBER sobre los resultados agregados
SELECT fecha, valor, ROW_NUMBER() OVER (PARTITION BY fecha ORDER BY freq DESC) AS rn FROM (nivel 1)
-- Nivel 3: filtrar WHERE rn = 1
SELECT fecha, valor FROM (nivel 2) WHERE rn = 1
```

---

### Identificador de Jugador Retail
- **Correcto**: `CONCAT(document_type, '_', document_id)` — CON guion bajo
- **Incorrecto**: `CONCAT(document_type, document_id)` — genera colisiones entre tipos de documento
- Aplica en: `user_summary_retail_daily`, `user_store_summary_retail_daily`

---

### Tabla fuente correcta para métricas de jugadores Retail
- **Correcto**: `user_summary_retail_daily` → granularidad 1 jugador × 1 día
- **Incorrecto para jugadores**: `user_store_summary_retail_daily` → granularidad 1 jugador × 1 tienda × 1 día (genera duplicados al agregar jugadores por día)
- `user_store_summary_retail_daily` sí es correcto para contar tiendas únicas

---

### Proveedores Retail y columnas
| Proveedor | Apostado | GGR | TX |
|-----------|----------|-----|----|
| BetConstruct | `amount_sport_wager` | `amount_sport_ggr` | `count_sport_tickets` |
| FIRST Retail | `amount_first_wager` | `amount_first_ggr` | `count_first_tickets` |
| GoldenRace | `amount_goldenrace_wager` | `amount_goldenrace_ggr` | `count_goldenrace_tickets` |
| Simulcast | `amount_simulcast_wager` | `amount_simulcast_ggr` | `count_simulcast_tickets` |

- **"Deportivas"** = BetConstruct + FIRST: `amount_sport_wager + amount_first_wager`
- Todos los montos en **soles** directamente (no centavos)

---

### Tiendas FIRST Retail
- ⚠️ `tm_d_users_bets_details` (calimaco) **no tiene columna de tienda** → imposible calcular tiendas FIRST desde tablas raw
- Para tiendas FIRST usar la copper `user_store_summary_retail_daily.amount_first_wager > 0`

---

### BetConstruct State Values
- State 3: ~44M registros (principal — apuestas resueltas)
- State 4: ~6.7M registros
- State 2: ~244K registros
- State 5: ~183K registros
- State 1: ~375 registros (muy poco)
- **Decisión**: Pendiente confirmar significado de cada state. Por ahora no se filtra por state.

---

### Tablas que NO aplican a Retail
- `tm_d_daily_summary_users_details` → solo Digital (no distingue canal dentro de `company = 'ATP'`)
- `tm_d_daily_summary_users_machines` → pendiente confirmar si aplica a Retail o solo Digital

---

## Pendientes

| Tarea | Prioridad |
|-------|-----------|
| ⚠️ Confirmar `type = 'REDEEM'` como retiros retail en `tm_d_operations` | Alta |
| Schema `golden_race` — 4 tablas documentadas (`tm_d_get_earning`, `tm_d_jackpot_find`, `vc_d_games`, `tm_d_tickets`) | ✅ Documentado |
| GGR GoldenRace: `stake - paid` — pendiente confirmar contra reporte oficial | ⚠️ Pendiente |
| Join `tm_d_tickets.unit_extId = tm_d_get_earning.entityExtId` — pendiente confirmar (hay blancos en unit_extId) | ⚠️ Pendiente |
| Schema `simulcast` — 2 tablas documentadas (`tm_d_bets_detail`, `tm_d_tickets`) | ✅ Documentado |
| `simulcast.tm_d_tickets`: `transaction_type` (0-10) y `bet_status` (0-2) pendientes de confirmar | ⚠️ Pendiente |
| `simulcast.tm_d_tickets`: `user_id` pendiente confirmar join con calimaco `user` | ⚠️ Pendiente |
| Completar lógica Canal Teleservicio (schema `mvt`) | Media |
| Confirmar significado de BetConstruct State values (3, 4, 2, 5, 1) | Baja |
| Confirmar si `tm_d_daily_summary_users_machines` aplica a Retail | Baja |
| Completar GLOSARIO DE TÉRMINOS (actualmente con ejemplos placeholder) | Baja |

---

## Reglas de validación aprendidas

1. **Cuadre con reporte**: Siempre comparar query nueva contra el reporte oficial de KPIs antes de dar por válida.
2. **Moda estadística**: Calcular con 3 niveles de subquery para evitar `[MISSING_AGGREGATION]`.
3. **Fechas Perú**: Siempre `DATE(summary_date_pe)` o `FROM_UTC_TIMESTAMP(col, 'America/Lima')`. Nunca `summary_date` (España).
4. **`summary_date_pe`** es TIMESTAMP, no DATE: usar `DATE()` al filtrar o comparar.
5. **Player ID** retail siempre con guion bajo: `CONCAT(document_type, '_', document_id)`.
6. **Centavos**: calimaco tables → ÷100. BetConstruct y copper → ya en soles.

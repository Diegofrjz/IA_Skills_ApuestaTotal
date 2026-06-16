# Skill: Asistente de Datos Databricks

Eres un experto en el universo de datos de la empresa. Tienes acceso completo al catálogo de tablas, reglas de negocio, y guías de indicadores definidas en los archivos de contexto de esta skill.

---

## ARQUITECTURA DE DATOS — CAPAS Y CATÁLOGOS

El universo de datos está organizado en catálogos por capas de calidad. Siempre prioriza la capa más alta disponible para la consulta.

### Catálogos disponibles

| Catálogo | Capa | Uso | Acceso en SQL |
|----------|------|-----|---------------|
| `dlh_silver` | Silver (procesada) | **Uso principal.** Datos limpios y listos para análisis. Contiene la mayoría de tablas para reportes e indicadores. | `dlh_silver.<schema>.<tabla>` |
| `dlh_apuesta_total` | Copper (cruda útil) | Datos en crudo pero útiles. Schema principal: `db_copper`. Usar cuando la información no existe en Silver. | `dlh_apuesta_total.db_copper.<tabla>` |
| `dlh_gold` | Gold (agregada) | Uso ocasional. Datos ya agregados para dashboards específicos. | `dlh_gold.<schema>.<tabla>` |

### Regla de prioridad al elegir fuente
1. Buscar primero en `dlh_silver` — si existe la tabla, usarla
2. Si no existe en Silver, buscar en `dlh_apuesta_total.db_copper`
3. `dlh_gold` solo si el requerimiento es un agregado ya precalculado

### Schemas dentro de dlh_silver

| Schema | Descripción | Estado | Canal |
|--------|-------------|--------|-------|
| `altenar` | Data del proveedor anterior (Altenar). **Dejó de usarse el 5 de enero de 2026.** El proveedor actual para digital y retail es FIRST. Consultar solo para históricos previos a esa fecha. | ⚠️ Deprecado (desde 2026-01-05) | Digital / Retail |
| `betconstruct` | Información del proveedor de apuestas y juegos. **Exclusivo del canal Retail.** | Activo | Retail |
| `calimaco` | **Schema fundamental y más importante.** Tablas que cuadran con el Back Office (BO) de la empresa Calimaco. Fuente principal de verdad para canal Digital y también Retail. | Activo | Digital / Retail |
| `crypto` | Contiene únicamente la función `decrypt_text`, usada para decodificar columnas encriptadas en otras tablas. No contiene tablas de datos. | Activo | — |
| `draws` | Información de eventos creados en Apuesta Total que requieren inscripción (clientes inscritos, detalle de eventos, etc.). | Activo | — |
| `gestion` | Datos y detalles demográficos de las tiendas Retail: información física de locales, agencias, puntos de venta. | Activo | Retail |
| `golden_race` | Información provista por el proveedor Golden Race. | Activo | — |
| `mvt` | Información de **MiBilletera** y datos importantes del canal de **Teleservicio**. | Activo | Teleservicio |
| `optimove` | Tablas e información provista por el proveedor Optimove (CRM/marketing). | Activo | — |
| `sic` | Información que se reporta sobre jugadores **excluidos por ludopatía** (requerimiento regulatorio). | Activo | — |
| `simulcast` | Información del producto Simulcast para Retail. Uso poco frecuente. | Activo (bajo uso) | Retail |

### Canales del negocio
- **Digital**: Canal online/web. Proveedor principal: FIRST (desde 2026-01-05). Antes: Altenar.
- **Retail**: Canal presencial en tiendas físicas. Proveedores: FIRST, Betconstruct.
- **Teleservicio**: Canal por operador/agente. Datos en schema `mvt`.

---

## Cómo usar esta skill

Cuando el usuario haga una consulta de datos, debes:

1. **Identificar la intención**: ¿Qué indicador, reporte o análisis necesita?
2. **Enrutar a las tablas correctas**: Consulta la sección `GUÍA DE INDICADORES` para saber qué tablas y columnas usar
3. **Aplicar las reglas de negocio**: Consulta `REGLAS DE NEGOCIO Y JOINS` antes de generar cualquier query
4. **Elegir el catálogo correcto**: Seguir la regla de prioridad Silver → Copper → Gold
5. **Generar el SQL**: Escribe el query en sintaxis Spark SQL / Databricks SQL con el path completo `catalogo.schema.tabla`
6. **Explicar la lógica**: Indica qué tablas usaste y por qué

---

## CATÁLOGO DE TABLAS

---

### calimaco.tm_d_daily_summary_users_details
**Ruta completa**: `dlh_silver.calimaco.tm_d_daily_summary_users_details`

> **Descripción**: Consolida la actividad diaria de los usuarios: depósitos, retiros y apuestas en productos deportivos y de casino. Es la tabla de resumen más usada para análisis numéricos rápidos del canal digital. Primera tabla a revisar para ver apuestas de casino y deportivas.
> **Granularidad**: 1 fila = 1 usuario × 1 día (solo aparece si el usuario tuvo actividad ese día)
> **Actualización**: Diaria
> **Fuente origen**: Back Office Calimaco
> **Uso principal**: Canal Digital. También contiene datos Retail (company = 'ATP'), pero esta tabla no permite distinguir Digital de Retail — para esa distinción se necesita cruzar con otras tablas que tienen la columna `provider`.

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `company` | STRING | Identifica el canal | `'ATP'` = Digital (y también Retail, indistinguible aquí) / `'ATPTS'` = Teleservicio |
| `user` | STRING | ID único del usuario | FK para cruces con otras tablas |
| `summary_date` | DATE | Fecha del resumen diario **ya en hora Perú** | ✅ Usar directamente — esta tabla NO tiene `summary_date_pe`. Solo existe registro si el usuario tuvo actividad ese día |
| `periodo` | STRING | AñoMes del summary_date | Formato `YYYYMM`. Ej: `202601` |
| `total_cash_sportbets` | DOUBLE | Monto apostado en apuestas deportivas (dinero real) | — |
| `total_cash_sportwins` | DOUBLE | Monto ganado en apuestas deportivas (dinero real) | — |
| `total_bonus_sportbets` | DOUBLE | Monto apostado en apuestas deportivas (bonos) | — |
| `total_bonus_sportwins` | DOUBLE | Monto ganado en apuestas deportivas (bonos) | — |
| `total_cash_casinobets` | DOUBLE | Monto apostado en casino (dinero real) | — |
| `total_cash_casinowins` | DOUBLE | Monto ganado en casino (dinero real) | — |
| `total_bonus_casinobets` | DOUBLE | Monto apostado en casino (bonos) | — |
| `total_bonus_casinowins` | DOUBLE | Monto ganado en casino (bonos) | — |
| `count_sportbets` | BIGINT | Recuento de apuestas deportivas del día | — |
| `count_casinobets` | BIGINT | Recuento de apuestas de casino del día | — |
| `total_deposits` | DOUBLE | Monto total depositado en el día | — |
| `count_deposits` | BIGINT | Cantidad de depósitos del día | — |
| `total_payouts` | DOUBLE | Monto total retirado en el día | — |
| `count_payouts` | BIGINT | Cantidad de retiros del día | — |
| `converted_promotions` | DOUBLE | Ganancia generada por promociones convertidas | — |

**Advertencias**:
- ⚠️ Si el usuario no tuvo actividad en un día, no habrá fila para ese día — tener en cuenta al calcular promedios o series de tiempo
- ⚠️ `company = 'ATP'` incluye tanto Digital como Retail. Si necesitas solo Digital, esta tabla no es suficiente sola; debes cruzar con tablas que tengan columna `provider` (se documentarán más adelante)
- ⚠️ Para Teleservicio usar `company = 'ATPTS'`
- ⚠️ Excluir siempre `company = 'TEST'`

---

### calimaco.tm_d_daily_summary_users_machines
**Ruta completa**: `dlh_silver.calimaco.tm_d_daily_summary_users_machines`

> **Descripción**: Resumen diario del uso de máquinas de casino por cliente. Detalla apuestas, ganancias, comisiones y contribuciones/premios de jackpot por máquina. Complemento directo de `tm_d_daily_summary_users_details` para el análisis de casino digital.
> **Granularidad**: 1 fila = 1 usuario × 1 máquina × 1 día
> **Actualización**: Diaria
> **Fuente origen**: Back Office Calimaco
> **Uso principal**: Casino digital — detalle a nivel de máquina

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `company` | STRING | ID de la compañía | Excluir siempre `'TEST'` |
| `db` | STRING | Base de datos de origen | Valores: `1`, `2`, `3`, `4`, `5`. Se usa como clave de cruce junto con `user` o `machine` en joins con otras tablas que también tengan `db` |
| `user` | STRING | ID único del usuario | FK para cruces con otras tablas (siempre acompañar con `db`) |
| `summary_date` | DATE | Fecha de resumen en horario de España | ⚠️ Usar `summary_date_pe` en su lugar |
| `summary_date_pe` | DATE | Fecha de resumen en horario de Perú | ✅ Usar esta siempre |
| `machine` | STRING | ID de la máquina de casino | Para apuestas en casino canal digital |
| `currency` | STRING | Moneda | Siempre `'PEN'` |
| `wagers` | DOUBLE | Total apostado (cash + bonos) | Representa todo lo apostado, es la suma de ambos tipos |
| `winnings` | DOUBLE | Total pagado por la máquina al usuario | — |
| `jackpot_contributions` | DOUBLE | Aporte al pozo (jackpot) por cada apuesta realizada | — |
| `jackpot_prizes` | DOUBLE | Dinero ganado por el usuario al reventar un jackpot | — |
| `commissions` | DOUBLE | Comisión generada | — |

**Advertencias**:
- ⚠️ Excluir siempre `company = 'TEST'`
- ⚠️ Usar `summary_date_pe` (horario Perú), nunca `summary_date` (horario España)
- ⚠️ **Filtro obligatorio para cuadre con tabla details**: `machine NOT IN ('27453', '27469')` — excluye máquinas de FIRST y La Penka que no son casino real
- ⚠️ Joins con otras tablas: usar `user + db + company + operation` (ver regla fundamental de joins)

---

### calimaco.tm_d_users_bets_details
**Ruta completa**: `dlh_silver.calimaco.tm_d_users_bets_details`

> **Descripción**: Detalle de apuestas deportivas por usuario. Cada fila representa la última operación de un ticket (game). Incluye estado, montos apostados y ganados, tipo de apuesta, proveedor y fechas clave. Usada para análisis de apuestas deportivas tanto en Digital como Retail.
> **Granularidad**: 1 fila = 1 ticket (`game`) en su última operación. Un ticket puede tener varias operaciones intermedias pero aquí solo aparece la última.
> **Actualización**: Diaria
> **Fuente origen**: Back Office Calimaco

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `db` | STRING | Base de datos de origen | Valores: `1` al `4`. Los valores de `operation` son únicos solo dentro de cada `db` |
| `company` | STRING | ID de la compañía | Excluir siempre `'TEST'` |
| `operation` | STRING | ID de la última operación monetaria del ticket | Se concatena con `db` o `user` para ser único entre tablas. Clave de join fundamental |
| `user` | STRING | ID del usuario | FK para cruces con otras tablas |
| `game` | STRING | ID único del ticket apostado | — |
| `status` | STRING | Estado del ticket | Ver tabla de estados abajo |
| `wager` | BIGINT | Monto apostado **en centavos** | ⚠️ Dividir siempre entre 100 para obtener soles |
| `winning` | BIGINT | Monto ganado **en centavos** | ⚠️ Dividir siempre entre 100 para obtener soles |
| `total_prize` | BIGINT | Monto potencial de ganancia **en centavos** | ⚠️ Dividir siempre entre 100 para obtener soles |
| `odds` | DOUBLE | Cuota total del ticket | — |
| `provider` | STRING | Proveedor / canal de la apuesta | `'FIRST'` o `'altenar'` = Digital / `'FIRST_RETAIL'` = Retail |
| `total_selections` | BIGINT | Cantidad de selecciones del ticket | — |
| `type` | STRING | Tipo de apuesta | `'prematch'` / `'live'` / `'mixed'`. Las simples solo pueden ser prematch o live; las múltiples pueden ser mixed |
| `created_date` | TIMESTAMP | Fecha de creación del ticket | — |
| `resolved_date` | TIMESTAMP | Fecha de pago o resolución del ticket | ✅ Usar esta para análisis temporales de apuestas resueltas/pagadas |

**Estados del ticket (`status`)**:

| Categoría | Valores |
|-----------|---------|
| ❌ Inválidos — excluir | `'CANCELLED'`, `'REJECTED'`, `'VOIDED'`, `'REJECTED_PAID'`, `'VOIDED_PAID'` |
| ✅ Válidos (ganados/pagados) | `'WON'`, `'WON_PAID'`, `'CASHOUT'`, `'PAID'` |

**Filtros estándar por caso de uso**:
```sql
-- Solo apuestas válidas (excluir canceladas/rechazadas/anuladas)
WHERE status NOT IN ('CANCELLED','REJECTED','VOIDED','REJECTED_PAID','VOIDED_PAID')

-- Solo apuestas pagadas/ganadas
WHERE status IN ('WON','WON_PAID','CASHOUT','PAID')

-- Solo canal Digital
WHERE provider IN ('FIRST', 'altenar')

-- Solo canal Retail
WHERE provider = 'FIRST_RETAIL'

-- Montos en soles (siempre dividir entre 100)
SELECT wager / 100 AS wager_soles, winning / 100 AS winning_soles
```

**Advertencias**:
- ⚠️ `wager`, `winning` y `total_prize` están en **centavos** — siempre dividir entre 100
- ⚠️ Un `game` puede tener múltiples operaciones en otras tablas, pero aquí solo aparece la última — tener en cuenta al cruzar con tablas que sí muestren todas las operaciones
- ⚠️ Para análisis de fecha usar `resolved_date` (cuándo se pagó/resolvió), no `created_date` (cuándo se creó el ticket)
- ⚠️ Excluir siempre `company = 'TEST'`
- ⚠️ `provider` es la columna definitiva para distinguir Digital de Retail en esta tabla

---

---

## REGLAS DE NEGOCIO Y JOINS

### REGLA FUNDAMENTAL DE JOINS (aplica a todas las tablas calimaco)

**Regla absoluta**: si dos tablas comparten las columnas `db`, `company`, `operation` y `user`, el join **siempre debe incluir las cuatro**. No es opcional.

```sql
ON a.db        = b.db
AND a.company  = b.company
AND a.operation = b.operation
AND a.user     = b.user
```

> ⚠️ Hacer join solo por `user` o solo por `operation` genera duplicados o cruces incorrectos — los valores de `operation` son únicos solo dentro de su `db`, y `user` puede aparecer en múltiples operaciones.
> ⚠️ Esta regla aplica a cualquier par de tablas de calimaco que compartan estas columnas, sin excepción.

---

### Regla: Cuadre entre tm_d_daily_summary_users_machines y tm_d_daily_summary_users_details (casino)

Para que los montos de casino de ambas tablas cuadren, la tabla `tm_d_daily_summary_users_machines` **siempre** debe llevar el siguiente filtro obligatorio:

```sql
-- Excluir máquinas de FIRST y La Penka (no son máquinas de casino real)
WHERE machine NOT IN ('27453', '27469')
```

Sin este filtro, `wagers` de machines no coincide con `total_cash_casinobets + total_bonus_casinobets` de details, porque machines incluye datos de FIRST y La Penka que no son consideradas máquinas de casino.

```
TABLA A: dlh_silver.calimaco.tm_d_daily_summary_users_details
TABLA B: dlh_silver.calimaco.tm_d_daily_summary_users_machines
JOIN: user + db + company (+ operation si aplica)
FILTRO OBLIGATORIO EN B: machine NOT IN ('27453', '27469')
FILTRO OBLIGATORIO EN AMBAS: company != 'TEST'
FECHA A USAR: summary_date_pe
```

---

## GUÍA DE INDICADORES

<!--
  Para cada KPI o indicador del negocio, define exactamente 
  qué tablas y columnas usar. Esto evita que se use la tabla incorrecta.
-->

### Indicador: Clientes Activos
- **Tabla principal**: `esquema.clientes`
- **Columna clave**: `estado = 'ACTIVO'`
- **Columna de fecha**: `fecha_ultimo_acceso`
- **Precaución**: No usar `esquema.clientes_historico`, esa tabla tiene registros duplicados por cambios de estado

### Indicador: Ingresos del Mes
- **Tabla principal**: `esquema.transacciones`
- **Columna de monto**: `monto_neto` (NO usar `monto_bruto`, incluye impuestos)
- **Columna de fecha**: `fecha_liquidacion` (NO usar `fecha_transaccion`, puede ser del mes anterior)
- **Filtros obligatorios**: `tipo_transaccion IN ('DEPOSITO','CREDITO')` y `estado_transaccion = 'COMPLETADA'`

### Indicador: [NOMBRE DEL INDICADOR]
- **Tabla principal**: 
- **Columna clave**: 
- **Precaución**: 

---

### calimaco.tm_d_users_bets_selections
**Ruta completa**: `dlh_silver.calimaco.tm_d_users_bets_selections`

> **Descripción**: Detalle de las selecciones individuales dentro de cada ticket de apuesta deportiva. Contiene información del evento, mercado, competición, deporte y la selección elegida por el usuario. Es el complemento de `tm_d_users_bets_details` a nivel de selección.
> **Granularidad**: 1 fila = 1 selección dentro de un ticket. Apuesta simple = 1 fila por ticket. Apuesta múltiple = N filas por ticket (una por cada evento incluido).
> **Actualización**: Diaria
> **Fuente origen**: Back Office Calimaco
> **Relación**: Se une con `tm_d_users_bets_details` por `db + company + operation + user`

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `db` | STRING | Base de datos de origen del registro | Numérico, identifica la BD de procedencia |
| `company` | STRING | Canal de la operación | `'ATP'` = Digital / Retail · `'ATPTS'` = Teleservicio. Excluir `'TEST'` |
| `operation` | STRING | ID de la operación del ticket | Numérico. Clave de join con `tm_d_users_bets_details` |
| `user` | STRING | ID único del usuario | Numérico. FK para cruces con otras tablas |
| `game` | STRING | ID único del ticket | Numérico almacenado como string |
| `event` | STRING | ID del evento o partido | Un ticket múltiple tendrá varias filas con distintos `event` para el mismo `game` |
| `event_name` | STRING | Nombre del evento o partido | — |
| `market` | STRING | Tipo de selección en alto nivel | Ej: `'1x2'` (resultado del partido) |
| `market_name` | STRING | Tipo de selección en detalle | Ej: `'1x2 Primer Tiempo'` |
| `championship` | STRING | ID de la competición/liga | Numérico almacenado como string |
| `championship_name` | STRING | Nombre de la competición/liga | — |
| `sport` | STRING | Código numérico del deporte | — |
| `sport_name` | STRING | Nombre del deporte (versión raw) | Puede tener inconsistencias — preferir `sport_name_final` |
| `sport_name_final` | STRING | Nombre del deporte estandarizado | ✅ Versión más limpia de `sport_name`. Usar esta para agrupar, aunque no es 100% perfecta en todos los casos |
| `selection` | STRING | Selección específica elegida por el usuario | Ej: si el market_name dice "marcador 2-1 en primer tiempo", selection contiene el equipo favorecido |
| `odds` | DOUBLE | Cuota unitaria de esta selección | Las cuotas individuales se multiplican para obtener la cuota total del ticket |
| `is_live` | INT | Indica si la selección fue en vivo | `1` = en vivo · `0` = no fue en vivo |
| `wager` | BIGINT | Monto apostado del ticket **en centavos** | ⚠️ Dividir entre 100 para soles. Es el monto total del ticket, repetido en cada fila si el ticket es múltiple |
| `event_date_pe` | TIMESTAMP | Fecha de inicio del evento en hora peruana (UTC-5) | Formato `2026-02-28 14:45:00.000`. Usar `DATE(event_date_pe)` para filtrar por día |
| `event_end_date_pe` | TIMESTAMP | Fecha de fin del evento en hora peruana (UTC-5) | Mismo formato que `event_date_pe` |

**Advertencias**:
- ⚠️ En apuestas múltiples hay **varias filas por ticket** (una por selección). Si se suma `wager` directamente se multiplicará el monto — usar `COUNT(DISTINCT game)` o deduplicar antes de agregar montos
- ⚠️ `wager` está en **centavos** — dividir entre 100
- ⚠️ Usar `sport_name_final` en lugar de `sport_name` para agrupar por deporte, sabiendo que no es 100% precisa
- ⚠️ Las fechas de evento (`event_date_pe`, `event_end_date_pe`) son TIMESTAMP — usar `DATE()` al filtrar
- ⚠️ Excluir siempre `company = 'TEST'`

---

### calimaco.tm_d_operations
**Ruta completa**: `dlh_silver.calimaco.tm_d_operations`

> **Descripción**: Registro de todas las operaciones realizadas por los usuarios: apuestas, ganancias, depósitos, retiros, bonos, etc. Es la tabla transaccional central de Calimaco. Permite rastrear cada movimiento monetario a nivel de operación individual.
> **Granularidad**: 1 fila = 1 operación
> **Actualización**: Diaria
> **Fuente origen**: Back Office Calimaco

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `db` | STRING | ID de la base de datos | Valores: `1` al `5` |
| `company` | STRING | Canal de la operación | `'ATP'` = Digital / Retail · `'ATPTS'` = Teleservicio. Excluir `'TEST'` |
| `operation` | STRING | ID único de la operación | Clave de join con otras tablas junto con `db`, `user`, `company` |
| `user` | STRING | ID del usuario | FK para cruces con otras tablas |
| `amount` | BIGINT | Monto de la operación **en centavos** | ⚠️ Dividir entre 100 para obtener soles |
| `nominal_amount` | BIGINT | Monto nominal | Poco usado en análisis |
| `currency` | STRING | Moneda | Siempre `'PEN'` |
| `method` | STRING | Método de pago | — |
| `method_account` | STRING | Proveedor donde se realizó el depósito/gasto | — |
| `periodo` | STRING | Periodo de la operación | Formato `YYYYMM`. Ej: `202601` |
| `type` | STRING | Tipo de operación | Principalmente: `'WAGER'` (apuesta) · `'WINNING'` (ganancia). Existen otros tipos. |
| `status` | STRING | Estado de la operación | ✅ Válido: `'SUCCESS'`. Todo otro estado debe excluirse |
| `useragent` | STRING | Dispositivo/navegador del cliente | `'M'` = Mobile · `'D'` = Desktop |
| `user_promotion_id` | STRING | ID del bono aplicado a la operación | Nulo si no aplica bono |
| `provider` | STRING | Proveedor del servicio | `'FIRST'` / `'altenar'` = Digital · `'FIRST_RETAIL'` = Retail |
| `machine` | STRING | ID de la máquina donde se realizó la operación | — |
| `game` | STRING | ID del juego donde se realizó la operación | — |
| `operation_date_pe` | TIMESTAMP | Fecha de la operación en hora Perú (UTC-5) | Formato `2026-02-28 14:45:00.000`. Usar `DATE(operation_date_pe)` al filtrar |
| `updated_date_pe` | TIMESTAMP | Fecha de última actualización en hora Perú | Mismo formato TIMESTAMP |
| `updated_date` | TIMESTAMP | Fecha de actualización en UTC | Para convertir a hora Perú usar `FROM_UTC_TIMESTAMP(updated_date, 'America/Lima')`. Usado en depósitos retail |
| `extra` | STRING | Campo extra con metadata adicional | En depósitos retail (`method = 'ATPAYMENTSERVICE'`), `RIGHT(extra, 4)` contiene el ID de tienda |
| `shop` | STRING | ID de la tienda (canal Retail) | — |

**Filtros estándar por caso de uso**:
```sql
-- Solo operaciones válidas (obligatorio siempre)
WHERE status = 'SUCCESS'
  AND company != 'TEST'

-- Solo apuestas
WHERE type = 'WAGER'

-- Solo ganancias
WHERE type = 'WINNING'

-- Monto en soles
SELECT amount / 100 AS amount_soles

-- Por canal
WHERE provider IN ('FIRST', 'altenar')   -- Digital
WHERE provider = 'FIRST_RETAIL'          -- Retail
```

**Advertencias**:
- ⚠️ `amount` en **centavos** — dividir siempre entre 100
- ⚠️ Filtrar siempre `status = 'SUCCESS'` — otros estados no son operaciones válidas
- ⚠️ `operation_date_pe` es TIMESTAMP — usar `DATE()` al comparar con fechas
- ⚠️ Excluir siempre `company = 'TEST'`

---

### calimaco.tm_d_movements
**Ruta completa**: `dlh_silver.calimaco.tm_d_movements`

> **Descripción**: Registro de todos los movimientos de saldo de los usuarios, mostrando el monto actual y el monto previo por cada movimiento. Es la tabla clave para distinguir si el dinero involucrado es real (CASH) o bono, y para ver el detalle de cada movimiento con su operación y juego asociado.
> **Granularidad**: 1 fila = 1 movimiento de saldo
> **Actualización**: Diaria
> **Fuente origen**: Back Office Calimaco

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `db` | STRING | ID de la base de datos | Valores: `1` al `5` |
| `company` | STRING | Canal de la operación | `'ATP'` = Digital / Retail · `'ATPTS'` = Teleservicio. Excluir `'TEST'` |
| `movement` | STRING | ID único del movimiento | — |
| `periodo` | STRING | Periodo del movimiento | Formato `YYYYMM`. Ej: `202502` |
| `operation` | STRING | ID de la operación asociada | Clave de join con otras tablas junto con `db`, `user`, `company` |
| `user` | STRING | ID del usuario | FK para cruces con otras tablas |
| `account` | STRING | **Tipo de dinero — columna crítica** | Ver tabla de valores abajo |
| `currency` | STRING | Moneda | Siempre `'PEN'` |
| `amount` | BIGINT | Monto del movimiento **en centavos** | ⚠️ Dividir entre 100 para obtener soles |
| `previous_amount` | BIGINT | Monto del saldo justo antes de este movimiento **en centavos** | ⚠️ Dividir entre 100 para obtener soles |
| `game` | STRING | ID del juego asociado al movimiento | — |
| `movement_date_pe` | TIMESTAMP | Fecha del movimiento en hora Perú (UTC-5) | Formato `2026-02-28 14:45:00.000`. Usar `DATE(movement_date_pe)` al filtrar |

**Valores de `account` y su uso**:

| Valor | Significado | Canal |
|-------|-------------|-------|
| `'CASH'` | Dinero real | Digital |
| `'CASH-RETAIL'` | Dinero real | Retail |
| otros | Ignorar por ahora | — |

**Filtros estándar por caso de uso**:
```sql
-- Solo dinero real Digital
WHERE account = 'CASH'
  AND company != 'TEST'

-- Solo dinero real Retail
WHERE account = 'CASH-RETAIL'
  AND company != 'TEST'

-- Dinero real ambos canales
WHERE account IN ('CASH', 'CASH-RETAIL')
  AND company != 'TEST'

-- Montos en soles
SELECT amount / 100 AS amount_soles,
       previous_amount / 100 AS previous_amount_soles
```

**Advertencias**:
- ⚠️ `amount` y `previous_amount` en **centavos** — dividir entre 100
- ⚠️ `account` es la columna definitiva para saber si el dinero es real o bono — siempre filtrar explícitamente según lo que se quiere analizar
- ⚠️ Para retail usar `account = 'CASH-RETAIL'`, no `'CASH'` — son valores distintos
- ⚠️ `movement_date_pe` es TIMESTAMP — usar `DATE()` al filtrar por día
- ⚠️ Excluir siempre `company = 'TEST'`

---

### calimaco.tm_d_users_promotions
**Ruta completa**: `dlh_silver.calimaco.tm_d_users_promotions`

> **Descripción**: Registro de promociones reclamadas por los usuarios. Muestra el estado actual de cada promoción, los montos antes y después de aplicarla, el tipo de bono y las fechas clave del ciclo de vida de la promo.
> **Granularidad**: 1 fila = 1 promoción reclamada por usuario
> **Actualización**: Diaria
> **Fuente origen**: Back Office Calimaco
> **Relación con otras tablas**: Se une con `tm_d_operations` usando la regla fundamental (`operation + db + company + user`). La columna `user_promotion_id` existe en ambas tablas y permite identificar qué promoción específica se asoció a una operación, pero no es la clave de join entre tablas.

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `db` | STRING | ID de la base de datos | Valores: `1` al `5` |
| `user_promotion_id` | STRING | ID único de la promoción reclamada | PK de esta tabla. FK en `tm_d_operations` |
| `company` | STRING | Canal de la operación | `'ATP'` = Digital / Retail · `'ATPTS'` = Teleservicio. Excluir `'TEST'` |
| `user` | STRING | ID del usuario | FK para cruces con otras tablas |
| `operation` | STRING | ID de la operación asociada | Clave de join junto con `db`, `user`, `company` |
| `promotion` | STRING | Nombre de la promoción | — |
| `status` | STRING | Estado actual de la promoción | `'EXPIRED'`, `'REDEEMED'`, `'USED'`, `'NOTIFIED'`, entre otros |
| `initial_amount` | BIGINT | Monto de saldo antes de aplicar la promo **en centavos** | ⚠️ Dividir entre 100 para soles |
| `amount` | BIGINT | Monto post aplicar la promo **en centavos** | ⚠️ Dividir entre 100 para soles |
| `account` | STRING | Tipo de bono/dinero | Ver tabla de valores abajo |
| `provider` | STRING | Proveedor del servicio | Aquí puede haber muchos proveedores (uno por cada proveedor de máquinas de casino digital). Filtrar según lo que se necesite analizar |
| `currency` | STRING | Moneda | Siempre `'PEN'` |
| `external_id` | STRING | ID externo | Uso desconocido |
| `idempotence` | STRING | Clave de idempotencia | Uso desconocido |
| `last_transaction` | STRING | ID de la última transacción | Poco usado |
| `created_date_pe` | TIMESTAMP | Fecha de creación de la promo en hora Perú | Formato `2026-02-28 14:45:00.000`. Usar `DATE()` al filtrar |
| `redeemed_date_pe` | TIMESTAMP | Fecha en que el usuario reclamó la promo | Mismo formato TIMESTAMP |
| `expiration_date_pe` | TIMESTAMP | Fecha de expiración de la promo | Mismo formato TIMESTAMP |
| `activation_date_pe` | TIMESTAMP | Fecha de activación de la promo | Mismo formato TIMESTAMP |
| `expiration_activation_date_pe` | TIMESTAMP | Fecha de expiración de la activación | Mismo formato TIMESTAMP |

**Valores de `account` en esta tabla**:

| Valor | Significado | Frecuencia |
|-------|-------------|------------|
| `'BETTING-BONUS'` | Bono de apuestas deportivas | Principal |
| `'CASINO-BONUS'` | Bono de casino | Principal |
| `'CASH'` | Dinero real | Poco común |
| `'BINGO-BONUS'` | Bono de bingo | Poco común |
| `'BETTING-BONUS-RETAIL'` | Bono de apuestas deportivas Retail | Poco común |

**Filtros estándar por caso de uso**:
```sql
-- Promos activas/usadas
WHERE status IN ('REDEEMED', 'USED')
  AND company != 'TEST'

-- Solo bonos de apuestas deportivas
WHERE account = 'BETTING-BONUS'

-- Solo bonos de casino
WHERE account = 'CASINO-BONUS'

-- Montos en soles
SELECT initial_amount / 100 AS initial_amount_soles,
       amount / 100 AS amount_soles
```

**Advertencias**:
- ⚠️ `initial_amount` y `amount` en **centavos** — dividir entre 100
- ⚠️ `provider` tiene muchos más valores que en otras tablas (proveedores de máquinas de casino) — no asumir que solo son FIRST/altenar/FIRST_RETAIL
- ⚠️ Todas las fechas `_pe` son TIMESTAMP — usar `DATE()` al filtrar por día
- ⚠️ Excluir siempre `company = 'TEST'`

---

### calimaco.tc_n_machines
**Ruta completa**: `dlh_silver.calimaco.tc_n_machines`

> **Descripción**: Catálogo/dimensión de máquinas de casino digital. Contiene el nombre, tipo, proveedor y características de cada máquina. Se usa para enriquecer tablas transaccionales que solo traen el ID de máquina (`machine`).
> **Tipo de tabla**: Dimensión (catálogo, no transaccional)
> **Granularidad**: 1 fila = 1 máquina
> **Fuente origen**: Back Office Calimaco
> **Relación**: Se une a `tm_d_daily_summary_users_machines`, `tm_d_operations` y otras tablas que tengan columna `machine`, usando `machine` como clave.

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `machine` | STRING | ID único de la máquina | PK. Clave de join con tablas transaccionales |
| `name` | STRING | Nombre interno de la máquina | — |
| `web_name` | STRING | Nombre de la máquina en el sitio web | — |
| `provider` | STRING | Nombre del proveedor de la máquina | Muchos proveedores posibles |
| `sub_provider` | STRING | Nombre del subproveedor | — |
| `external_id` | STRING | ID externo de la máquina | — |
| `type` | STRING | Tipo de máquina | `'BINGO'`, `'SPORTSBETTING'`, `'VIRTUALES'`, `'CASINO'`, `'PENKA'`, `'CASINO_EN_VIVO'`, `'SLOT'`, `'TOURNAMENT'`, `'POKER'` |
| `external_launcher_id` | STRING | ID externo del lanzador | — |
| `is_live` | INT | Indica si la máquina es en vivo | `1` = en vivo · `0` = no |
| `allow_freespins` | INT | Indica si la máquina permite freespins | `1` = sí · `0` = no |

**Uso típico — enriquecer transacciones con nombre y tipo de máquina**:
```sql
SELECT
    t.user,
    m.name        AS nombre_maquina,
    m.type        AS tipo_maquina,
    m.provider    AS proveedor,
    SUM(t.wagers) AS total_apostado
FROM dlh_silver.calimaco.tm_d_daily_summary_users_machines t
JOIN dlh_silver.calimaco.tc_n_machines m ON t.machine = m.machine
WHERE machine NOT IN ('27453', '27469')  -- excluir FIRST y La Penka
  AND t.company != 'TEST'
GROUP BY 1, 2, 3, 4
```

**Advertencias**:
- ⚠️ No tiene columnas `db/company/operation/user` — la regla fundamental de joins no aplica aquí; el join es solo por `machine`
- ⚠️ Recordar aplicar el filtro `machine NOT IN ('27453', '27469')` en la tabla transaccional al cruzar, no en esta

---

### calimaco.tc_n_machines_tags
**Ruta completa**: `dlh_silver.calimaco.tc_n_machines_tags`

> **Descripción**: Catálogo de tags descriptivos para las máquinas de casino. No se une directamente a las tablas transaccionales — requiere pasar por la tabla puente `tc_n_machines_tags_machines`.
> **Tipo de tabla**: Dimensión (catálogo de tags)
> **Granularidad**: 1 fila = 1 tag

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `company` | STRING | Canal | Excluir `'TEST'` |
| `tag` | STRING | Identificador del tag | Clave de join con `tc_n_machines_tags_machines` |
| `name` | STRING | Nombre descriptivo del tag/categoría | — |
| `description` | STRING | Descripción del tag | — |

---

### calimaco.tc_n_machines_tags_machines
**Ruta completa**: `dlh_silver.calimaco.tc_n_machines_tags_machines`

> **Descripción**: Tabla puente que vincula máquinas con sus tags. Es la tabla intermedia necesaria para cruzar `tc_n_machines` con `tc_n_machines_tags`.
> **Tipo de tabla**: Puente (many-to-many)
> **Granularidad**: 1 fila = 1 relación máquina-tag

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `company` | STRING | Canal | Excluir `'TEST'` |
| `tag` | STRING | ID del tag | FK hacia `tc_n_machines_tags` |
| `machine` | STRING | ID de la máquina | FK hacia `tc_n_machines` |

**Modelo completo de joins para obtener máquina + tag + descripción**:
```sql
SELECT
    m.machine,
    m.name          AS nombre_maquina,
    m.type          AS tipo_maquina,
    t.name          AS nombre_tag,
    t.description   AS descripcion_tag
FROM dlh_silver.calimaco.tc_n_machines m
JOIN dlh_silver.calimaco.tc_n_machines_tags_machines tm
    ON m.machine = tm.machine
JOIN dlh_silver.calimaco.tc_n_machines_tags t
    ON tm.tag     = t.tag
    AND tm.company = t.company
WHERE tm.company != 'TEST'
```

**Relación entre las tres tablas de máquinas**:
```
tc_n_machines  ──(machine)──  tc_n_machines_tags_machines  ──(tag + company)──  tc_n_machines_tags
```

---

### calimaco.tm_d_users
**Ruta completa**: `dlh_silver.calimaco.tm_d_users`

> **Descripción**: Registro maestro de usuarios de la plataforma Calimaco. Contiene datos de identificación, contacto, documentos, estado regulatorio y configuración de cuenta. Es la tabla de dimensión principal para enriquecer cualquier análisis con datos del usuario.
> **Tipo de tabla**: Dimensión maestra de usuarios
> **Granularidad**: 1 fila = 1 usuario
> **Fuente origen**: Back Office Calimaco
> **Relación**: Se une a tablas transaccionales de calimaco por `user + db + company` (sin `operation` — es tabla maestra, no transaccional)

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `db` | STRING | ID de la base de datos | Clave de join |
| `user` | STRING | ID único del usuario | PK. Clave de join con todas las tablas transaccionales |
| `company` | STRING | Canal | Excluir `'TEST'` |
| `first_name` | STRING | Primer nombre | — |
| `middle_name` | STRING | Segundo nombre | — |
| `last_name_encrypted` | STRING | Apellido encriptado | Desencriptar con `dlh_silver.crypto.decrypt_text()` |
| `alias_encrypted` | STRING | Alias encriptado | Desencriptar con `dlh_silver.crypto.decrypt_text()` |
| `email_encrypted` | STRING | Correo electrónico encriptado | Desencriptar con `dlh_silver.crypto.decrypt_text()` |
| `mobile_encrypted` | STRING | Celular encriptado | Desencriptar con `dlh_silver.crypto.decrypt_text()` |
| `national_id_encrypted` | STRING | DNI encriptado | Desencriptar con `dlh_silver.crypto.decrypt_text()`. Puede usarse para cruces pero no es la clave principal |
| `national_id_type` | STRING | Tipo de documento de identidad | — |
| `address_encrypted` | STRING | Dirección domiciliaria encriptada | Desencriptar con `dlh_silver.crypto.decrypt_text()` |
| `regulatory_status` | STRING | Estado regulatorio según normativa | `'ABIERTO'`, `'CERRADO'`, `'SUSPENDIDO'`, `'PROHIBIDO'`, entre otros |
| `birthday` | TIMESTAMP | Fecha de nacimiento | Formato ISO 8601: `1994-07-11T00:00:00.000+00:00`. Usar `DATE(birthday)` para operar con ella |
| `gender` | STRING | Género | — |
| `country` | STRING | País | — |
| `currency` | STRING | Moneda | — |
| `city` | STRING | Ciudad | — |
| `state` | STRING | Departamento | — |
| `province` | STRING | Provincia | — |
| `nationality` | STRING | Nacionalidad | — |
| `promotion` | STRING | Código de promoción aplicada al registrarse | — |
| `created_date_pe` | TIMESTAMP | Fecha de creación de la cuenta en hora Perú | Formato `2026-02-28 14:45:00.000`. Usar `DATE()` al filtrar |
| `modified_date_pe` | TIMESTAMP | Fecha de última modificación en hora Perú | Mismo formato TIMESTAMP |

**Uso típico — enriquecer transacciones con datos del usuario**:
```sql
SELECT
    u.first_name,
    dlh_silver.crypto.decrypt_text(u.last_name_encrypted)  AS apellido,
    dlh_silver.crypto.decrypt_text(u.email_encrypted)      AS email,
    u.regulatory_status,
    u.gender,
    u.city,
    SUM(t.total_deposits) AS total_depositado
FROM dlh_silver.calimaco.tm_d_daily_summary_users_details t
JOIN dlh_silver.calimaco.tm_d_users u
    ON t.user    = u.user
    AND t.db     = u.db
    AND t.company = u.company
WHERE t.company != 'TEST'
  AND u.regulatory_status = 'ABIERTO'
GROUP BY 1, 2, 3, 4, 5, 6
```

**Advertencias**:
- ⚠️ Las columnas `_encrypted` deben desencriptarse con `dlh_silver.crypto.decrypt_text()` antes de mostrarlas — nunca exponer el valor encriptado directamente
- ⚠️ `birthday` tiene formato ISO 8601 con timezone (`+00:00`) — usar `DATE(birthday)` para cálculos de edad o filtros por fecha
- ⚠️ El join con tablas transaccionales es por `user + db + company` (no incluye `operation`)
- ⚠️ Excluir siempre `company = 'TEST'`

---

## LÓGICA POR CANAL

---

### Canal Digital

**Filtros de identificación**:
```sql
company = 'ATP'   -- excluye TEST implícitamente y aplica para Digital
-- En tablas que tienen provider, para mayor precisión:
-- provider IN ('FIRST', 'altenar')
```

**Fecha a usar**: `summary_date` directamente — esta tabla ya está en hora Perú y es tipo DATE. No tiene columna `summary_date_pe`.

**Definiciones de métricas validadas**:

| Métrica | Fórmula |
|---------|---------|
| Apostado Total / AD / CAS | Solo CASH: `total_cash_sportbets`, `total_cash_casinobets` |
| GGR Total | `(cash_sportbets + cash_casinobets) - (cash_sportwins + cash_casinowins)` |
| GGR AD | `total_cash_sportbets - total_cash_sportwins` |
| GGR CAS | `total_cash_casinobets - total_cash_casinowins` |
| NGR | `GGR Total - converted_promotions` |
| Hold | `GGR / Apostado` (ratio) |
| Conversión Bonos Total | `SUM(converted_promotions)` de summary_details |
| Conversión Bonos CAS | De `tm_d_users_promotions`: `account = 'CASINO-BONUS' AND provider IS NULL` OR `(provider != 'altenar' AND provider IS NOT NULL AND account IS NULL)` |
| Conversión Bonos AD | De `tm_d_users_promotions`: todo lo que NO cumple la condición casino |
| Rejuego | Ratio `Apostado / Depósitos` — cuántas veces apostó lo que depositó |
| First Time Bettors | Usuarios cuyo `MIN(summary_date_pe)` con actividad cae en ese día |
| First Time Depositors | Usuarios cuyo `MIN(summary_date_pe)` con depósito cae en ese día |
| Registros | `COUNT(DISTINCT user)` de `tm_d_users` por `DATE(created_date_pe)` |
| Jugadores Mixtos | Apostaron en AD y CAS el mismo día |
| Jugadores Solo AD | Solo apostaron en AD (`cash_casinobets = 0`) |
| Jugadores Solo CAS | Solo apostaron en CAS (`cash_sportbets = 0`) |

**Query base validado — Canal Digital (KPIs diarios)**:

```sql
WITH
-- Normalizar centavos a soles y filtrar Digital
normalized_margins AS (
    SELECT
        user,
        summary_date,                                -- ya está en hora Perú, no necesita conversión
        total_deposits       / 100.0     AS total_deposits,
        total_payouts        / 100.0     AS total_payouts,
        total_cash_sportbets / 100.0     AS total_cash_sportbets,
        total_cash_casinobets / 100.0    AS total_cash_casinobets,
        total_cash_sportwins / 100.0     AS total_cash_sportwins,
        total_cash_casinowins / 100.0    AS total_cash_casinowins,
        converted_promotions / 100.0     AS converted_promotions,
        count_deposits,
        count_payouts
    FROM dlh_silver.calimaco.tm_d_daily_summary_users_details
    WHERE company = 'ATP'
),

-- First Time Bettors
first_time_bets AS (
    SELECT user, MIN(summary_date) AS first_bet_date
    FROM normalized_margins
    WHERE total_cash_sportbets > 0 OR total_cash_casinobets > 0
    GROUP BY user
),
first_time_bettors AS (
    SELECT first_bet_date, COUNT(DISTINCT user) AS count_ftb
    FROM first_time_bets
    GROUP BY first_bet_date
),

-- First Time Depositors
first_time_deposits AS (
    SELECT user, MIN(summary_date) AS first_deposit_date
    FROM normalized_margins
    WHERE total_deposits > 0
    GROUP BY user
),
first_time_depositors AS (
    SELECT first_deposit_date, COUNT(DISTINCT user) AS count_ftd
    FROM first_time_deposits
    GROUP BY first_deposit_date
),

-- Registros (nuevos usuarios)
daily_registers AS (
    SELECT
        DATE(created_date_pe) AS register_date,
        COUNT(DISTINCT user)  AS count_registers
    FROM dlh_silver.calimaco.tm_d_users
    WHERE company = 'ATP'
    GROUP BY DATE(created_date_pe)
),

-- Conversión de bonos separada por producto
daily_bonuses AS (
    SELECT
        DATE(redeemed_date_pe) AS conversion_date,
        SUM(CASE
            WHEN (account = 'CASINO-BONUS' AND provider IS NULL)
              OR (provider != 'altenar' AND provider IS NOT NULL AND account IS NULL)
            THEN amount / 100.0 ELSE 0
        END) AS converted_promotions_casino,
        SUM(CASE
            WHEN NOT (
                (account = 'CASINO-BONUS' AND provider IS NULL)
             OR (provider != 'altenar' AND provider IS NOT NULL AND account IS NULL)
            )
            THEN amount / 100.0 ELSE 0
        END) AS converted_promotions_sports
    FROM dlh_silver.calimaco.tm_d_users_promotions
    WHERE company = 'ATP'
    GROUP BY DATE(redeemed_date_pe)
),

-- Métricas diarias consolidadas
daily_metrics AS (
    SELECT
        m.summary_date,
        WEEKOFYEAR(m.summary_date)                                          AS semana,

        -- Depósitos
        SUM(m.total_deposits)                                               AS depositos,
        SUM(m.count_deposits)                                               AS tx_depositos,
        COUNT(DISTINCT CASE WHEN m.total_deposits > 0 THEN m.user END)     AS depositantes,

        -- Retiros
        SUM(m.total_payouts)                                                AS retiros,
        SUM(m.count_payouts)                                                AS tx_retiros,
        COUNT(DISTINCT CASE WHEN m.total_payouts > 0 THEN m.user END)      AS retirantes,
        TRY_DIVIDE(SUM(m.total_payouts), SUM(m.total_deposits))            AS ratio_rd,

        -- Apostado (solo CASH)
        SUM(m.total_cash_sportbets + m.total_cash_casinobets)              AS apostado_total,
        SUM(m.total_cash_sportbets)                                         AS apostado_ad,
        SUM(m.total_cash_casinobets)                                        AS apostado_cas,

        -- GGR y NGR
        SUM(m.total_cash_sportbets + m.total_cash_casinobets
          - m.total_cash_sportwins - m.total_cash_casinowins
          - m.converted_promotions)                                         AS ngr,
        SUM(m.total_cash_sportbets + m.total_cash_casinobets
          - m.total_cash_sportwins - m.total_cash_casinowins)              AS ggr,
        SUM(m.total_cash_sportbets - m.total_cash_sportwins)               AS ggr_ad,
        SUM(m.total_cash_casinobets - m.total_cash_casinowins)             AS ggr_cas,

        -- Hold
        TRY_DIVIDE(
            SUM(m.total_cash_sportbets + m.total_cash_casinobets - m.total_cash_sportwins - m.total_cash_casinowins),
            SUM(m.total_cash_sportbets + m.total_cash_casinobets))         AS hold_total,
        TRY_DIVIDE(
            SUM(m.total_cash_sportbets - m.total_cash_sportwins),
            SUM(m.total_cash_sportbets))                                    AS hold_ad,
        TRY_DIVIDE(
            SUM(m.total_cash_casinobets - m.total_cash_casinowins),
            SUM(m.total_cash_casinobets))                                   AS hold_cas,

        -- Conversión Bonos
        SUM(m.converted_promotions)                                         AS conversion_bonos,
        MAX(db.converted_promotions_sports)                                 AS conversion_bonos_ad,
        MAX(db.converted_promotions_casino)                                 AS conversion_bonos_cas,
        TRY_DIVIDE(SUM(m.converted_promotions),
            SUM(m.total_cash_sportbets + m.total_cash_casinobets - m.total_cash_sportwins - m.total_cash_casinowins)) AS ratio_conv_ggr,
        TRY_DIVIDE(MAX(db.converted_promotions_sports),
            SUM(m.total_cash_sportbets - m.total_cash_sportwins))          AS ratio_conv_ggr_ad,
        TRY_DIVIDE(MAX(db.converted_promotions_casino),
            SUM(m.total_cash_casinobets - m.total_cash_casinowins))        AS ratio_conv_ggr_cas,
        TRY_DIVIDE(SUM(m.converted_promotions),
            SUM(m.total_cash_sportbets + m.total_cash_casinobets))         AS ratio_conv_bet,
        TRY_DIVIDE(MAX(db.converted_promotions_sports),
            SUM(m.total_cash_sportbets))                                    AS ratio_conv_bet_ad,
        TRY_DIVIDE(MAX(db.converted_promotions_casino),
            SUM(m.total_cash_casinobets))                                   AS ratio_conv_bet_cas,

        -- Jugadores
        COUNT(DISTINCT CASE WHEN m.total_cash_sportbets > 0
                            OR m.total_cash_casinobets > 0
                            THEN m.user END)                                AS jugadores_total,
        COUNT(DISTINCT CASE WHEN m.total_cash_sportbets > 0
                            THEN m.user END)                                AS jugadores_ad,
        COUNT(DISTINCT CASE WHEN m.total_cash_casinobets > 0
                            THEN m.user END)                                AS jugadores_cas,
        COUNT(DISTINCT CASE WHEN m.total_cash_sportbets > 0
                            AND m.total_cash_casinobets > 0
                            THEN m.user END)                                AS jugadores_mixtos,
        COUNT(DISTINCT CASE WHEN m.total_cash_sportbets > 0
                            AND m.total_cash_casinobets = 0
                            THEN m.user END)                                AS jugadores_solo_ad,
        COUNT(DISTINCT CASE WHEN m.total_cash_casinobets > 0
                            AND m.total_cash_sportbets = 0
                            THEN m.user END)                                AS jugadores_solo_cas,

        -- Adquisición
        MAX(ftb.count_ftb)                                                  AS first_time_bettors,
        MAX(ftd.count_ftd)                                                  AS first_time_depositors,
        MAX(dr.count_registers)                                             AS registros,

        -- Rejuego (ratio apostado / depositado)
        TRY_DIVIDE(SUM(m.total_cash_sportbets + m.total_cash_casinobets),
            SUM(m.total_deposits))                                          AS rejuego_total,
        TRY_DIVIDE(SUM(m.total_cash_sportbets), SUM(m.total_deposits))     AS rejuego_ad,
        TRY_DIVIDE(SUM(m.total_cash_casinobets), SUM(m.total_deposits))    AS rejuego_cas

    FROM normalized_margins m
    LEFT JOIN first_time_bettors  ftb ON m.summary_date = ftb.first_bet_date
    LEFT JOIN first_time_depositors ftd ON m.summary_date = ftd.first_deposit_date
    LEFT JOIN daily_registers      dr  ON m.summary_date = dr.register_date
    LEFT JOIN daily_bonuses        db  ON m.summary_date = db.conversion_date
    GROUP BY m.summary_date
)

SELECT * FROM daily_metrics
ORDER BY summary_date DESC
```

---

---

## CATÁLOGO DE TABLAS — dlh_apuesta_total.db_copper

---

### db_copper.user_store_summary_retail_daily
**Ruta completa**: `dlh_apuesta_total.db_copper.user_store_summary_retail_daily`

> **Descripción**: Tabla resumen diaria de actividad retail por usuario y tienda. Equivalente para Retail de lo que es `tm_d_daily_summary_users_details` para Digital. Consolida los 4 proveedores retail: BetConstruct, FIRST, GoldenRace y Simulcast.
> **Granularidad**: 1 fila = 1 jugador (`document_type + document_id`) × 1 tienda (`store_cc_id`) × 1 día (`summary_date`)
> **Canal**: Retail
> **Moneda**: Soles directamente (no centavos)

| Columna | Tipo | Descripción | Proveedor |
|---------|------|-------------|-----------|
| `summary_date` | DATE | Fecha del resumen | — |
| `document_type` | STRING | Tipo de documento del jugador | — |
| `document_id` | STRING | Número de documento del jugador | Junto con `document_type` forma el ID único del jugador retail |
| `store_cc_id` | STRING | ID de la tienda (centro de costos) | — |
| `amount_total_wager` | DOUBLE | Apostado total en soles | Todos los proveedores |
| `amount_sport_wager` | DOUBLE | Apostado en soles | BetConstruct |
| `amount_first_wager` | DOUBLE | Apostado en soles | FIRST Retail |
| `amount_goldenrace_wager` | DOUBLE | Apostado en soles | GoldenRace |
| `amount_simulcast_wager` | DOUBLE | Apostado en soles | Simulcast |
| `amount_total_ggr` | DOUBLE | GGR total en soles | Todos los proveedores |
| `amount_sport_ggr` | DOUBLE | GGR en soles | BetConstruct |
| `amount_goldenrace_ggr` | DOUBLE | GGR en soles | GoldenRace |
| `amount_simulcast_ggr` | DOUBLE | GGR en soles | Simulcast |
| `count_total_tickets` | BIGINT | TX total (número de tickets) | Todos los proveedores |
| `count_sport_tickets` | BIGINT | TX | BetConstruct |
| `count_goldenrace_tickets` | BIGINT | TX | GoldenRace |
| `count_simulcast_tickets` | BIGINT | TX | Simulcast |

> ⚠️ `amount_first_ggr` y `count_first_tickets` probablemente existen con ese nombre — pendiente confirmar.
> ⚠️ Jugadores se identifican con `CONCAT(document_type, '_', document_id)` — diferente al `user` ID de calimaco.
> ⚠️ Montos ya en soles — no dividir entre 100.

---

### db_copper.user_summary_retail_daily
**Ruta completa**: `dlh_apuesta_total.db_copper.user_summary_retail_daily`

> **Descripción**: Resumen diario de actividad retail **por jugador** (sin tienda). Fuente correcta para calcular métricas de jugadores, wager, GGR y tickets. Distinta de `user_store_summary_retail_daily` que tiene granularidad por jugador × tienda y genera duplicados al agregar jugadores.
> **Granularidad**: 1 fila = 1 jugador × 1 día
> **Canal**: Retail
> **Moneda**: Soles directamente

| Columna | Tipo | Descripción | Proveedor |
|---------|------|-------------|-----------|
| `summary_date` | DATE | Fecha del resumen | — |
| `document_type` | STRING | Tipo de documento | Parte del ID de jugador |
| `document_id` | STRING | Número de documento | Parte del ID de jugador |
| `amount_total_wager` | DOUBLE | Apostado total | Todos |
| `amount_sport_wager` | DOUBLE | Apostado | BetConstruct |
| `amount_first_wager` | DOUBLE | Apostado | FIRST Retail |
| `amount_goldenrace_wager` | DOUBLE | Apostado | GoldenRace |
| `amount_simulcast_wager` | DOUBLE | Apostado | Simulcast |
| `amount_total_ggr` | DOUBLE | GGR total | Todos |
| `amount_sport_ggr` | DOUBLE | GGR | BetConstruct |
| `amount_first_ggr` | DOUBLE | GGR | FIRST Retail |
| `amount_goldenrace_ggr` | DOUBLE | GGR | GoldenRace |
| `amount_simulcast_ggr` | DOUBLE | GGR | Simulcast |
| `count_total_tickets` | BIGINT | TX total | Todos |
| `count_sport_tickets` | BIGINT | TX | BetConstruct |
| `count_first_tickets` | BIGINT | TX | FIRST Retail |
| `count_goldenrace_tickets` | BIGINT | TX | GoldenRace |
| `count_simulcast_tickets` | BIGINT | TX | Simulcast |

> ⚠️ **ID de jugador**: `CONCAT(document_type, '_', document_id)` — con guion bajo como separador. Usar siempre esta forma para conteos únicos.
> ⚠️ **"Deportivas"** = BetConstruct + FIRST combinados: `amount_sport_wager + amount_first_wager`

---

### db_copper.summary_daily_deposits_payouts
**Ruta completa**: `dlh_apuesta_total.db_copper.summary_daily_deposits_payouts`

> **Descripción**: Tabla pre-agregada diaria con métricas de depósitos y retiros del canal Retail. Alternativa conveniente cuando no se necesita granularidad por operación. **La fuente preferida actualmente es `tm_d_operations`** (calimaco silver) que provee granularidad real y fue validada contra el reporte oficial.
> **Granularidad**: 1 fila = 1 día
> **Canal**: Retail

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `summary_date` | DATE | Fecha |
| `count_deposits` | BIGINT | TX de depósitos |
| `amount_deposits` | DOUBLE | Monto total depositado |
| `count_deposits_players` | BIGINT | Jugadores únicos que depositaron |
| `count_deposits_stores` | BIGINT | Tiendas con depósitos |
| `count_payouts` | BIGINT | TX de retiros |
| `amount_payouts` | DOUBLE | Monto total retirado |
| `count_payouts_players` | BIGINT | Jugadores únicos que retiraron |
| `count_payouts_stores` | BIGINT | Tiendas con retiros |
| `mode_deposit` | DOUBLE | Moda estadística del monto de depósito |
| `mode_payout` | DOUBLE | Moda estadística del monto de retiro |

---

## CATÁLOGO DE TABLAS — dlh_silver.betconstruct

> Schema exclusivo del canal **Retail**. Contiene la información provista por el proveedor BetConstruct.

---

### betconstruct.tm_d_bet
**Ruta completa**: `dlh_silver.betconstruct.tm_d_bet`

> **Descripción**: Tabla de apuestas del canal Retail para el proveedor BetConstruct. Fuente principal para calcular Apostado, GGR, TX y Jugadores de BetConstruct.
> **Granularidad**: 1 fila = 1 ticket de apuesta
> **Canal**: Retail — BetConstruct
> **Moneda**: Soles directamente (⚠️ NO centavos, a diferencia de las tablas calimaco)

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `Id` | STRING | ID único del ticket | Usar para `COUNT(Id)` = TX (número de tickets) |
| `DocumentId` | STRING | ID del jugador | Usar para `COUNT(DISTINCT DocumentId)` = Jugadores únicos |
| `Amount` | DOUBLE | Monto apostado **en soles** | ✅ Ya en soles, no dividir entre 100 |
| `WinningAmount` | DOUBLE | Monto ganado **en soles** | ✅ Ya en soles. GGR = `Amount - WinningAmount` |
| `CashDeskId` | STRING | ID del terminal/caja donde se realizó la apuesta | FK hacia `tc_n_cashdesk_locales.cashdesk_id` para obtener tienda |
| `Created_pe` | TIMESTAMP | Fecha y hora de la apuesta en hora Perú | Formato `2026-02-12 11:10:32.619`. Usar `DATE(Created_pe)` al filtrar |

> ⚠️ **Filtro de validez por status**: esta tabla tiene una columna de estado con valores numéricos. Los valores válidos a incluir/excluir están pendientes de confirmar — no aplicar filtro de status sin verificar primero.

**Join para obtener tienda desde apuesta**:
```sql
FROM dlh_silver.betconstruct.tm_d_bet b
JOIN dlh_silver.betconstruct.tc_n_cashdesk_locales c
  ON b.CashDeskId = c.cashdesk_id
```

---

### betconstruct.tc_n_cashdesk_locales
**Ruta completa**: `dlh_silver.betconstruct.tc_n_cashdesk_locales`

> **Descripción**: Dimensión que mapea cada terminal/caja (cashdesk) a su tienda retail. Necesaria para identificar en qué tienda se realizó cada apuesta de BetConstruct.
> **Tipo de tabla**: Dimensión (catálogo de terminales y tiendas)
> **Granularidad**: 1 fila = 1 terminal/caja
> **Canal**: Retail — BetConstruct
> **Relación**: Se une a `tm_d_bet` por `cashdesk_id = CashDeskId`

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `cashdesk_id` | STRING | ID del terminal/caja | PK. FK desde `tm_d_bet.CashDeskId` |
| `cc_id` | STRING | ID de la tienda (centro de costos) | Identificador único de la tienda retail. Usar para contar tiendas únicas |
| `local_name` | STRING | Nombre de la tienda | — |
| `tipo_cashdesk` | STRING | Tipo de terminal | Ej: `'CAJA'`, `'TERMINAL'`, etc. |

---

### betconstruct.tc_n_cashdesk
**Ruta completa**: `dlh_silver.betconstruct.tc_n_cashdesk`

> **Descripción**: Detalle de los terminales/cajas de BetConstruct. Usada para filtrar solo terminales activas al cruzar con las apuestas.
> **Tipo de tabla**: Dimensión (detalle de terminales)
> **Granularidad**: 1 fila = 1 terminal
> **Canal**: Retail — BetConstruct
> **Relación**: Se une a `tm_d_bet` por `Id = CashDeskId`

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `Id` | STRING | ID del terminal | PK. FK desde `tm_d_bet.CashDeskId` |
| `BetshopId` | STRING | ID del betshop padre | — |
| `IsDeleted` | BOOLEAN | Indica si el terminal fue eliminado | Filtrar `IsDeleted = false` para terminales activas |
| `IsClosed` | BOOLEAN | Indica si el terminal está cerrado | Filtrar `IsClosed = false` para terminales activas |

**Filtro estándar — solo terminales activas**:
```sql
WHERE IsDeleted = false
  AND IsClosed = false
```

---

### betconstruct.tc_n_betshop
**Ruta completa**: `dlh_silver.betconstruct.tc_n_betshop`

> **Descripción**: Catálogo de betshops (puntos de venta) de BetConstruct. Permite identificar el nombre, región, agrupación y tipo de cada betshop. Útil para segmentar resultados por zona geográfica o red de tiendas.
> **Tipo de tabla**: Dimensión (catálogo de betshops)
> **Granularidad**: 1 fila = 1 betshop
> **Canal**: Retail — BetConstruct
> **Relación**: Se une a `tc_n_cashdesk` por `Id = BetshopId`

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `Id` | STRING | ID único del betshop | PK. FK desde `tc_n_cashdesk.BetshopId` |
| `Name` | STRING | Nombre del betshop | Identificación visual de la tienda |
| `RegionId` | STRING | ID de región geográfica | Útil para segmentar análisis por zona |
| `IsDeleted` | BOOLEAN | Indica si el betshop fue eliminado | Filtrar `IsDeleted = false` |
| `IsTest` | BOOLEAN | Indica si es un betshop de prueba | Filtrar `IsTest = false` — equivalente al `company != 'TEST'` de calimaco |
| `IsAgent` | BOOLEAN | Indica si es un agente o betshop propio | `true` = agente / `false` = betshop propio |
| `GroupId` | STRING | Agrupación del betshop | Puede mapear a redes o zonas comerciales |

**Filtro estándar**:
```sql
WHERE IsDeleted = false
  AND IsTest = false
```

---

**Modelo completo de joins — BetConstruct Retail**:
```
tm_d_bet
  ├──(CashDeskId = cashdesk_id)──> tc_n_cashdesk_locales  [tienda: cc_id, local_name]
  └──(CashDeskId = Id)──────────> tc_n_cashdesk           [filtro: IsDeleted, IsClosed]
                                        └──(BetshopId = Id)──> tc_n_betshop  [región, grupo, IsTest]
```

**Query base BetConstruct con tienda y filtros completos**:
```sql
FROM dlh_silver.betconstruct.tm_d_bet b
JOIN dlh_silver.betconstruct.tc_n_cashdesk      cd  ON b.CashDeskId  = cd.Id
JOIN dlh_silver.betconstruct.tc_n_cashdesk_locales cl ON b.CashDeskId = cl.cashdesk_id
JOIN dlh_silver.betconstruct.tc_n_betshop       bs  ON cd.BetshopId  = bs.Id
WHERE cd.IsDeleted = false
  AND cd.IsClosed  = false
  AND bs.IsDeleted = false
  AND bs.IsTest    = false
```

---

### Canal Teleservicio

> ⚠️ Pendiente de documentar — filtro base: `company = 'ATPTS'`

---

### Canal Retail

**Enfoque híbrido validado (cuadra con reporte oficial de KPIs)**:

| Fuente | Uso | Granularidad |
|--------|-----|-------------|
| `user_summary_retail_daily` (copper) | Jugadores, wager, GGR, TX, prom. GGR | 1 jugador × 1 día |
| `user_store_summary_retail_daily` (copper) | Tiendas únicas por proveedor | 1 jugador × 1 tienda × 1 día |
| `tm_d_operations` (silver calimaco) | Depósitos y retiros — fuente granular | 1 operación |

> ✅ `tm_d_operations` es la fuente preferida para depósitos (`type='DEPOSIT'`, `method='ATPAYMENTSERVICE'`) y retiros (`type='REDEEM'` — ⚠️ pendiente confirmar tipo correcto). `summary_daily_deposits_payouts` (copper) existe como alternativa pre-agregada pero no es la fuente principal.
> ⚠️ Para métricas de jugadores usar `user_summary_retail_daily` — NO `user_store_summary_retail_daily`. Esta última tiene granularidad por tienda y genera duplicados de jugadores al agregar.

**Identificador de jugador**: `CONCAT(document_type, '_', document_id)` — con guion bajo.

**Proveedores y columnas**:

| Proveedor | Apostado | GGR | TX |
|-----------|----------|-----|----|
| BetConstruct | `amount_sport_wager` | `amount_sport_ggr` | `count_sport_tickets` |
| FIRST Retail | `amount_first_wager` | `amount_first_ggr` | `count_first_tickets` |
| GoldenRace | `amount_goldenrace_wager` | `amount_goldenrace_ggr` | `count_goldenrace_tickets` |
| Simulcast | `amount_simulcast_wager` | `amount_simulcast_ggr` | `count_simulcast_tickets` |
| **Total** | `amount_total_wager` | `amount_total_ggr` | `count_total_tickets` |

**"Deportivas"** = BetConstruct + FIRST agrupados: `amount_sport_wager + amount_first_wager`

**Definiciones de métricas**:

| Métrica | Fórmula |
|---------|---------|
| Hold | `ggr / wager` por proveedor |
| Ticket Promedio | `wager / tickets` por proveedor |
| Prom. GGR por jugador (ARPU) | `SUM(ggr) / COUNT(DISTINCT player_id)` del segmento |
| Depósitos / Retiros | Directamente de `summary_daily_deposits_payouts` |

**Query base validado — Canal Retail (KPIs diarios)**:

> Fuentes híbridas: `user_summary_retail_daily` (jugadores/wager/GGR/TX) + `user_store_summary_retail_daily` (tiendas) + `tm_d_operations` (depósitos y retiros granulares desde fuente calimaco)

```sql
WITH
-- Métricas de jugadores y apuestas por proveedor
player_data AS (
  SELECT
    summary_date,
    SUM(amount_total_wager)          AS amount_total_wager,
    SUM(amount_sport_wager)          AS amount_betconstruct_wager,  -- BetConstruct
    SUM(amount_first_wager)          AS amount_first_wager,         -- FIRST Retail
    SUM(amount_goldenrace_wager)     AS amount_goldenrace_wager,
    SUM(amount_simulcast_wager)      AS amount_simulcast_wager,
    SUM(amount_total_ggr)            AS amount_total_ggr,
    SUM(amount_sport_ggr)            AS amount_betconstruct_ggr,
    SUM(amount_goldenrace_ggr)       AS amount_goldenrace_ggr,
    SUM(amount_simulcast_ggr)        AS amount_simulcast_ggr,
    SUM(count_total_tickets)         AS count_total_tickets,
    SUM(count_sport_tickets)         AS count_betconstruct_tickets,
    SUM(count_goldenrace_tickets)    AS count_goldenrace_tickets,
    SUM(count_simulcast_tickets)     AS count_simulcast_tickets,

    -- Jugadores únicos por proveedor (identificados con document_type + document_id)
    COUNT(DISTINCT CASE WHEN amount_total_wager > 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_total_players,
    COUNT(DISTINCT CASE WHEN amount_sport_wager > 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_betconstruct_players,
    COUNT(DISTINCT CASE WHEN amount_goldenrace_wager > 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_goldenrace_players,
    COUNT(DISTINCT CASE WHEN amount_simulcast_wager > 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_simulcast_players,

    -- Jugadores exclusivos por proveedor
    COUNT(DISTINCT CASE WHEN amount_sport_wager > 0
      AND amount_goldenrace_wager = 0 AND amount_simulcast_wager = 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_betconstruct_only_players,
    COUNT(DISTINCT CASE WHEN amount_goldenrace_wager > 0
      AND amount_sport_wager = 0 AND amount_simulcast_wager = 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_goldenrace_only_players,
    COUNT(DISTINCT CASE WHEN amount_simulcast_wager > 0
      AND amount_sport_wager = 0 AND amount_goldenrace_wager = 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_simulcast_only_players,

    -- Jugadores en combinaciones de proveedores
    COUNT(DISTINCT CASE WHEN amount_sport_wager > 0
      AND amount_goldenrace_wager > 0 AND amount_simulcast_wager > 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_all_products_players,
    COUNT(DISTINCT CASE WHEN amount_sport_wager > 0
      AND amount_goldenrace_wager > 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_sport_goldenrace_players,
    COUNT(DISTINCT CASE WHEN amount_sport_wager > 0
      AND amount_simulcast_wager > 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_sport_simulcast_players,
    COUNT(DISTINCT CASE WHEN amount_goldenrace_wager > 0
      AND amount_simulcast_wager > 0
      THEN CONCAT(document_type, '_', document_id) END)                  AS count_goldenrace_simulcast_players,

    -- Prom. GGR por segmento de jugador
    AVG(CASE WHEN amount_total_wager > 0
      THEN amount_total_ggr END)                                     AS avg_ggr_total_players,
    AVG(CASE WHEN amount_sport_wager > 0
      AND amount_goldenrace_wager > 0 AND amount_simulcast_wager > 0
      THEN amount_total_ggr END)                                     AS avg_ggr_all_products_players,
    AVG(CASE WHEN amount_sport_wager > 0
      THEN amount_total_ggr END)                                     AS avg_ggr_betconstruct_players,
    AVG(CASE WHEN amount_goldenrace_wager > 0
      THEN amount_total_ggr END)                                     AS avg_ggr_goldenrace_players,
    AVG(CASE WHEN amount_simulcast_wager > 0
      THEN amount_total_ggr END)                                     AS avg_ggr_simulcast_players,
    AVG(CASE WHEN amount_sport_wager > 0
      AND amount_goldenrace_wager > 0
      THEN amount_total_ggr END)                                     AS avg_ggr_sport_goldenrace_players,
    AVG(CASE WHEN amount_sport_wager > 0
      AND amount_simulcast_wager > 0
      THEN amount_total_ggr END)                                     AS avg_ggr_sport_simulcast_players,
    AVG(CASE WHEN amount_goldenrace_wager > 0
      AND amount_simulcast_wager > 0
      THEN amount_total_ggr END)                                     AS avg_ggr_goldenrace_simulcast_players,
    AVG(CASE WHEN amount_sport_wager > 0
      AND amount_goldenrace_wager = 0 AND amount_simulcast_wager = 0
      THEN amount_total_ggr END)                                     AS avg_ggr_betconstruct_only_players,
    AVG(CASE WHEN amount_goldenrace_wager > 0
      AND amount_sport_wager = 0 AND amount_simulcast_wager = 0
      THEN amount_total_ggr END)                                     AS avg_ggr_goldenrace_only_players,
    AVG(CASE WHEN amount_simulcast_wager > 0
      AND amount_sport_wager = 0 AND amount_goldenrace_wager = 0
      THEN amount_total_ggr END)                                     AS avg_ggr_simulcast_only_players

  FROM dlh_apuesta_total.db_copper.user_summary_retail_daily
  GROUP BY summary_date
),

-- Métricas de tiendas
store_data AS (
  SELECT
    summary_date,
    COUNT(DISTINCT CASE WHEN amount_total_wager > 0
      THEN store_cc_id END)                                          AS count_total_stores,
    COUNT(DISTINCT CASE WHEN amount_sport_wager > 0
      THEN store_cc_id END)                                          AS count_betconstruct_stores,
    COUNT(DISTINCT CASE WHEN amount_goldenrace_wager > 0
      THEN store_cc_id END)                                          AS count_goldenrace_stores,
    COUNT(DISTINCT CASE WHEN amount_simulcast_wager > 0
      THEN store_cc_id END)                                          AS count_simulcast_stores,
    COUNT(DISTINCT CASE WHEN amount_sport_wager > 0
      AND amount_goldenrace_wager > 0 AND amount_simulcast_wager > 0
      THEN store_cc_id END)                                          AS count_all_products_stores,
    COUNT(DISTINCT CASE WHEN amount_sport_wager > 0
      AND amount_goldenrace_wager > 0
      THEN store_cc_id END)                                          AS count_sport_goldenrace_stores,
    COUNT(DISTINCT CASE WHEN amount_sport_wager > 0
      AND amount_simulcast_wager > 0
      THEN store_cc_id END)                                          AS count_sport_simulcast_stores,
    COUNT(DISTINCT CASE WHEN amount_goldenrace_wager > 0
      AND amount_simulcast_wager > 0
      THEN store_cc_id END)                                          AS count_goldenrace_simulcast_stores
  FROM dlh_apuesta_total.db_copper.user_store_summary_retail_daily
  GROUP BY summary_date
),

-- Depósitos retail desde calimaco (centavos → soles)
store_deposits_data AS (
  SELECT
    DATE_FORMAT(FROM_UTC_TIMESTAMP(updated_date, 'America/Lima'), 'yyyy-MM-dd') AS summary_date,
    COUNT(DISTINCT operation)                   AS count_deposits,
    SUM(amount) / 100.0                         AS amount_deposits,
    COUNT(DISTINCT CONCAT(db, user))            AS count_deposits_players,
    COUNT(DISTINCT RIGHT(extra, 4))             AS count_deposits_stores  -- ID tienda en últimos 4 chars de 'extra'
  FROM dlh_silver.calimaco.tm_d_operations
  WHERE type    = 'DEPOSIT'
    AND method  = 'ATPAYMENTSERVICE'
    AND status  = 'SUCCESS'
  GROUP BY summary_date
),

-- Moda de depósito por día (3 niveles para evitar [MISSING_AGGREGATION])
deposit_mode AS (
  SELECT summary_date, amount_soles AS mode_deposit
  FROM (
    SELECT
      fecha AS summary_date, amount_soles,
      ROW_NUMBER() OVER (PARTITION BY fecha ORDER BY freq DESC, amount_soles ASC) AS rn
    FROM (
      SELECT
        DATE_FORMAT(FROM_UTC_TIMESTAMP(updated_date, 'America/Lima'), 'yyyy-MM-dd') AS fecha,
        amount / 100.0 AS amount_soles,
        COUNT(*)       AS freq
      FROM dlh_silver.calimaco.tm_d_operations
      WHERE type   = 'DEPOSIT'
        AND method = 'ATPAYMENTSERVICE'
        AND status = 'SUCCESS'
      GROUP BY
        DATE_FORMAT(FROM_UTC_TIMESTAMP(updated_date, 'America/Lima'), 'yyyy-MM-dd'),
        amount / 100.0
    )
  )
  WHERE rn = 1
),

-- Retiros retail (⚠️ type='REDEEM' — pendiente confirmar si es el tipo correcto para retiros)
payouts_data AS (
  SELECT
    DATE_FORMAT(FROM_UTC_TIMESTAMP(updated_date, 'America/Lima'), 'yyyy-MM-dd') AS summary_date,
    COUNT(DISTINCT operation)          AS count_payouts,
    SUM(amount) / 100.0               AS amount_payouts,
    COUNT(DISTINCT CONCAT(db, user))  AS count_payouts_players,
    COUNT(DISTINCT RIGHT(extra, 4))   AS count_payouts_stores
  FROM dlh_silver.calimaco.tm_d_operations
  WHERE type   = 'REDEEM'
    AND status = 'SUCCESS'
  GROUP BY DATE_FORMAT(FROM_UTC_TIMESTAMP(updated_date, 'America/Lima'), 'yyyy-MM-dd')
),

payout_mode AS (
  SELECT summary_date, amount_soles AS mode_payout
  FROM (
    SELECT
      fecha AS summary_date, amount_soles,
      ROW_NUMBER() OVER (PARTITION BY fecha ORDER BY freq DESC, amount_soles ASC) AS rn
    FROM (
      SELECT
        DATE_FORMAT(FROM_UTC_TIMESTAMP(updated_date, 'America/Lima'), 'yyyy-MM-dd') AS fecha,
        amount / 100.0 AS amount_soles,
        COUNT(*)       AS freq
      FROM dlh_silver.calimaco.tm_d_operations
      WHERE type   = 'REDEEM'
        AND status = 'SUCCESS'
      GROUP BY
        DATE_FORMAT(FROM_UTC_TIMESTAMP(updated_date, 'America/Lima'), 'yyyy-MM-dd'),
        amount / 100.0
    )
  )
  WHERE rn = 1
)

SELECT
  p.summary_date,

  -- Apostado por proveedor
  amount_total_wager,
  amount_betconstruct_wager,
  amount_first_wager,
  amount_goldenrace_wager,
  amount_simulcast_wager,

  -- GGR por proveedor
  amount_total_ggr,
  amount_betconstruct_ggr,
  amount_goldenrace_ggr,
  amount_simulcast_ggr,

  -- TX por proveedor
  count_total_tickets,
  count_betconstruct_tickets,
  count_goldenrace_tickets,
  count_simulcast_tickets,

  -- Ticket promedio
  amount_betconstruct_wager / NULLIF(count_betconstruct_tickets, 0) AS avg_betconstruct_ticket,
  amount_goldenrace_wager   / NULLIF(count_goldenrace_tickets, 0)   AS avg_goldenrace_ticket,
  amount_simulcast_wager    / NULLIF(count_simulcast_tickets, 0)    AS avg_simulcast_ticket,

  -- Hold
  amount_total_ggr          / NULLIF(amount_total_wager, 0)         AS hold_total,
  amount_betconstruct_ggr   / NULLIF(amount_betconstruct_wager, 0)  AS hold_betconstruct,
  amount_goldenrace_ggr     / NULLIF(amount_goldenrace_wager, 0)    AS hold_goldenrace,
  amount_simulcast_ggr      / NULLIF(amount_simulcast_wager, 0)     AS hold_simulcast,

  -- Jugadores
  count_total_players,
  count_betconstruct_players,
  count_goldenrace_players,
  count_simulcast_players,
  count_betconstruct_only_players,
  count_goldenrace_only_players,
  count_simulcast_only_players,
  count_all_products_players,
  count_sport_goldenrace_players,
  count_sport_simulcast_players,
  count_goldenrace_simulcast_players,

  -- Prom. GGR por segmento
  avg_ggr_total_players,
  avg_ggr_all_products_players,
  avg_ggr_betconstruct_players,
  avg_ggr_goldenrace_players,
  avg_ggr_simulcast_players,
  avg_ggr_betconstruct_only_players,
  avg_ggr_goldenrace_only_players,
  avg_ggr_simulcast_only_players,
  avg_ggr_sport_goldenrace_players,
  avg_ggr_sport_simulcast_players,
  avg_ggr_goldenrace_simulcast_players,

  -- Tiendas
  count_total_stores,
  count_betconstruct_stores,
  count_goldenrace_stores,
  count_simulcast_stores,
  count_all_products_stores,
  count_sport_goldenrace_stores,
  count_sport_simulcast_stores,
  count_goldenrace_simulcast_stores,

  -- Depósitos (granular desde tm_d_operations)
  d.count_deposits,
  d.amount_deposits,
  d.count_deposits_players,
  d.count_deposits_stores,
  dm.mode_deposit,

  -- Retiros (type='REDEEM' — ⚠️ pendiente confirmar tipo correcto)
  po.count_payouts,
  po.amount_payouts,
  po.count_payouts_players,
  po.count_payouts_stores,
  pm.mode_payout

FROM player_data p
LEFT JOIN store_data          s  ON p.summary_date = s.summary_date
LEFT JOIN store_deposits_data d  ON p.summary_date = d.summary_date
LEFT JOIN deposit_mode        dm ON p.summary_date = dm.summary_date
LEFT JOIN payouts_data        po ON p.summary_date = po.summary_date
LEFT JOIN payout_mode         pm ON p.summary_date = pm.summary_date
ORDER BY p.summary_date DESC
```

---

---

## CATÁLOGO DE TABLAS — dlh_silver.golden_race

> Schema del proveedor **GoldenRace** (juegos virtuales). Exclusivo del canal **Retail**.
> ⚠️ Las entidades (`entityId`, `entityName`) son el sistema propio de GoldenRace — **no tienen join directo confirmado** con `store_cc_id` (copper) ni `cc_id` (BetConstruct).

---

### golden_race.tm_d_get_earning
**Ruta completa**: `dlh_silver.golden_race.tm_d_get_earning`

> **Descripción**: Resumen de earnings (ingresos) por entidad y período para el proveedor GoldenRace. Contiene métricas de apostado, pagado, ganado, tickets y jackpots. Fuente principal para KPIs de GoldenRace en Retail.
> **Granularidad**: 1 fila = 1 entidad (`entityId`) × 1 período (diario: `startDate_pe` → `endDate_pe`)
> **Canal**: Retail — GoldenRace
> **Moneda**: Soles directamente (no centavos)

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `entityExtId` | STRING | ID externo de la entidad en el sistema GoldenRace | Puede ser numérico, UUID o `null` |
| `entityId` | STRING | ID interno de la entidad en GoldenRace | Identificador numérico único de la entidad |
| `entityName` | STRING | Nombre de la entidad | ⚠️ Mix de dos tipos: tiendas (patrón `RedAT*`, `AUT_*_RedAT*`) y agentes/cajeros (nombres de personas). No mapea directamente a `store_cc_id` ni `cc_id` |
| `gameType` | STRING | Tipo de juego GoldenRace | — |
| `currency` | STRING | Moneda | Siempre `'PEN'` |
| `timezone` | STRING | Zona horaria del período | — |
| `startDate` | TIMESTAMP | Inicio del período en UTC | Formato ISO: `2026-01-09T00:00:00.000+00:00`. ⚠️ Usar `startDate_pe` |
| `endDate` | TIMESTAMP | Fin del período en UTC | Formato ISO: `2026-01-10T00:00:00.000+00:00`. ⚠️ Usar `endDate_pe` |
| `startDate_pe` | TIMESTAMP | Inicio del período en hora Perú | ✅ Usar esta para filtrar por fecha |
| `endDate_pe` | TIMESTAMP | Fin del período en hora Perú | ✅ Usar esta |
| `minTimePlayed` | TIMESTAMP | Primera apuesta del período (UTC) | ⚠️ Usar `minTimePlayed_pe` |
| `maxTimePlayed` | TIMESTAMP | Última apuesta del período (UTC) | ⚠️ Usar `maxTimePlayed_pe` |
| `minTimePlayed_pe` | TIMESTAMP | Primera apuesta del período en hora Perú | — |
| `maxTimePlayed_pe` | TIMESTAMP | Última apuesta del período en hora Perú | — |
| `stake` | DOUBLE | Monto total apostado **en soles** | ✅ Apostado GoldenRace |
| `stakeCancelled` | DOUBLE | Monto apostado en tickets cancelados **en soles** | — |
| `paid` | DOUBLE | Monto total pagado a jugadores **en soles** | Usar para GGR: `stake - paid` |
| `won` | DOUBLE | Monto ganado por jugadores **en soles** | ⚠️ Puede diferir de `paid` — pendiente confirmar cuál usar para GGR |
| `playedTickets` | BIGINT | Total de tickets jugados (TX) | ✅ Usar para conteo de tickets |
| `wonTickets` | BIGINT | Tickets ganadores | — |
| `lostTickets` | BIGINT | Tickets perdedores | — |
| `cancelledtickets` | BIGINT | Tickets cancelados | — |
| `bonus` | DOUBLE | Monto de bonos | ⚠️ Siempre `0` — ignorar |
| `promotionalStake` | DOUBLE | Apuesta promocional | — |
| `jackpot` | DOUBLE | Monto de jackpot | — |
| `jackpotContribution` | DOUBLE | Contribución al jackpot | — |
| `jackpotPaid` | DOUBLE | Jackpot pagado | — |
| `megajackpot` | DOUBLE | Monto de mega jackpot | — |
| `megajackpotContribution` | DOUBLE | Contribución al mega jackpot | — |
| `megajackpotPaid` | DOUBLE | Mega jackpot pagado | — |
| `lapsed` | DOUBLE | Monto vencido/caducado | — |
| `targetBalance` | DOUBLE | Balance objetivo | Uso desconocido |
| `taxes` | DOUBLE | Impuestos sobre apuestas | — |
| `taxesCancelled` | DOUBLE | Impuestos de tickets cancelados | — |
| `taxesPaidOut` | DOUBLE | Impuestos pagados en premios | — |
| `_fecha_ingesta` | TIMESTAMP | Fecha de carga del registro en el datalake | Columna técnica — no usar en análisis |

**Filtros estándar por caso de uso**:
```sql
-- Apostado GoldenRace del día
WHERE DATE(startDate_pe) = '2026-06-01'

-- GGR GoldenRace (⚠️ fórmula pendiente confirmar: stake - paid vs stake - won)
SELECT
  DATE(startDate_pe)    AS fecha,
  SUM(stake)            AS apostado,
  SUM(paid)             AS pagado,
  SUM(stake - paid)     AS ggr,       -- pendiente confirmar
  SUM(playedTickets)    AS tickets
FROM dlh_silver.golden_race.tm_d_get_earning
WHERE DATE(startDate_pe) = '2026-06-01'
GROUP BY DATE(startDate_pe)
```

**Advertencias**:
- ⚠️ Montos (`stake`, `paid`, `won`, etc.) ya en **soles** — no dividir entre 100
- ⚠️ Usar siempre `startDate_pe` para filtrar por fecha — nunca `startDate` (UTC)
- ⚠️ `entityName` mezcla tiendas y agentes — no usar directamente para contar tiendas únicas sin filtrar por patrón de nombre
- ⚠️ GGR = `stake - paid` está **pendiente de confirmar** contra reporte oficial
- ⚠️ `bonus` siempre es `0` — no aporta información
- ⚠️ No hay join confirmado con tablas de otros proveedores o copper tables

---

### golden_race.tm_d_jackpot_find
**Ruta completa**: `dlh_silver.golden_race.tm_d_jackpot_find`

> **Descripción**: Catálogo/snapshot del estado de los jackpots por entidad en GoldenRace. Muestra el balance actual, montos de reserva y configuración de cada jackpot. Tabla de referencia para enriquecer análisis de jackpots de `tm_d_get_earning`.
> **Tipo de tabla**: Dimensión / snapshot de estado (no transaccional)
> **Granularidad**: 1 fila = 1 jackpot (`jackpotId`) × 1 entidad (`entityId`)
> **Canal**: Retail — GoldenRace
> **Relación**: Se une a `tm_d_get_earning` por `entityId`

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `entityId` | STRING | ID de la entidad GoldenRace | FK hacia `tm_d_get_earning.entityId` |
| `jackpotId` | STRING | ID único del jackpot | PK junto con `entityId` |
| `classType` | STRING | Tipo/clase del jackpot | Distingue jackpot regular de mega jackpot u otras categorías |
| `balance` | DOUBLE | Balance actual del jackpot | Monto acumulado disponible en el jackpot |
| `amountFront` | DOUBLE | Monto mostrado al jugador | Valor visible en la interfaz del terminal |
| `amountReserve` | DOUBLE | Monto en reserva | Parte del jackpot guardada en reserva |
| `initAmount` | DOUBLE | Monto inicial del jackpot | Valor con el que arranca el jackpot tras ser ganado |
| `lastChange` | TIMESTAMP | Última modificación del jackpot | — |
| `lastConsolidate` | TIMESTAMP | Última consolidación | — |
| `code` | STRING | Código de moneda | `'PEN'` |
| `currency` | STRING | Moneda | `'PEN'` (duplicado de `code`) |
| `decimals` | INT | Número de decimales de los montos | — |

**Uso típico — enriquecer análisis de jackpots**:
```sql
SELECT
  e.entityId,
  e.entityName,
  j.jackpotId,
  j.classType,
  j.balance        AS jackpot_balance_actual,
  j.amountFront    AS jackpot_visible_jugador,
  j.initAmount     AS jackpot_monto_inicial
FROM dlh_silver.golden_race.tm_d_get_earning e
JOIN dlh_silver.golden_race.tm_d_jackpot_find j ON e.entityId = j.entityId
WHERE DATE(e.startDate_pe) = '2026-06-01'
```

**Advertencias**:
- ⚠️ Es una tabla de estado (snapshot) — no tiene columna de fecha de período como `tm_d_get_earning`. Representa el estado más reciente de cada jackpot.
- ⚠️ `code` y `currency` contienen lo mismo (`'PEN'`) — usar cualquiera o ignorar ambas
- ⚠️ No está claro si los montos están en soles o en unidades con decimales — verificar con `decimals` si el valor es distinto de 2

---

### golden_race.vc_d_games
**Ruta completa**: `dlh_silver.golden_race.vc_d_games`

> **Descripción**: Catálogo de juegos disponibles en GoldenRace para Retail. Permite enriquecer análisis de `tm_d_get_earning` con el nombre y tipo de juego. Tabla de dimensión pura.
> **Tipo de tabla**: Dimensión (catálogo de juegos)
> **Granularidad**: 1 fila = 1 juego
> **Canal**: Retail — GoldenRace
> **Relación**: Join con `tm_d_get_earning` por `game_type = gameType`

| Columna | Tipo | Descripción | Valores conocidos |
|---------|------|-------------|-------------------|
| `game_id` | STRING | ID único del juego | Numérico |
| `game_desc` | STRING | Nombre/descripción del juego | Ej: `'Italy League 2025 - OS32'`, `'GreyhoundsV2'`, `'Keno Deluxe'` |
| `game_type` | STRING | Código del tipo de juego | Ver tabla de valores abajo |
| `competition_type` | STRING | Tipo de competición | Solo aplica a fútbol (`CH`): `'LEAGUE'`, `'CHAMPION'`, `'SOCCER'`. `null` para otros tipos |
| `competition_subtype` | STRING | Subtipo de competición | Mismos valores que `competition_type`. `null` para no-fútbol |

**Valores de `game_type`**:

| Código | Tipo de juego |
|--------|---------------|
| `CH` | Fútbol virtual (ligas, champions, naciones) |
| `DOG` | Carreras de galgos (Greyhounds) |
| `PENALTY` | Penales |
| `SN` | Spin2Win (giro/ruleta) |
| `KN` | Keno |
| `HORSE` | Carreras de caballos |
| `SX` | Perfect Six |

**Uso típico — enriquecer earning con nombre de juego**:
```sql
SELECT
  DATE(e.startDate_pe)  AS fecha,
  g.game_desc           AS juego,
  g.game_type           AS tipo_juego,
  g.competition_type,
  SUM(e.stake)          AS apostado,
  SUM(e.stake - e.paid) AS ggr,
  SUM(e.playedTickets)  AS tickets
FROM dlh_silver.golden_race.tm_d_get_earning e
LEFT JOIN dlh_silver.golden_race.vc_d_games g ON e.gameType = g.game_type
WHERE DATE(e.startDate_pe) = '2026-06-01'
GROUP BY 1, 2, 3, 4
ORDER BY apostado DESC
```

**Advertencias**:
- ⚠️ `competition_type` y `competition_subtype` son `null` para todos los juegos que no son fútbol (`CH`) — no usar como filtro general sin considerar esto
- ⚠️ El join con `tm_d_get_earning` es por `gameType` (columna en earning) = `game_type` (columna en este catálogo). Verificar si el join es 1:1 o 1:N antes de agregar

---

### golden_race.tm_d_tickets
**Ruta completa**: `dlh_silver.golden_race.tm_d_tickets`

> **Descripción**: Detalle de tickets a nivel individual de GoldenRace. Es la tabla más granular del schema — permite análisis por ticket, cajero, terminal y estado. Complementa `tm_d_get_earning` con la visión a nivel de operación individual.
> **Granularidad**: 1 fila = 1 ticket (`ticketId`)
> **Canal**: Retail — GoldenRace
> **Moneda**: Soles directamente (no centavos)
> **Relación con tm_d_get_earning**: join por `unit_extId = entityExtId` — ⚠️ pendiente confirmar validez (hay `unit_extId` en blanco)

| Columna | Tipo | Descripción | Valores / Notas |
|---------|------|-------------|-----------------|
| `ticketId` | STRING | ID único del ticket | PK |
| `status` | STRING | Estado del ticket | Ver tabla de estados abajo |
| `numBets` | INT | Número de apuestas dentro del ticket | — |
| `stake` | DOUBLE | Monto apostado **en soles** | ✅ Apostado del ticket |
| `stakePromotional` | DOUBLE | Monto apostado con saldo promocional | — |
| `stakeTaxes` | DOUBLE | Monto de impuestos sobre el stake | — |
| `stakeTaxesPercent` | DOUBLE | Porcentaje de impuesto aplicado al stake | — |
| `advancedInfo_groupId` | STRING | ID de grupo para apuestas combinadas/parlay | — |
| `sellStaff_id` | STRING | ID del cajero/agente que vendió el ticket | Nivel de agente |
| `sellStaff_extId` | STRING | ID externo del cajero/agente | — |
| `sellStaff_name` | STRING | Nombre del cajero/agente | — |
| `unit_extId` | STRING | ID externo del terminal/unidad | ⚠️ Tiene valores en blanco — join con `tm_d_get_earning.entityExtId` pendiente confirmar |
| `unit_name` | STRING | Nombre del terminal/unidad | Texto libre — no usar para joins |
| `timePrint_pe` | TIMESTAMP | Fecha/hora de impresión del ticket (Perú) | ✅ Usar para filtrar por fecha de apuesta |
| `timeRegister_pe` | TIMESTAMP | Fecha/hora de registro del ticket (Perú) | Alternativa a `timePrint_pe` |
| `timeResolved_pe` | TIMESTAMP | Fecha/hora de resolución del ticket (Perú) | Cuando se determinó el resultado |
| `timePaid_pe` | TIMESTAMP | Fecha/hora de pago al jugador (Perú) | Solo aplica a `status IN ('WON', 'PAIDOUT')` |
| `timeCancelled_pe` | TIMESTAMP | Fecha/hora de cancelación (Perú) | Solo aplica a `status = 'CANCELLED'` |
| `timeClosedMarket_pe` | TIMESTAMP | Fecha/hora de cierre del mercado (Perú) | — |

**Estados del ticket (`status`)**:

| Estado | Significado | Incluir en análisis |
|--------|-------------|---------------------|
| `WON` | Ganado — pendiente de cobro | ✅ Sí |
| `PAIDOUT` | Ganado y cobrado | ✅ Sí |
| `LOST` | Perdido | ✅ Sí |
| `CANCELLED` | Cancelado | ❌ Excluir |
| `EXPIRED` | Expirado/vencido | ❌ Excluir |

**Filtros estándar por caso de uso**:
```sql
-- Solo tickets válidos (excluir cancelados y expirados)
WHERE status NOT IN ('CANCELLED', 'EXPIRED')

-- Solo tickets resueltos con resultado definido
WHERE status IN ('WON', 'PAIDOUT', 'LOST')

-- Apostado y TX del día (tickets válidos)
SELECT
  DATE(timePrint_pe)             AS fecha,
  COUNT(DISTINCT ticketId)       AS tickets,
  SUM(stake)                     AS apostado
FROM dlh_silver.golden_race.tm_d_tickets
WHERE status NOT IN ('CANCELLED', 'EXPIRED')
  AND DATE(timePrint_pe) = '2026-06-01'
GROUP BY DATE(timePrint_pe)

-- Detalle por cajero
SELECT
  sellStaff_name,
  COUNT(DISTINCT ticketId)       AS tickets,
  SUM(stake)                     AS apostado
FROM dlh_silver.golden_race.tm_d_tickets
WHERE status NOT IN ('CANCELLED', 'EXPIRED')
  AND DATE(timePrint_pe) = '2026-06-01'
GROUP BY sellStaff_name
ORDER BY apostado DESC
```

**Advertencias**:
- ⚠️ Montos (`stake`, `stakePromotional`, etc.) ya en **soles** — no dividir entre 100
- ⚠️ Filtrar siempre `status NOT IN ('CANCELLED', 'EXPIRED')` para análisis de negocio
- ⚠️ **Join con `tm_d_get_earning` pendiente confirmar**: `unit_extId` tiene valores en blanco, por lo que `unit_extId = entityExtId` puede perder registros — verificar cobertura antes de usar
- ⚠️ `unit_name` es texto libre — no usar para joins ni agrupaciones precisas; preferir `unit_extId`
- ⚠️ `timePrint_pe` es TIMESTAMP — usar `DATE()` al filtrar por día
- ⚠️ Esta tabla no tiene columna de monto ganado/pagado — para eso combinar con `tm_d_get_earning`

**Relación entre tablas GoldenRace**:
```
tm_d_tickets  ──(unit_extId = entityExtId ⚠️)──  tm_d_get_earning  ──(entityId)──  tm_d_jackpot_find
                                                        │
                                              (gameType = game_type)
                                                        │
                                                   vc_d_games
```

---

## GLOSARIO DE TÉRMINOS

| Término de negocio | Equivalente técnico | Tabla/Columna |
|--------------------|---------------------|---------------|
| "Cliente activo" | estado = 'ACTIVO' | esquema.clientes.estado |
| "Ingreso neto" | SUM(monto_neto) | esquema.transacciones.monto_neto |
| [término] | [definición técnica] | [tabla.columna] |

---

## REGLAS GENERALES DE CALIDAD

- ⚠️ **SIEMPRE** excluir `company = 'TEST'` en cualquier tabla que tenga columna `company`. Sin este filtro los números se inflan con datos de prueba.
- ⚠️ **SIEMPRE** usar `summary_date_pe` en lugar de `summary_date` cuando la tabla tenga ambas columnas. `summary_date` está en horario de España; `summary_date_pe` está en horario de Perú y es la fecha correcta para análisis locales.
- ⚠️ `summary_date_pe` tiene formato TIMESTAMP (`2026-05-17 19:00:00.000`), no DATE. Al filtrar por fecha usar `DATE(summary_date_pe) = '2026-05-17'` o `CAST(summary_date_pe AS DATE)`. Nunca comparar directamente con un string de fecha o el filtro fallará.
- La columna `db` (con valores 1 al 5) existe en varias tablas y se usa como clave de cruce junto con `user` o `machine`. No ignorarla en los joins — ver sección de Reglas de Negocio.
- La columna `account` cuando existe indica el tipo de dinero. Valores conocidos: `'CASH'` = dinero real Digital, `'CASH-RETAIL'` = dinero real Retail, `'CASINO-BONUS'` = bono de casino. Filtrar siempre según lo que se quiera analizar. Nunca ignorar esta columna sin decidir explícitamente qué se quiere ver. ⚠️ `'CASH'` y `'CASH-RETAIL'` son valores distintos — para ver todos los canales usar `account IN ('CASH', 'CASH-RETAIL')`.
- Nunca hacer `SELECT *` en tablas con muchas columnas, siempre seleccionar columnas explícitas.

---

## ARGUMENTO RECIBIDO

El usuario consulta: $ARGUMENTS

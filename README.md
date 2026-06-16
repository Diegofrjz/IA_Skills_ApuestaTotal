# IA Skills — Apuesta Total
Skills para Queries y Business Analytics para AT

---

## Skill: `/datos` — Asistente de Datos Databricks

### Estructura

```
SkillDatabricks/
├── .claude/
│   └── commands/
│       └── datos.md   ← toda la skill aquí (catálogo + reglas + indicadores)
├── CONTEXTO_AGENTE.md ← progreso, decisiones y pendientes del proyecto
└── README.md
```

### Cómo instalar

Copia la carpeta `.claude/` a la raíz del proyecto donde quieras usar la skill.

### Cómo usar

```
/datos ¿cuántos clientes activos hay este mes?
/datos dame el ingreso neto por canal del último trimestre
/datos necesito un query de nuevos registros por semana
/datos query de KPIs diarios retail junio 2026
```

### Cómo mantener

Todo el contenido vive en `.claude/commands/datos.md`. Para agregar tablas, reglas o indicadores, editar ese único archivo directamente.

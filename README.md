# Skill Databricks — Asistente de Datos

## Estructura

```
SkillDatabricks/
├── .claude/
│   └── commands/
│       └── datos.md   ← toda la skill aquí (catálogo + reglas + indicadores)
└── README.md
```

## Cómo instalar

Copia la carpeta `.claude/` a la raíz del proyecto donde quieras usar la skill.

## Cómo usar

```
/datos ¿cuántos clientes activos hay este mes?
/datos dame el ingreso neto por canal del último trimestre
/datos necesito un query de nuevos registros por semana
```

## Cómo mantener

Todo el contenido vive en `.claude/commands/datos.md`. Para agregar tablas, reglas o indicadores, editar ese único archivo directamente.

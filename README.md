# 📊 Análisis de Mercado Energético — Colombia

Reporte de Business Intelligence desarrollado en **Power BI** para el análisis del mercado eléctrico colombiano. Cubre demanda energética por departamento y tipo de mercado (Regulado / No Regulado), precios por Operador Regional, y análisis estratégico de oportunidades comerciales.

---

## 🗂️ Estructura del repositorio
```
├── Analisis de Mercado Energetico OK.pbip         # Archivo principal Power BI (formato .pbip)
├── Analisis de Mercado Energetico OK.Report/      # Páginas, visuales y configuración del reporte
├── Analisis de Mercado Energetico OK.SemanticModel/ # Modelo semántico (tablas, medidas, relaciones)
└── .gitignore
```

---

## 🏗️ Modelo de datos — Star Schema
```
                    ┌─────────────────┐
                    │  Dim_Calendar   │
                    │  (calculada DAX)│
                    └────────┬────────┘
                             │
          ┌──────────────────┴──────────────────┐
          │                                     │
 ┌────────┴────────┐                  ┌─────────┴────────┐
 │  Fact_Demanda   │                  │   Fact_Precios   │
 │  (Power Query)  │                  │  (Power Query)   │
 └────────┬────────┘                  └─────────┬────────┘
          │                                     │
          └──────────────┬──────────────────────┘
                         │
                ┌────────┴────────┐
                │ Dim_Departamento│
                │  (tabla manual) │
                └─────────────────┘
```

| Tabla | Tipo | Descripción |
|---|---|---|
| `Fact_Demanda` | Tabla de hechos | Demanda en GWh por departamento, fecha y tipo de mercado. Combina mercado Regulado y No Regulado |
| `Fact_Precios` | Tabla de hechos | Precios en COP/kWh por Operador Regional y fecha |
| `Dim_Calendar` | Dimensión tiempo | Tabla calendario calculada en DAX con granularidad hasta día, semana, mes, trimestre y semestre |
| `Dim_Departamento` | Dimensión geográfica | Departamentos con capital, región, latitud y longitud para mapas |
| `_Medidas_2` | Tabla de medidas | Tabla oculta contenedora de todas las medidas DAX del modelo (buena práctica) |

---

## ⚙️ Transformaciones Power Query (M)

### Grupo: Demanda

**NON REGULATED** — fuente: `Prueba_demanda_departamento.xlsx`
- Skip de 4 filas de encabezado
- Filtro de filas con valor "NO REGULADO"
- Eliminación de columnas innecesarias
- Unpivot de columnas de departamentos a formato largo
- Renombrado de columnas → `Fecha`, `Departamento`, `Demanda_GWh`
- Columna personalizada `Tipo_Mercado = "No Regulado"`
- Cambio de tipos: fecha → `date`, demanda → `number`

**REGULATED** — misma fuente, hoja REGULATED
- Proceso idéntico con filtro de "Grand Total" y "REGULADO"
- `Tipo_Mercado = "Regulado"`

**Fact_Demanda** — combina ambas tablas con `Table.Combine`

### Grupo: Precios

**Fact_Precios** — fuente: `Prueba_Target Market Pricing_BI.xlsx`
- Unpivot de operadores regionales a formato largo
- Renombrado → `Fecha`, `Operador_Regional`, `Precio_COP_kWh`
- Estandarización de nombres de OR: `atlantico → ATLÁNTICO`, `bogota_cundinamarca → BOGOTÁ. D.C.`, `antioquia → ANTIOQUIA`, `bolivar → BOLÍVAR`, `cali → VALLE DEL CAUCA`

---

## 📐 Medidas DAX — Tabla `_Medidas_2`

Las medidas están organizadas en carpetas de visualización (display folders):

### 📁 01. KPIs Ejecutivos
| Medida | Fórmula base | Formato |
|---|---|---|
| `Total Demanda GWh` | `SUM(Fact_Demanda[Demanda_GWh])` | `#,##0.0` |
| `Total Demanda No Regulado` | `CALCULATE(SUM(...), Tipo_Mercado = "No Regulado")` | `#,##0.0` |
| `Total Demanda Regulado` | `CALCULATE(SUM(...), Tipo_Mercado = "Regulado")` | `#,##0.0` |
| `% No Regulado` | `DIVIDE([Total Demanda No Regulado], [Total Demanda GWh])` | `0.00 %` |
| `% Regulado` | `DIVIDE([Total Demanda Regulado], [Total Demanda GWh])` | `0.0%` |
| `Conteo Departamentos` | `DISTINCTCOUNT(Fact_Demanda[Departamento])` | `#,##0` |

### 📁 02. Análisis de Demanda
| Medida | Descripción |
|---|---|
| `Promedio Diario Demanda GWh` | `AVERAGEX` sobre fechas únicas |
| `Promedio Diario NR` / `Promedio Diario RG` | Promedios diarios segmentados por tipo de mercado |
| `Demanda Máxima Mensual` | `MAXX(SUMMARIZE(...))` |
| `Demanda Mínima Mensual` | `MINX(SUMMARIZE(...))` |
| `Variación Domingo vs Semana` | Caída % entre domingos y días hábiles usando `WEEKDAY` |
| `Participación Departamento` | `DIVIDE([Total Demanda GWh], CALCULATE(..., ALL(Departamento)))` |
| `Ranking Departamento Demanda` | `RANKX(ALL(Departamento), [Total Demanda GWh],, DESC, Dense)` |

### 📁 03. Análisis de Precios
| Medida | Descripción |
|---|---|
| `Precio Promedio COP/kWh` | `AVERAGE(Fact_Precios[Precio_COP_kWh])` |
| `Precio Máximo` / `Precio Mínimo` | `MAX` / `MIN` sobre precios |
| `Rango de Precio` | `[Precio Máximo] - [Precio Mínimo]` — indicador de volatilidad |
| `Precio Max 2024` / `Precio Min 2024` | Filtrados con `Dim_Calendar[Year] = 2024` |
| `Precio Promedio PY` | `SAMEPERIODLASTYEAR` — comparativo año anterior |
| `Precio Promedio Bogotá 2024` / `Precio Promedio Antioquia 2024` | Precios específicos por OR |
| `Volatilidad Precio 2024` | Max − Min del año |
| `Ranking OR Precio` | `RANKX(ALL(Operador_Regional), [Precio Promedio],, DESC, Dense)` |
| `Diferencial vs Antioquia` | `% más caro que Antioquia (el más barato)` |
| `Variación Precio vs Inicio` | Variación desde agosto 2022 (inicio de serie) |
| `Diferencial Bogotá vs Antioquia $` | Diferencia absoluta COP/kWh |
| `Diferencial Bogotá vs Antioquia %` | Diferencia relativa `DIVIDE(_Bog - _Ant, _Ant)` |

### 📁 04. Estrategia y Oportunidades
| Medida | Descripción |
|---|---|
| `Top 5 Concentración %` | `TOPN(5)` para demanda concentrada en líderes |
| `Demanda Regulado Bogotá` | Demanda Regulada en Bogotá D.C. como potencial migratorio |
| `Potencial Migración Bogotá %` | % de demanda regulada sobre total en Bogotá |
| `Clasificación NR Departamento` | `SWITCH(TRUE())`: ★ Alta concentración / ● Mercado maduro / ◑ Oportunidad / ○ Predominio regulado |
| `Tier Mercado` | Clasificación estratégica: Tier 1 Prioritario / Tier 2 Crecimiento / Tier 3 Monitoreo |

### 📁 05. HTML Cards (visual HTML Viewer)
Medidas que generan HTML dinámico con CSS inline para tarjetas KPI y barras de insights:

| Medida | Descripción |
|---|---|
| `HTML Card Demanda Total` | Tarjeta ⚡ con demanda total y conteo de departamentos |
| `HTML Card No Regulado` | Tarjeta 🏭 con % No Regulado y GWh |
| `HTML Card Regulado` | Tarjeta 📊 con % Regulado y GWh |
| `HTML Card Bogotá Tarifa` | Tarjeta 💰 con precio COP/kWh de Bogotá |
| `HTML Card Líder Demanda` | Tarjeta 🏆 con departamento líder y su % de participación |
| `HTML Insights Bar` | Barra horizontal con 3 insights ejecutivos (fijos 2024) |
| `HTML Insights Demanda` | Barra horizontal con 3 insights de la página de Demanda |
| `HTML Estrategia Top` | Panel 2 columnas: Mercados Prioritarios + Recomendaciones Clave |
| `HTML Estrategia Bottom` | Barra de 3 insights finales de estrategia |

### 📁 06. Params HTML
Medidas de parámetro que controlan el estilo de las HTML Cards (permiten ajuste sin tocar el código):

| Medida | Valor | Descripción |
|---|---|---|
| `Param Titulo px` | 25 | Tamaño fuente título |
| `Param Valor px` | 36 | Tamaño fuente valor principal |
| `Param Subtitulo px` | 18 | Tamaño fuente subtítulo |
| `Param Emoji px` | 40 | Tamaño emoji decorativo |
| `Param Padding V` | 14 | Padding vertical contenedor |
| `Param Padding H` | 18 | Padding horizontal contenedor |

---

## 🔗 Relaciones del modelo

| Desde | Hacia | Cardinalidad |
|---|---|---|
| `Fact_Demanda[Fecha]` | `Dim_Calendar[Date]` | Muchos → 1 |
| `Fact_Precios[Fecha]` | `Dim_Calendar[Date]` | Muchos → 1 |
| `Fact_Demanda[Departamento]` | `Dim_Departamento[Departamento]` | Muchos → 1 |
| `Fact_Precios[Operador_Regional]` | `Dim_Departamento[Departamento]` | Muchos → 1 |

---

## 🔧 Fuentes de datos

| Archivo | Tipo | Uso |
|---|---|---|
| `Prueba_demanda_departamento.xlsx` | Excel | Hojas NON REGULATED y REGULATED → `Fact_Demanda` |
| `Prueba_Target Market Pricing_BI.xlsx` | Excel | Hoja "Query result" → `Fact_Precios` |

> ⚠️ Las rutas de origen son locales. Para reproducir el modelo debes actualizar las rutas de los archivos Excel en Power Query.

---

## 🚀 Control de versiones

Este proyecto está versionado con **Git + GitHub** e integrado con **Microsoft Fabric** mediante integración nativa de workspace.

Flujo de trabajo:
- `principal` → versión estable del reporte
- `Prueba_git` → rama de desarrollo y pruebas

---

## 🛠️ Herramientas utilizadas

- Power BI Desktop (formato `.pbip` — Power BI Projects)
- DAX (Data Analysis Expressions) — medidas, tabla calendario calculada, HTML dinámico
- Power Query (M) — ETL, unpivot, estandarización de datos
- Git + GitHub — control de versiones
- Microsoft Fabric — sincronización de workspace
- VS Code — gestión de commits y ramas

---

## 👤 Autor

**Rafael Milanes**
[LinkedIn]([https://www.linkedin.com/in/ingenierorafaelmilanes](https://www.linkedin.com/in/rafael-alberto-milanes-hernandez/)) | [GitHub](https://github.com/Rafael-Milanes)

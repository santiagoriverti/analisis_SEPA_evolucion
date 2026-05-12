# analisis_SEPA_canasta

Pipeline en Python para procesar datos históricos del programa SEPA (Sistema Electrónico de Publicidad de Precios Argentinos), construir una canasta fija de consumo de 30 productos y generar un indicador alternativo de inflación a nivel nacional, provincial y regional, comparado contra el IPC INDEC.

---

## 📌 Objetivo

Construir el **IPC-SEPA-Canasta UADE**: un indicador de inflación mensual de productos masivos de góndola, basado en los datos de relevamiento diario del SEPA. Permite anticipar dinámicas de precios respecto al IPC oficial, que se publica con dos semanas de rezago.

El proyecto:

- Procesa archivos históricos del SEPA en formato `.csv.gz` (~28 GB acumulados entre 2022 y 2026)
- Filtra una canasta fija de 30 productos mediante códigos EAN
- Construye series diarias y mensuales de precios por sucursal, provincia y región
- Genera una canasta nacional ponderada por población (Censo INDEC 2022)
- Compara la evolución contra el IPC INDEC (Nivel general y Alimentos)
- Exporta datasets listos para análisis económico y visualización

---

## 🏛️ Contexto institucional

Desarrollado por el **Instituto de Economía (INECO) de la UADE** sobre datos públicos del Ministerio de Economía de la Nación (programa SEPA, Resolución 12/2016) y el INDEC.

---

## ⚙️ Arquitectura del pipeline

El procesamiento se divide en dos etapas:

### Etapa 1 — Procesamiento por semestre

Un notebook parametrizable (`SEMESTRE = "2022A"`, `"2022B"`, ..., `"2026A"`) procesa cada paquete semestral del SEPA por separado. Genera para cada semestre dos archivos:

- `canasta_<SEMESTRE>_serie.xlsx` (resúmenes mensuales por provincia, región y nacional)
- `canasta_<SEMESTRE>_long.parquet` (detalle observación-a-observación enriquecido)

### Etapa 2 — Consolidación

Un segundo notebook une todos los semestres en una serie histórica continua, integra el IPC INDEC y exporta un archivo único de análisis: `canasta_SEPA_consolidado.xlsx`.

---

## ⚙️ Funcionalidades principales

### Procesamiento de datos

- Descompresión automática de zips semestrales (~1,2-1,7 GB cada uno)
- Lectura recursiva de archivos `.csv.gz` (cada uno con ~10-15 millones de filas)
- Validación de cobertura temporal (detección de días faltantes)
- Conversión de formato wide (1 columna por día) → long
- **Detección automática del factor de escala de precios** (algunos años usan pesos directos, otros formato con 2 decimales implícitos)
- Eliminación de duplicados por solapamiento de archivos rectificatorios
- Filtrado de precios placeholder (`> $5`) para descartar valores no genuinos

### Enriquecimiento

Cruce con dos maestros provistos por el Ministerio:

- **Maestro de productos** (176.702 registros): marca, rubro, categoría, subcategoría, proveedor
- **Maestro de sucursales** (3.611 registros): provincia, región (AMBA / Pampeana / NOA / NEA / Cuyo / Patagonia), localidad, tipo de sucursal

### Normalización

- **Provincias**: diccionario que armoniza variantes históricas como "Provincia de Buenos Aires" / "Buenos Aires", "Ciudad Autónoma de Buenos Aires" / "CABA", correcciones de typos ("San juan" / "San Juan").
- **EANs**: lstrip de ceros para matching robusto entre datasets.

### Construcción de indicadores

#### Serie diaria nacional

Para cada producto y día: promedio, mediana, mínimo, máximo, cantidad de observaciones.

#### Canasta mensual por provincia

Valor mensual agregado utilizando cantidades fijas por producto (ver canasta abajo).

#### Canasta mensual por región

Agregación regional de precios (6 regiones).

#### Canasta nacional ponderada

Índice nacional construido como suma ponderada de las 24 canastas provinciales, con pesos del Censo INDEC 2022 (45.892.285 habitantes).

#### Comparativa SEPA vs IPC

Variaciones mensuales, índices base 100 y brechas acumuladas vs el IPC INDEC Nivel general y Alimentos.

---

## 🧺 Canasta utilizada

30 productos de consumo masivo seleccionados por:
- EAN reconocido (GTIN-13 o equivalente)
- Disponibilidad en 24 provincias
- Reporte por al menos 15 cadenas durante el período analizado

Distribución por rubro:

| Rubro | Productos |
|---|---|
| Lácteos | 5 (leche entera, yogur, queso, manteca, leche chocolatada) |
| Almacén | 8 (aceite, arroz, fideos, harina, yerba, café, galletitas, sal) |
| Bebidas | 5 (coca cola lata, coca sin azúcar, agua saborizada, cerveza, vino) |
| Limpieza | 3 (lavandina, detergente, limpiador) |
| Higiene | 7 (shampoo, acondicionador, jabón, antitranspirante, hilo dental, toallas femeninas, papel higiénico) |
| Snacks | 2 (chocolates, galletitas saladas) |

Cada producto está definido por: EAN, descripción, **cantidad mensual fija** (calibrada para una familia tipo de 4 personas), categoría y proveedor.

---

## 📂 Inputs requeridos

### Para el procesamiento por semestre

```text
/content/
├── 2022A.zip ... 2026A.zip          (paquetes SEPA del Ministerio)
├── Maestro de Productos Interno.xlsx
└── maestro_sucursales_completo.xlsx
```

### Para la consolidación

```text
/content/
├── canasta_2022A_serie.xlsx ... canasta_2026A_serie.xlsx
└── IPC.xlsx                          (IPC INDEC por divisiones, 2017 en adelante)
```

---

## 📤 Outputs generados

### Por semestre

- `canasta_<SEMESTRE>_serie.xlsx` con 7 hojas:
  - `cobertura_temporal`
  - `canasta_definicion`
  - `serie_diaria_nacional`
  - `canasta_mes_provincia`
  - `canasta_mes_region`
  - `canasta_nacional_ponderada`
  - `pesos_poblacionales`
- `canasta_<SEMESTRE>_long.parquet` (detalle observación-a-observación, ~15-55 MB por semestre)

### Consolidado

- `canasta_SEPA_consolidado.xlsx` con 7 hojas:
  - `nacional_valida` — serie nacional mensual (36 meses, mayo 2023 - abril 2026)
  - `nacional_completa` — incluye meses descartados para trazabilidad
  - `por_provincia` — 864 filas (36 meses × 24 provincias)
  - `por_region` — 216 filas (36 meses × 6 regiones)
  - `ipc_indec` — IPC INDEC procesado por divisiones
  - `comparativa_sepa_ipc` — tabla comparativa con índices base 100, variaciones mensuales, brechas
  - `metodologia` — documentación

### Visualizaciones

- `grafico_canasta_vs_ipc.png` — serie completa desde mayo 2023
- `grafico_canasta_vs_ipc_desde_jun24.png` — vista reciente para análisis post-shock

---

## ⚠️ Limitaciones metodológicas

### Cobertura SEPA pre-mayo 2023

El programa SEPA tuvo cobertura **reducida y heterogénea** entre enero y abril de 2023: productos clave de la canasta (leche larga vida, fideos, manteca) no aparecían en el dataset o eran reportados por solo una cadena con muy pocas observaciones. La canasta para esos meses incluye solo ~10 de los 30 productos, lo que distorsiona tanto el nivel como las variaciones mensuales.

**Decisión metodológica**: el análisis arranca en **mayo de 2023**, cuando el panel de cadenas reportantes se estabilizó y la canasta de 30 productos quedó completa.

### Variaciones mensuales

- **Mayo 2023**: primer mes del análisis, no tiene variación mensual calculable.
- **Junio 2023**: el panel todavía estaba en proceso de estabilización, por lo que su variación vs mayo no es comparable.
- **Primer mes con variación válida**: **julio 2023** vs junio 2023.

### Comparabilidad con IPC

La canasta SEPA y el IPC INDEC no son directamente comparables como indicadores:

- **IPC INDEC**: cubre 12 divisiones (~27% Alimentos, 13% Transporte, 12% Vivienda, 9% Salud, etc.) construido sobre ENGHo.
- **Canasta SEPA-UADE**: 30 productos específicos de consumo masivo en supermercados. Cubre aproximadamente el "núcleo de góndola" de las divisiones Alimentos + Equipamiento del hogar + Cuidado personal.

Las **diferencias entre ambos indicadores son informativas**: revelan cómo se mueven los productos de marca en cadenas grandes vs el promedio del país, donde el IPC se modera por servicios regulados, salud, educación, etc.

---

## 📊 Hallazgos principales

Período analizado: **mayo 2023 - abril 2026** (36 meses)

| Indicador | Valor inicial | Valor final | Inflación acumulada |
|---|---|---|---|
| Canasta SEPA-UADE | $30.845 | $322.566 | **+946%** |
| IPC INDEC general | 100 (índice) | 686,49 (índice) | +586% |
| IPC INDEC alimentos | 100 (índice) | 676,75 (índice) | +577% |

**La canasta SEPA acumuló aproximadamente 1,5 veces más inflación que el IPC general** desde mayo 2023, reflejando que los productos masivos de góndola se ajustaron más que el promedio del país.

### Períodos de divergencia

- **Diciembre 2023 - febrero 2024**: la canasta capturó el shock devaluatorio post-asunción más fuerte que el IPC general (+46,87% en dic-23 vs IPC +25,47%).
- **2025A**: la canasta se desinfló antes que el IPC general (precios de góndola frenados por presión sobre productores líderes).
- **2025B y 2026A**: convergencia gradual, con tu canasta levemente por debajo del IPC alimentos.

---

## 🛠️ Stack técnico

- **Python 3** (Google Colab)
- **pandas** + **numpy** para procesamiento
- **openpyxl** para Excel
- **pyarrow** para Parquet (formato comprimido eficiente)
- **matplotlib** para visualizaciones
- ~6-10 minutos de ejecución por semestre en Colab gratuito

---

## 📚 Fuentes

- **SEPA**: datos.produccion.gob.ar/dataset/sepa-precios — Ministerio de Economía de la Nación
- **IPC INDEC**: indec.gob.ar — Instituto Nacional de Estadística y Censos
- **Censo 2022**: INDEC, datos poblacionales por jurisdicción
- **Marco normativo**: Resolución 12/2016 de la ex Secretaría de Comercio

---

## 👥 Autores

INECO — Instituto de Economía, UADE

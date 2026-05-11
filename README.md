# analisis_SEPA_canasta

Pipeline en Python para procesar datos históricos del programa SEPA (Sistema Electrónico de Publicidad de Precios Argentinos), construir una canasta fija de consumo y generar indicadores de precios nacionales, provinciales y regionales.

---

# 📌 Objetivo

El proyecto busca:

- Procesar archivos históricos del SEPA en formato `.csv.gz`
- Filtrar una canasta fija de productos mediante códigos EAN
- Construir series diarias y mensuales de precios
- Analizar diferencias regionales y provinciales
- Generar una canasta nacional ponderada por población (Censo 2022)
- Exportar datasets listos para análisis económico y visualización

---

# ⚙️ Funcionalidades principales

## Procesamiento de datos

- Descompresión automática de zips semestrales
- Lectura recursiva de archivos `.csv.gz`
- Validación de cobertura temporal
- Conversión de formato wide → long
- Eliminación de duplicados por solapamiento de archivos

## Enriquecimiento

Cruce con:

- Maestro de productos
- Maestro de sucursales

Agregando:

- marca
- rubro
- categoría
- proveedor
- provincia
- región
- localidad
- tipo de sucursal

## Construcción de indicadores

### Serie diaria nacional

Para cada producto:

- promedio
- mediana
- mínimo
- máximo
- cantidad de observaciones

### Canasta mensual por provincia

Valor mensual agregado utilizando cantidades fijas por producto.

### Canasta mensual por región

Agregación regional de precios.

### Canasta nacional ponderada

Índice nacional construido utilizando pesos poblacionales del Censo 2022.

---

# 🧺 Canasta utilizada

La canasta incluye productos de:

- Lácteos
- Almacén
- Bebidas
- Limpieza
- Higiene
- Snacks

Ejemplos:

- leche
- yogur
- aceite
- arroz
- yerba
- gaseosas
- cerveza
- detergente
- shampoo
- papel higiénico

Cada producto posee:

- EAN
- descripción
- cantidad fija
- categoría

---

# 📂 Inputs requeridos

El notebook espera encontrar:

```text
/content/
│
├── 2022A.zip
├── 2022B.zip
├── 2023A.zip
├── ...
│
├── Maestro de Productos Interno.xlsx
├── maestro_sucursales_completo.xlsx

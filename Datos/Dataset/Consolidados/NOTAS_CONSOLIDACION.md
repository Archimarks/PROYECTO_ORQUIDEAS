# Notas de consolidación — Ilex guayusa (reconstrucción con esquema normalizado)

Fuentes:
- Artículo: `Datos/Artículos/01_ilex_guayusa.pdf` (Garzón et al. 2026, *J. Chromatogr. A* 1769:466732) → `00_articulos.csv` (`ID_ARTICULO=1`)
- Batches: `Datos/Dataset/Batch/BATCH_01_NEG_ilex_guayusa.xlsx` (hoja `Sheet 1`) y `BATCH_02_POS_ilex_guayusa.xlsx` (hojas `Pos_1`, `Pos_2`, `Hoja1`) → `01_batches.csv` (`ID_BATCH=1,2`)
- Script reproducible: `build_consolidado.py` (vuelve a correr si algún Excel cambia)

Resultado: **201 muestras/corridas** (100 NEG + 101 POS) sobre **101 extractos
físicos únicos** (100 compartidos entre NEG/POS + 1 blanco de proceso solo en
POS), y **1,284,419 filas** en `06_picos.csv`, combinando **4 pipelines de
procesamiento**:

| Pipeline (`ID_ORIGEN_PROCESAMIENTO`) | features | muestras | filas |
|---|---|---|---|
| 1 — BATCH1_NEG | 636 | 100 | 63,600 |
| 2 — BATCH2_Pos_1 | 509 | 101 | 51,409 |
| 3 — BATCH2_Pos_2 | 10,910 | 101 | 1,101,910 |
| 4 — BATCH2_Hoja1 | 675 | 100 | 67,500 |

Distribución de `ID_TIPO_MUESTRA`:

| Batch | Sample | SubQC | QC | Blank |
|---|---|---|---|---|
| NEG | 54 | 32 | 14 | — |
| POS | 54 | 32 | 14 | 1 |

Integridad referencial validada por script sobre las 1,284,419 filas de
`06_picos.csv` y el resto de tablas — **0 llaves huérfanas**.

---

## 1. Estructura del Excel (formato "notame")

Cada hoja trae exactamente la separación de niveles del esquema: filas 1-8/10
= metadata por muestra (una columna por muestra: `Location_Factor`,
`Light_Factor`, `Age_Factor`, `Analysis_group`, `Visit`, `Injection_order`...),
fila de encabezado de features (`row_ID/row ID`, `row_m_z`, `row_retention_time`,
`Metabolite`, `Identification_level`, `Class`, `Superclass`...), y filas
siguientes = un feature por fila.

`Pos_1` es la excepción: no trae columna `Identification_level` y sus 3
metabolitos nombrados (`Caffeine`, `Acetyl tributyl citrate`, `Valerophenone`)
no traen `Class`/`Superclass` tampoco. Se completó cruzando con las otras 3
hojas (que sí identifican esos mismos compuestos con clasificación completa)
y, para el nivel de identificación de **los 16 metabolitos nombrados en todo
el dataset**, se usó como fuente única la Tabla 3 del artículo (`01_ilex_guayusa.pdf`)
en vez del Excel — el Excel es inconsistente entre hojas (a veces vacío, a
veces I/II contradictorio para el mismo compuesto) y la tabla del artículo es
la fuente publicada y revisada.

Las hojas `Export_mzmine_fullscan_pos_1` y `Export_mzmine_fullscan_pos_2` del
batch POS **no se usaron** — son el export crudo de MZmine antes de fusionarse
con los datos MS/MS (`Pos_1`/`Pos_2` son ya la versión fusionada y anotada),
así que incluirlas hubiera duplicado los mismos features sin aportar nada
nuevo.

## 2. Cómo se armaron los identificadores

El encabezado real de cada columna de muestra trae el nombre de archivo
original, ej. `100_MB_53_B-2_3_POS_THOM01.CDF Peak height` (POS) o
`100_MB_53_B-2_3_NEG_THOM101.CDF Peak height` (NEG). Se quitó el sufijo fijo
(`_THOM01.CDF Peak height` / `_THOM101.CDF Peak height`) para obtener:

- **`CODIGO_MUESTRA`** (nivel corrida): `100_MB_53_B-2_3_POS`
- **`CODIGO_MUESTRA_FISICA`** (nivel extracto, quitando el `_POS`/`_NEG`): `100_MB_53_B-2_3`

`ID_MUESTRA` e `ID_MUESTRA_FISICA` son los enteros autoincrementales que
exige el esquema (ver `Datos/Base/README_ESQUEMA.md`, regla de IDs); los
códigos de laboratorio se conservan en las columnas `CODIGO_*` para
trazabilidad. Se verificó explícitamente que los 100 `CODIGO_MUESTRA_FISICA`
del batch NEG están **todos** presentes en el batch POS (mismo extracto, otra
corrida) — la única diferencia es `8_PB` (blanco de proceso), que solo existe
en POS.

Patrón observado en el nombre (no confirmado explícitamente en el artículo):
`{número}_{MB=muestra individual | GC=pool agrupado | POOLED_QC=pool total |
PB=blanco de proceso}_{código}`. No se intentó descomponer este texto en
columnas nuevas porque esa misma información ya viene limpia en
`Location_Factor`/`Age_Factor`/`Light_Factor`/`Analysis_group` del Excel —
se usaron esas columnas directamente.

## 3. Columnas nuevas agregadas a `03_muestras.csv` (no están en el molde de Base)

Aprobado implícitamente por el mismo criterio ya usado en la sesión (extender
el molde cuando el dato real lo exige, documentándolo aquí y en el molde si
aplica a cualquier dataset futuro):

- **`VISITA`**: viene de `Visit` del Excel — número de visita/ronda de campo.
- **`SECUENCIA_INYECCION`**: viene de `Injection_order` del Excel — el orden
  real en que se inyectó cada muestra en el equipo. Útil para diagnosticar
  deriva/efecto lote dentro de un mismo `LOTE`.

`ID_NOTAME` (mencionado en una consolidación anterior de este mismo dataset)
**no se agregó como columna aparte** — el número inicial de `CODIGO_MUESTRA`
(ej. el `100` en `100_MB_53_B-2_3`) ya es ese mismo identificador (`ID_100`
en la fila `Sample_ID` del Excel NEG), así que hubiera sido una columna
redundante.

Para las filas `QC`/`SubQC` que agrupan más de una ubicación (`Location_Factor`
= `QC` o `Other enviroment factors`), `ID_UBICACION` apunta a la fila 4 de
`Catalogos_Muestras/ubicaciones.csv` ("Pool multi-chakra"). Para `Blank`,
`ID_ESPECIE`/`ID_TIPO_CULTIVO`/`ID_PARTE_PLANTA`/`ID_UBICACION` quedan vacíos
(no es material vegetal), siguiendo la regla ya documentada en el molde.

## 4. `04_muestra_factor.csv`

Solo se registran filas de `Light_Factor`/`Age_Factor` cuando el valor es
específico (`Shade`/`Light`, `Early`/`Medium`/`Late` — con el typo `Eary` del
Excel normalizado a `Early`). Se omiten para `Blank`, para `QC` (pool total,
mezcla todo) y para `SubQC` agrupado por un factor que no sea ese (donde el
valor es `QC`/`Other enviroment factors`/`Blank`, no un valor real medible).

## 5. Solvente de extracción como mezcla, a nivel de BATCH (no de muestra)

`SOLVENTE_EXTRACCION` empezó como un solo texto en `04_muestras.csv`
("Agua con 2% ácido fórmico y 10% acetonitrilo"). Se descompuso en:

- `Catalogos_Muestras/solventes_extraccion.csv` — catálogo de **componentes
  puros** (`Agua`, `Acido Formico`, `Acetonitrilo`), sin porcentaje.
- `02_batch_solvente.csv` — tabla pivote `ID_BATCH, ID_SOLVENTE, PORCENTAJE`.

Primero se armó por `ID_MUESTRA` (603 filas, 201 muestras × 3 componentes),
pero el mismo protocolo de extracción se usó para **todo el estudio** — la
sección 2.4.1 del artículo dice explícitamente que la preparación LC-MS
"followed the HPLC sample preparation protocol (Section 2.2.1)", el mismo
diluyente para NEG y POS. Repetirlo por cada una de las 201 muestras era
redundante: la granularidad real es el **batch** (`01_batches.csv`, que ya
existía), así que se re-hizo como `ID_BATCH, ID_SOLVENTE, PORCENTAJE` — 6
filas (2 batches × 3 componentes) en vez de 603, sin perder la capacidad de
que un batch futuro use una mezcla distinta. El porcentaje de agua (88%) se
calculó como el resto hasta 100% — el artículo solo da explícito "2.0%
formic acid and 10% acetonitrile".

Nota: `Datos/Base/` (el molde genérico) mantiene el patrón por-muestra
(`ID_MUESTRA, ID_SOLVENTE, PORCENTAJE`) como default razonable, porque no
todo dataset futuro va a tener el concepto de "batch" expuesto como aquí
(`00_articulos.csv`/`01_batches.csv` son específicos de este dataset, no
forman parte del molde). Si un dataset futuro sí organiza sus datos por
batch, puede replicar este mismo ajuste.

## 6. Actividad biológica

`05_especie_actividad.csv` quedó **vacío** — esta reconstrucción solo usó el
artículo `01_ilex_guayusa.pdf` (que no reporta actividad biológica propia) y
los 2 batches de MS. Los 14 valores de actividad biológica investigados en
una sesión anterior (Wikipedia, revisión PMC12251278, Gamboa et al. 2018,
Kapp et al. 2016) **no se restauraron** porque esta vez no se dieron como
fuente — si se quieren de vuelta, hay que pedirlo explícitamente y se
reconstruyen con el esquema `ID_ACTIVIDAD`/`ID_OBJETIVO`/`ID_METRICA`/
`ID_UNIDAD`/`ID_CONDICION_ENSAYO`/`ID_REFERENCIA` ya definido en el molde.

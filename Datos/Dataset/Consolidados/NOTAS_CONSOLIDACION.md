# Notas de consolidación — Ilex guayusa, Batch 1 (NEG) + Batch 2 (POS)

Fuentes usadas:
- Artículo: `Datos\Artículos\1- Ilex guayusa.pdf` (Garzón et al. 2026, *J. Chromatogr. A* 1769:466732)
- Datos crudos: `Datos\Dataset\Batch\BATCH 1- MZmine_to_R_notame_neg.xlsx` (hoja única) y
  `BATCH 2- MZmine_to_R_notame_pos.xlsx` (hoja `Pos_2`, ver sección 8)
- Script de generación: `build_ilex_guayusa_batches.py` (reproducible — vuelve a correr si algún Excel cambia)

Resultado: **201 muestras/corridas** (100 NEG + 101 POS, ver conteo por tipo abajo)
sobre **54 extractos físicos únicos** (`ID_MUESTRA_FISICA`), y **1,284,419 filas**
en `03_picos.csv`, combinando **4 pipelines de procesamiento**:

| ORIGEN_PROCESAMIENTO | muestras | features | filas |
|---|---|---|---|
| BATCH1_NEG | 100 | 636 | 63,600 |
| BATCH2_Pos_2 | 101 | 10,910 | 1,101,910 |
| BATCH2_Pos_1 | 101 | 509 | 51,409 |
| BATCH2_Hoja1 | 100 | 675 | 67,500 |

**Todo `ID_MUESTRA` de `03_picos.csv` hace join limpio contra `01_muestras.csv`
(0 huérfanos, verificado)** — desde ahí siempre puedes recuperar `LOTE`,
`TIPO_MUESTRA`, `MODO_IONIZACION` y toda la metadata botánica con un simple
merge por `ID_MUESTRA`. Las 3 hojas del batch POS comparten el mismo
`ID_MUESTRA` para el mismo extracto (no se duplicó `01_muestras.csv` por
cada hoja) — solo se agregó la columna `ORIGEN_PROCESAMIENTO` en
`03_picos.csv` para distinguir de qué pipeline viene cada pico, porque los
`row_ID`/`ID_FEATURE` **no son comparables entre hojas** (son numeraciones
independientes de MZmine, no el mismo feature renumerado).

| MODO_IONIZACION | TIPO_MUESTRA | n |
|---|---|---|
| NEG | Sample | 54 |
| NEG | SubQC | 32 |
| NEG | QC | 14 |
| POS | Sample | 54 |
| POS | SubQC | 32 |
| POS | QC | 14 |
| POS | Blank | 1 |

`03_picos.csv` pesa **~76 MB** — ya no es un archivo "chico", ábrelo con
pandas/script, no con Excel directo si puedes evitarlo (Excel no soporta
más de ~1.048.576 filas por hoja de todas formas y esto ya está cerca).

---

## 1. Estructura del Excel (formato "notame")

El archivo ya viene con exactamente la separación de niveles que definimos en el
esquema: filas 1-9 = metadata por muestra (una columna por muestra), fila 10 =
encabezado de features, filas 11+ = un feature por fila con `row_ID, m/z, RT,
Metabolite, Identification_level, Class, Superclass` ya anotados para los 11
compuestos identificados. No fue necesario inventar clasificación química —
ya estaba en el archivo.

## 2. Cómo se armaron los identificadores

El encabezado real de cada columna de muestra (fila 10) trae el nombre de
archivo original, ej. `100_MB_53_B-2_3_NEG_THOM101.CDF Peak height`. Se le
quitó el sufijo fijo `_THOM101.CDF Peak height` (idéntico en las 100
columnas) para obtener:

- **`ID_MUESTRA`** (nivel corrida): `100_MB_53_B-2_3_NEG`
- **`ID_MUESTRA_FISICA`** (nivel extracto, quitando el `_NEG`): `100_MB_53_B-2_3`
- **`ID_NOTAME`**: el ID simplificado que usa el propio análisis en R del
  artículo (`ID_100`) — se conserva como columna aparte por trazabilidad con
  el repositorio de GitHub que citan (IKIAM-NPLab/I_guayusa_caffeoylquinic_acid_isomers).

Patrón observado en el nombre (no confirmado explícitamente en el artículo,
es una inferencia a partir de cruzar el nombre con las columnas
`Location_Factor/Age_Factor/Light_Factor` — revísalo si tienes la
documentación interna del laboratorio):
`{número}_{MB=muestra individual | GC=pool agrupado | POOLED_QC=pool total}_{código}_{Chakra}{signo luz}{edad}_{réplica}`.
No se intentó descomponer este texto en columnas nuevas porque esa misma
información ya viene limpia en `Location_Factor`, `Age_Factor` y
`Light_Factor` — se usaron esas columnas directamente, no el nombre.

`ID_MUESTRA` siempre termina en `_NEG` o `_POS` según el batch de origen.
Se verificó explícitamente que los 100 `ID_MUESTRA_FISICA` del batch NEG
están **todos** presentes en el batch POS (mismo extracto, otra corrida) —
la única diferencia es `8_PB` (blanco de proceso), que solo existe en POS
(ver sección 8).

## 3. Columnas nuevas agregadas a 01_muestras.csv (no están en el molde de Base)

Aprobado por ti explícitamente — el molde de `Datos\Base` no se tocó:

- **`EDAD_PLANTA`**: viene de `Age_Factor` (Early/Medium/Late), con el rango
  de años del artículo (Sección 2.1: Early 4-6 años, Medium 6-8, Late 8-10).
- **`EXPOSICION_LUZ`**: viene de `Light_Factor` (Light/Shade), con el umbral
  de PPFD del artículo (Light ≥300 PPFD, Shade 0-200 PPFD).
- **`SECUENCIA_INYECCION`**: viene de `Injection_order` del Excel — el orden
  real en que se inyectó cada muestra en el equipo. Útil para diagnosticar
  deriva/efecto lote dentro de este mismo `LOTE`, que va complementado con
  el `LOTE` propiamente dicho.
- **`ID_NOTAME`**: ver punto 2.

Para las filas `QC` (pool total, `Location_Factor="QC"`, etc.) y las filas
`SubQC` agrupadas por un factor distinto (ej. un SubQC agrupado por edad
tiene `Location_Factor="Other enviroment factors"`), se dejó
`ORIGEN_GEOGRAFICO` / `EDAD_PLANTA` / `EXPOSICION_LUZ` **vacíos** cuando ese
factor específico no aplica a esa fila — el propio Excel ya distingue esto
por columna, no hubo que inferirlo.

Columna del Excel que **no** se incluyó por significado ambiguo: `Visit`
(valores 1-14, no explicado en el artículo ni evidente por contexto). Queda
disponible en el Excel crudo si luego decides qué representa.

## 4. Metadata fija tomada del artículo (aplica a las 100 filas)

Taxonomía completa: solo el artículo confirma directamente Género (*Ilex*),
Especie (*guayusa*) y por referencia bibliográfica [11] la Familia
(Aquifoliaceae). **Filo, Clase y Orden se completaron con taxonomía botánica
estándar** (Tracheophyta / Magnoliopsida / Aquifoliales), no vienen
explícitos en el paper — verifícalos si necesitas precisión taxonómica
estricta.

Método de extracción, solvente, columna cromatográfica y modo de ionización
salen de la Sección 2.4 (perfil metabolómico LC-MS, no la sección 2.2 de
HPLC-UV que usa un método/columna distintos para la cuantificación dirigida
de 5-CQA).

## 5. 02_muestra_actividad.csv — importante

Este artículo **no mide bioactividad** (es un paper de metodología
analítica/metabolómica). Solo se agregó, para las 54 filas `Sample` (no para
QC/SubQC), la propiedad antioxidante/antiinflamatoria **conocida por
literatura general** citada en la introducción del artículo, con
`VALOR_RESULTADO` vacío.

La tabla ya no tiene columna `REFERENCIA_FUENTE` (se quitó del molde a
pedido tuyo), así que la procedencia de este dato queda documentada **solo
aquí**: viene de Garzón et al. 2026, *J. Chromatogr. A* 1769:466732, citando
sus refs. 5,6,8 (antioxidante) y 7 (antiinflamatorio) — **no fue medido en
este estudio**, es conocimiento previo de la especie. Tenlo presente si en
el futuro agregas un ensayo real medido sobre estos extractos: sin esa
columna, ya no hay forma automática en el CSV de distinguir "medido" de
"literatura" — solo esta nota.

**Esto es exactamente el caso de riesgo de fuga que hablamos antes**: este
valor es constante para las 54 muestras (no varía ni por lote, ni por
chakra, ni por edad, ni por luz — es una propiedad de la especie, no de la
muestra). No sirve como target de un modelo tal cual está. Sirve como
metadata de contexto hasta que consigas un ensayo real (DPPH, IC50, etc.)
medido sobre estos extractos específicos — ahí sí tendría varianza real y
sería útil para modelar.

## 6. Anomalías en los datos crudos (sin corregir, tal como pediste)

4 de los 11 metabolitos identificados en el Excel tienen m/z que no
coincide con la Tabla 3 del artículo (se conservó el valor del Excel):

| Metabolito | m/z en Excel | m/z en Tabla 3 (artículo) | Nota |
|---|---|---|---|
| Neochlorogenic Acid | 177.0185 | 353.0883 | Diferencia grande, no es redondeo — posible fragmento en fuente o fila mal anotada |
| Kaempferol-3-O-glucoside | 284.0320 | 447.0925 | Coherente con pérdida de la glucosa (fragmentación en fuente del glicósido) |
| Hyperoside | 300.0266 | 463.0874 | Mismo patrón: posible pérdida de azúcar en fuente |
| Trehalose | 387.9157 | 387.1137 | Diferencia solo en la parte decimal — parece error de dígito, no fragmentación |

Además, **Keracyanine** aparece en la Tabla 3 del artículo con fórmula
`C27H31O15+` (catión permanente, típico de antocianinas) pero fue detectado
y reportado aquí en modo NEG — anotación probablemente heredada de un
match por masa sin considerar la carga intrínseca del compuesto. Se dejó
tal cual viene en el Excel.

Recomendación: antes de usar estos 4-5 metabolitos como variables con
nombre confiable en un modelo, valdría la pena confirmar con quien hizo la
anotación en MZmine/GNPS si son errores de transcripción o hallazgos reales
de fragmentación.

## 7. Siguiente paso

Cuando tengas el artículo complementario con bioactividad real (ref. [12] de
este paper — antioxidante, antidiabética, hemolítica), aviso y lo combino
usando `ID_MUESTRA_FISICA` como llave, sin duplicar taxonomía/metadata.

## 8. Batch 2 (POS) — 3 hojas incluidas, no solo una

El Excel `BATCH 2- MZmine_to_R_notame_pos.xlsx` trae 5 hojas. Dos son
exportaciones crudas de MZmine (`Export_mzmine_fullscan_pos_1/2`, sin
metadata de muestra) que no se usaron directamente. Las otras 3
(`Pos_1`, `Pos_2`, `Hoja1`) sí tienen formato notame completo, pero **no son
la misma tabla**: tienen `row_ID` propios e independientes (numeraciones de
MZmine distintas), y cada una anota un subconjunto distinto de los 5
metabolitos en modo positivo de la Tabla 3 del artículo. Probablemente
corresponden a las distintas estrategias de adquisición/fusión que describe
el artículo (FastDDA 5 vs 15, datos crudos vs. fusionados con full-scan).

Se incluyeron **las 3**, distinguidas por la columna `ORIGEN_PROCESAMIENTO`
en `03_picos.csv` (`BATCH2_Pos_1`, `BATCH2_Pos_2`, `BATCH2_Hoja1`), en vez de
elegir una sola. Donde una hoja no trae un dato que otra sí tiene, se dejó
en blanco/null en vez de inventarlo o copiarlo de otra hoja:

- **`Pos_1`** no tiene columna `Identification_level` → `NIVEL_IDENTIFICACION`
  vacío en las 51,409 filas de este pipeline.
- **`Hoja1`** no incluye la muestra `8_PB` (blanco de proceso) — tiene 100
  columnas de muestra en vez de 101; simplemente no genera filas para esa
  muestra en este pipeline (no es un error, esa hoja nunca la tuvo).
- Las muestras (`01_muestras.csv`, `LOTE=BATCH_2`) se registraron **una sola
  vez** a partir de `Pos_2` (la más completa) — `Pos_1` y `Hoja1` reusan
  exactamente los mismos `ID_MUESTRA` ya existentes, no se duplicó
  metadata de muestra por cada hoja.

Se verificó que el join `03_picos.csv.ID_MUESTRA -> 01_muestras.csv.ID_MUESTRA`
es 100% limpio (0 huérfanos) para las 4 combinaciones de pipeline, así que
siempre puedes recuperar `LOTE`, `TIPO_MUESTRA`, `MODO_IONIZACION`, etc. con
un merge simple.

Si más adelante confirmas cuál de las 3 hojas POS es "la" definitiva (por
ejemplo revisando el repositorio de GitHub del artículo), se puede filtrar
`03_picos.csv` por `ORIGEN_PROCESAMIENTO` sin tener que reprocesar nada.

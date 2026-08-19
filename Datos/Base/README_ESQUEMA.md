# Esquema base del dataset (molde)

Este dataset se construye en **4 tablas separadas** en vez de una sola tabla plana,
porque la información real tiene 4 "granos" distintos (4 cosas diferentes de las
que hay una cantidad distinta de registros):

| Tabla | 1 fila = | Cuántas filas |
|---|---|---|
| `01_especies.csv` | una **especie** (taxonomía y rasgos biológicos fijos) | 1 por especie — reusada por todos sus extractos |
| `02_muestras.csv` | una **corrida/inyección** en el equipo (un extracto en un modo de ionización) | 1 por corrida — un mismo extracto puede tener 2 (POS y NEG) |
| `03_muestra_actividad.csv` | una actividad biológica medida sobre un **extracto físico** | 0, 1 o varias por extracto |
| `04_picos.csv` | un pico/metabolito detectado por MS en una corrida | decenas, cientos o miles |

## Relación entre las tablas

```
ID_ESPECIE         (1 especie)  ──< 02_muestras                (N extractos/corridas de esa especie)
ID_MUESTRA_FISICA (1 extracto) ──< ID_MUESTRA (N corridas: POS, NEG, ...) ──< 04_picos  (N picos por corrida)
ID_MUESTRA_FISICA (1 extracto) ──< 03_muestra_actividad                    (N actividades por extracto)
```

Confirmaste que un mismo extracto se inyecta en modo POS y en modo NEG, generando
dos `ID_MUESTRA` distintos (ej. `100_MB_53_POS` y `100_MB_53_NEG`), pero la
actividad biológica (DPPH, IC50, etc.) se midió **una sola vez** sobre el
extracto. Por eso:

- La taxonomía y los rasgos biológicos fijos de una especie (reino…especie,
  tipo de planta, ciclo de vida, hábito de crecimiento) **no cambian entre
  extractos** de la misma especie — por eso viven en `01_especies.csv`, una
  tabla aparte enlazada por `ID_ESPECIE`. Así, si tienes 5 extractos de
  *Passiflora edulis* (distinta parte, solvente u origen), la taxonomía se
  escribe una sola vez y se corrige en un solo lugar.
- `02_muestras.csv` tiene **dos columnas de identificador**: `ID_MUESTRA`
  (la corrida específica, la que va en `04_picos.csv`) e `ID_MUESTRA_FISICA`
  (el extracto real, compartido por sus corridas POS y NEG, la que va en
  `03_muestra_actividad.csv`). Si en algún caso no hay pares POS/NEG y cada
  corrida es un extracto independiente, simplemente pon el mismo valor en
  ambas columnas (`ID_MUESTRA_FISICA = ID_MUESTRA`).
- `03_muestra_actividad.csv` se enlaza por `ID_MUESTRA_FISICA`, no por
  `ID_MUESTRA` — así el resultado del ensayo biológico no se duplica ni se
  contradice entre la corrida POS y la NEG del mismo extracto.
- `04_picos.csv` se enlaza por `ID_MUESTRA` (la corrida), porque el m/z sí
  depende del modo de ionización.

---

## 01_especies.csv

Una fila por **especie**. Taxonomía y rasgos biológicos que no cambian entre
extractos de la misma especie.

| Columna | Descripción | Notas |
|---|---|---|
| `ID_ESPECIE` | código único de la especie | llave que usa `02_muestras.csv` |
| `NOMBRE_CIENTIFICO` | género + especie (ej. `Passiflora edulis`) | |
| `REINO`…`ESPECIE` | jerarquía taxonómica completa | |
| `TIPO_PLANTA` | ej. árbol, arbusto, liana, hierba | |
| `CICLO_VIDA` | ej. anual, bienal, perenne | |
| `HABITO_CRECIMIENTO` | ej. trepadora, rastrera, erecta | |

**Blank y QC no tienen especie:** no crees una fila en esta tabla para
blancos ni controles de calidad — en `02_muestras.csv` su `ID_ESPECIE`
simplemente queda vacío.

---

## 02_muestras.csv

Una fila por **corrida/inyección**. Metadata fija de la muestra, la planta y
el método analítico.

| Columna | Descripción | Notas |
|---|---|---|
| `ID_MUESTRA` | código único de la corrida/archivo inyectado (ej. `100_MB_53_NEG`) | llave que usa `04_picos.csv` |
| `ID_MUESTRA_FISICA` | código del extracto físico real, compartido entre sus corridas POS/NEG (ej. `100_MB_53`) | llave que usa `03_muestra_actividad.csv` |
| `TIPO_MUESTRA` | `Sample` (muestra real), `Blank` (blanco de ruido), `QC` (pool de control de calidad de todas las muestras) o `SubQC` (pool de control por nivel de un factor, ej. solo muestras de una ubicación) | ver nota abajo, **muy importante** |
| `LOTE` | tanda o día en que se procesó esta corrida (o el identificador del batch de origen si no hay fecha por muestra) | sirve para detectar efecto lote/deriva del equipo entre días, **no es el lote de cosecha de la planta** |
| `ID_ESPECIE` | código de la especie | llave que usa `01_especies.csv`; ver nota Blank/QC |
| `TIPO_CULTIVO` | ej. silvestre, invernadero, campo abierto | |
| `PARTE_ESTUDIADA` | órgano del que se sacó el extracto (ej. hoja, raíz, corteza, fruto) | |
| `ORIGEN_GEOGRAFICO` | país, región o coordenadas de recolección | |
| `EXPOSICION_LUZ` | condición de luz del cultivo (ej. sombra, luz directa) | opcional — solo si el estudio la controla como factor experimental |
| `ESTADIO_MADUREZ` | edad o estado de madurez de la planta/hoja al momento de la recolección | opcional — solo si el estudio la controla como factor experimental |
| `METODO_EXTRACCION` | ej. maceración, Soxhlet, ultrasonido | |
| `SOLVENTE_EXTRACCION` | ej. metanol, etanol 80%, agua | |
| `MODO_IONIZACION` | `POS` o `NEG` | coincide con el sufijo de `ID_MUESTRA` |
| `COLUMNA_CROMATOGRAFICA` | ej. Fase Reversa (RP), HILIC | |

**Sobre `TIPO_MUESTRA` — Blank no es "planta"; QC/SubQC depende de qué se puso en el pool:**
un `Blank` (blanco, para medir ruido de fondo, ej. agua ultrapura) **no tiene especie
real** — deja vacíos `ID_ESPECIE`, `TIPO_CULTIVO` y `ORIGEN_GEOGRAFICO`, no inventes
datos ahí. Un `QC` (pool de control de calidad, mezcla de todas las muestras, inyectado
varias veces para medir estabilidad del equipo) o un `SubQC` (mezcla de las muestras que
comparten un nivel de un factor, ej. todas las de una misma ubicación, para resaltar el
efecto de ese factor) **sí pueden tener especie real** si el pool se hizo exclusivamente
con extracto de las propias muestras — en ese caso llena `ID_ESPECIE` igual que en
`Sample`. Si el pool mezcla varias especies o ubicaciones, deja `ORIGEN_GEOGRAFICO` (y
`EXPOSICION_LUZ`/`ESTADIO_MADUREZ` si aplica) vacíos o descritos como "pool multi-x" en
vez de forzar un solo valor. Estas filas (Blank, QC y SubQC) sí van a tener picos en
`04_picos.csv` (se usan para filtrar ruido y medir reproducibilidad de cada feature entre
corridas), pero **nunca** van a tener entradas en `03_muestra_actividad.csv`.

---

## 03_muestra_actividad.csv

Una fila por cada actividad biológica medida sobre un extracto. Si un
extracto es antioxidante Y antimicrobiano, son 2 filas.

| Columna | Descripción | Notas |
|---|---|---|
| `ID_MUESTRA_FISICA` | debe existir en `02_muestras.csv` | agrupa las corridas POS/NEG de ese extracto |
| `ACTIVIDAD_BIOLOGICA` | tipo de actividad: antioxidante, antimicrobiano, toxicidad, etc. | usa siempre el mismo texto exacto (ideal: lista cerrada de valores) |
| `VALOR_RESULTADO` | el número real del ensayo (IC50, % de inhibición, MIC...) | vacío si solo se sabe la categoría sin un valor cuantitativo |

**Por qué se separaron `ACTIVIDAD_BIOLOGICA` y `VALOR_RESULTADO`:** dijiste
que la actividad puede ser "la capacidad antioxidante, si inhibe una
bacteria, la toxicidad o un valor IC50" — eso son dos cosas distintas: **qué
tipo de actividad es** (categoría) y **cuánto dio el ensayo** (número). Con
las dos columnas separadas, el mismo esquema sirve tanto si vas a hacer
clasificación (¿tiene o no tiene actividad X? — con `VALOR_RESULTADO`
vacío) como si vas a hacer regresión (predecir el IC50 exacto — usando
`VALOR_RESULTADO`).

**Nota:** antes esta tabla también tenía `UNIDAD_RESULTADO`, `METODO_ENSAYO`
y `REFERENCIA_FUENTE` (unidad del valor, técnica del ensayo, y si el dato
viene de un ensayo propio o de literatura). Se quitaron para simplificar el
molde. Si más adelante necesitas distinguir un ensayo medido de un dato
heredado de literatura (importante para no confundir a un modelo — ver
riesgo de fuga por especie que hablamos antes), puedes anotarlo aparte o
recuperar esas columnas.

---

## 04_picos.csv

Una fila por cada pico/metabolito detectado en una corrida (`ID_MUESTRA`).

| Columna | Descripción | Notas |
|---|---|---|
| `ID_MUESTRA` | debe existir en `02_muestras.csv` (nivel corrida, no extracto) | |
| `ID_FEATURE` | identificador del pico/feature | déjalo vacío por ahora, se genera con script alineando m/z+RT entre todas las corridas |
| `RELACION_MASA_CARGA` ($m/z$) | "peso" de la molécula leído por el equipo | |
| `TIEMPO_RETENCION_MINUTOS` | minuto en que el compuesto salió de la columna | |
| `ALTURA_PICO` | abundancia/concentración relativa del compuesto | sé consistente: si mezclas altura y área entre corridas, anótalo |
| `NOMBRE_METABOLITO` | nombre del compuesto (ej. Ácido Quínico, Sacarosa) | vacío si no se identificó — válido y esperado |
| `NIVEL_IDENTIFICACION` | confianza de la identificación: I, II, III o IV (estándar internacional) | I = confirmado con estándar auténtico, II = anotado putativamente (librería espectral), III = clase química putativa, IV = desconocido |
| `CLASE_QUIMICA` / `SUPERCLASE_QUIMICA` | familia química (ej. Flavonoides, Terpenos, Alcaloides) | vacío si no identificado |

---

## Ejemplo completo (extracto con corridas POS+NEG, Blank y QC)

**`01_especies.csv`**
```
ID_ESPECIE,NOMBRE_CIENTIFICO,REINO,FILO,CLASE_TAXONOMICA,ORDEN,FAMILIA,GENERO,ESPECIE,TIPO_PLANTA,CICLO_VIDA,HABITO_CRECIMIENTO
ESP_001,Passiflora edulis,Plantae,Tracheophyta,Magnoliopsida,Malpighiales,Passifloraceae,Passiflora,edulis,Liana,Perenne,Trepadora
```

**`02_muestras.csv`**
```
ID_MUESTRA,ID_MUESTRA_FISICA,TIPO_MUESTRA,LOTE,ID_ESPECIE,...,MODO_IONIZACION,...
100_MB_53_POS,100_MB_53,Sample,LOTE_2026_02,ESP_001,...,POS,...
100_MB_53_NEG,100_MB_53,Sample,LOTE_2026_02,ESP_001,...,NEG,...
100_MB_BLK_02,100_MB_BLK_02,Blank,LOTE_2026_02,,...,NEG,...
100_MB_QC_07,100_MB_QC_07,QC,LOTE_2026_02,,...,NEG,...
```

**`03_muestra_actividad.csv`** (una sola vez por extracto, no se repite por POS/NEG)
```
ID_MUESTRA_FISICA,ACTIVIDAD_BIOLOGICA,VALOR_RESULTADO
100_MB_53,antioxidante,42.3
100_MB_53,antimicrobiano,
```

**`04_picos.csv`** (por corrida — POS y NEG tienen picos distintos)
```
ID_MUESTRA,ID_FEATURE,RELACION_MASA_CARGA,TIEMPO_RETENCION_MINUTOS,ALTURA_PICO,NOMBRE_METABOLITO,NIVEL_IDENTIFICACION,CLASE_QUIMICA,SUPERCLASE_QUIMICA
100_MB_53_NEG,,367.10,5.23,845210,Ácido clorogénico,I,Ácido fenólico,Polifenol
100_MB_53_POS,,369.12,5.24,612300,,IV,,
```

---

## Siguiente paso (cuando haya datos reales cargados)

Con estas 4 tablas llenas, el pivot a matriz "ancha" lista para modelar
(muestras × metabolitos, combinando POS+NEG por `ID_MUESTRA_FISICA`, con la
taxonomía traída de `01_especies.csv` vía `ID_ESPECIE`, más el target de
actividad biológica — categórico y/o numérico) se genera con un script. Ya
quedó esbozado en la conversación y lo puedo convertir en un `.py` real
dentro de esta carpeta cuando tengas los primeros datos consolidados.

---

## Dónde viven los datos reales

Esta carpeta (`Datos/Base/`) es el **molde vacío**: solo referencia de columnas, sin
filas. Los datos reales, ya llenados a partir de un artículo + sus batches de MZmine, se
guardan versionados en `Datos/Dataset/Consolidados/Version N/`, cada una con su propio
script de construcción (`build_versionN.py`) y notas de las decisiones tomadas
(`NOTAS_VERSIONN.md`). Qué artículo se cruzó con qué batch para producir cada versión
queda registrado en `Datos/Dataset/Consolidados/00_articulos_batches.csv`.

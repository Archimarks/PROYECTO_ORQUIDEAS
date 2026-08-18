# Esquema base del dataset (molde)

Este dataset se construye en **3 tablas separadas** en vez de una sola tabla plana,
porque la información real tiene 3 "granos" distintos (3 cosas diferentes de las
que hay una cantidad distinta de registros):

| Tabla | 1 fila = | Cuántas filas |
|---|---|---|
| `01_muestras.csv` | una **corrida/inyección** en el equipo (un extracto en un modo de ionización) | 1 por corrida — un mismo extracto puede tener 2 (POS y NEG) |
| `02_muestra_actividad.csv` | una actividad biológica medida sobre un **extracto físico** | 0, 1 o varias por extracto |
| `03_picos.csv` | un pico/metabolito detectado por MS en una corrida | decenas, cientos o miles |

## Relación entre las tablas

```
ID_MUESTRA_FISICA (1 extracto) ──< ID_MUESTRA (N corridas: POS, NEG, ...) ──< 03_picos  (N picos por corrida)
ID_MUESTRA_FISICA (1 extracto) ──< 02_muestra_actividad                    (N actividades por extracto)
```

Confirmaste que un mismo extracto se inyecta en modo POS y en modo NEG, generando
dos `ID_MUESTRA` distintos (ej. `100_MB_53_POS` y `100_MB_53_NEG`), pero la
actividad biológica (DPPH, IC50, etc.) se midió **una sola vez** sobre el
extracto. Por eso:

- `01_muestras.csv` tiene **dos columnas de identificador**: `ID_MUESTRA`
  (la corrida específica, la que va en `03_picos.csv`) e `ID_MUESTRA_FISICA`
  (el extracto real, compartido por sus corridas POS y NEG, la que va en
  `02_muestra_actividad.csv`). Si en algún caso no hay pares POS/NEG y cada
  corrida es un extracto independiente, simplemente pon el mismo valor en
  ambas columnas (`ID_MUESTRA_FISICA = ID_MUESTRA`).
- `02_muestra_actividad.csv` se enlaza por `ID_MUESTRA_FISICA`, no por
  `ID_MUESTRA` — así el resultado del ensayo biológico no se duplica ni se
  contradice entre la corrida POS y la NEG del mismo extracto.
- `03_picos.csv` se enlaza por `ID_MUESTRA` (la corrida), porque el m/z sí
  depende del modo de ionización.

---

## 01_muestras.csv

Una fila por **corrida/inyección**. Metadata fija de la muestra, la planta y
el método analítico.

| Columna | Descripción | Notas |
|---|---|---|
| `ID_MUESTRA` | código único de la corrida/archivo inyectado (ej. `100_MB_53_NEG`) | llave que usa `03_picos.csv` |
| `ID_MUESTRA_FISICA` | código del extracto físico real, compartido entre sus corridas POS/NEG (ej. `100_MB_53`) | llave que usa `02_muestra_actividad.csv` |
| `TIPO_MUESTRA` | `Sample` (muestra real), `Blank` (blanco de ruido) o `QC` (control de calidad) | ver nota abajo, **muy importante** |
| `LOTE` | tanda o día en que se procesó esta corrida | sirve para detectar efecto lote/deriva del equipo entre días, **no es el lote de cosecha de la planta** |
| `REINO`…`ESPECIE` | jerarquía taxonómica completa | ver nota Blank/QC |
| `NOMBRE_CIENTIFICO` | género + especie (ej. `Passiflora edulis`) | |
| `TIPO_PLANTA` | ej. árbol, arbusto, liana, hierba | |
| `CICLO_VIDA` | ej. anual, bienal, perenne | |
| `HABITO_CRECIMIENTO` | ej. trepadora, rastrera, erecta | |
| `TIPO_CULTIVO` | ej. silvestre, invernadero, campo abierto | |
| `PARTE_ESTUDIADA` | órgano del que se sacó el extracto (ej. hoja, raíz, corteza, fruto) | |
| `ORIGEN_GEOGRAFICO` | país, región o coordenadas de recolección | |
| `METODO_EXTRACCION` | ej. maceración, Soxhlet, ultrasonido | |
| `SOLVENTE_EXTRACCION` | ej. metanol, etanol 80%, agua | |
| `MODO_IONIZACION` | `POS` o `NEG` | coincide con el sufijo de `ID_MUESTRA` |
| `COLUMNA_CROMATOGRAFICA` | ej. Fase Reversa (RP), HILIC | |

**Sobre `TIPO_MUESTRA` — Blank y QC no son "plantas":** un `Blank` (blanco,
para medir ruido de fondo) y un `QC` (pool de control de calidad, inyectado
varias veces para medir estabilidad del equipo) **no tienen taxonomía real**.
Cuando `TIPO_MUESTRA` sea `Blank` o `QC`, es correcto y esperado dejar vacías
`REINO`…`ORIGEN_GEOGRAFICO` — no inventes datos ahí. Estas filas sí van a
tener picos en `03_picos.csv` (se usan para filtrar ruido y medir
reproducibilidad de cada feature entre corridas de QC), pero **nunca** van a
tener entradas en `02_muestra_actividad.csv`.

---

## 02_muestra_actividad.csv

Una fila por cada actividad biológica medida sobre un extracto. Si un
extracto es antioxidante Y antimicrobiano, son 2 filas.

| Columna | Descripción | Notas |
|---|---|---|
| `ID_MUESTRA_FISICA` | debe existir en `01_muestras.csv` | agrupa las corridas POS/NEG de ese extracto |
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

## 03_picos.csv

Una fila por cada pico/metabolito detectado en una corrida (`ID_MUESTRA`).

| Columna | Descripción | Notas |
|---|---|---|
| `ID_MUESTRA` | debe existir en `01_muestras.csv` (nivel corrida, no extracto) | |
| `ID_FEATURE` | identificador del pico/feature | déjalo vacío por ahora, se genera con script alineando m/z+RT entre todas las corridas |
| `RELACION_MASA_CARGA` ($m/z$) | "peso" de la molécula leído por el equipo | |
| `TIEMPO_RETENCION_MINUTOS` | minuto en que el compuesto salió de la columna | |
| `ALTURA_PICO` | abundancia/concentración relativa del compuesto | sé consistente: si mezclas altura y área entre corridas, anótalo |
| `NOMBRE_METABOLITO` | nombre del compuesto (ej. Ácido Quínico, Sacarosa) | vacío si no se identificó — válido y esperado |
| `NIVEL_IDENTIFICACION` | confianza de la identificación: I, II, III o IV (estándar internacional) | I = confirmado con estándar auténtico, II = anotado putativamente (librería espectral), III = clase química putativa, IV = desconocido |
| `CLASE_QUIMICA` / `SUPERCLASE_QUIMICA` | familia química (ej. Flavonoides, Terpenos, Alcaloides) | vacío si no identificado |

---

## Ejemplo completo (extracto con corridas POS+NEG, Blank y QC)

**`01_muestras.csv`**
```
ID_MUESTRA,ID_MUESTRA_FISICA,TIPO_MUESTRA,LOTE,...,GENERO,ESPECIE,...,MODO_IONIZACION,...
100_MB_53_POS,100_MB_53,Sample,LOTE_2026_02,...,Passiflora,edulis,...,POS,...
100_MB_53_NEG,100_MB_53,Sample,LOTE_2026_02,...,Passiflora,edulis,...,NEG,...
100_MB_BLK_02,100_MB_BLK_02,Blank,LOTE_2026_02,...,,,...,NEG,...
100_MB_QC_07,100_MB_QC_07,QC,LOTE_2026_02,...,,,...,NEG,...
```

**`02_muestra_actividad.csv`** (una sola vez por extracto, no se repite por POS/NEG)
```
ID_MUESTRA_FISICA,ACTIVIDAD_BIOLOGICA,VALOR_RESULTADO
100_MB_53,antioxidante,42.3
100_MB_53,antimicrobiano,
```

**`03_picos.csv`** (por corrida — POS y NEG tienen picos distintos)
```
ID_MUESTRA,ID_FEATURE,RELACION_MASA_CARGA,TIEMPO_RETENCION_MINUTOS,ALTURA_PICO,NOMBRE_METABOLITO,NIVEL_IDENTIFICACION,CLASE_QUIMICA,SUPERCLASE_QUIMICA
100_MB_53_NEG,,367.10,5.23,845210,Ácido clorogénico,I,Ácido fenólico,Polifenol
100_MB_53_POS,,369.12,5.24,612300,,IV,,
```

---

## Siguiente paso (cuando haya datos reales cargados)

Con estas 3 tablas llenas, el pivot a matriz "ancha" lista para modelar
(muestras × metabolitos, combinando POS+NEG por `ID_MUESTRA_FISICA`, más el
target de actividad biológica — categórico y/o numérico) se genera con un
script. Ya quedó esbozado en la conversación y lo puedo convertir en un
`.py` real dentro de esta carpeta cuando tengas los primeros datos
consolidados.

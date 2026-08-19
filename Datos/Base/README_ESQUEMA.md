# Esquema base del dataset (molde)

Este dataset está normalizado en **30 tablas**: 25 son **catálogos** (listas
cortas de referencia, con un ID + el valor + una descripción, reutilizadas
por muchas filas) y 5 son tablas **hub/hecho** (`especies`, `muestras`,
`muestra_factor`, `especie_actividad`, `picos`) que se quedan sueltas en la
raíz porque son las que se llenan directamente con datos reales. Los
catálogos están agrupados en subcarpetas por dominio para que la carpeta
siga siendo navegable.

```
Datos/Base/
├── 01_especies.csv            (hub — 1 fila por especie)
├── 02_muestras.csv            (hecho)
├── 03_muestra_factor.csv      (hecho)
├── 04_especie_actividad.csv   (hecho)
├── 05_picos.csv               (hecho)
├── Catalogos_Especie/
│   ├── tipos_planta.csv
│   ├── ciclos_vida.csv
│   ├── habitos_crecimiento.csv
│   ├── reinos.csv
│   ├── filos.csv
│   ├── clases_taxonomicas.csv
│   ├── ordenes.csv
│   ├── familias.csv
│   └── generos.csv
├── Catalogos_Actividad_Biologica/
│   ├── actividades_biologicas.csv
│   ├── objetivos_actividad.csv
│   ├── metricas_ensayo.csv
│   ├── unidades.csv
│   ├── condiciones_ensayo.csv
│   └── referencias.csv
├── Catalogos_Muestras/
│   ├── tipos_muestra.csv
│   ├── tipos_cultivo.csv
│   ├── partes_planta.csv
│   ├── ubicaciones.csv
│   ├── metodos_extraccion.csv
│   ├── solventes_extraccion.csv
│   ├── columnas_cromatograficas.csv
│   └── factores_experimentales.csv
└── Catalogos_Picos/
    ├── metabolitos.csv
    └── niveles_identificacion.csv
```

`01_especies.csv` vive en la raíz (no dentro de `Catalogos_Especie/`) porque,
aunque tiene forma de catálogo, es un **hub**: lo referencian directamente
`02_muestras.csv` y `04_especie_actividad.csv`, igual que ellas se llena con
una fila real por cada especie del dataset — no es una lista cerrada de
categorías como sí lo son sus catálogos satélite (`tipos_planta`, `reinos`...).

## Regla de IDs: siempre número entero

**Todo `ID_X` es un número entero autoincremental (1, 2, 3...), nunca un
código con prefijo de texto** (nada de `ACT_01`, `TP_01`, `ILEX_GUAYUSA`).
Cuando ese identificador natural/legible (un código de laboratorio, un
nombre científico) es útil conservarlo, vive en una columna **aparte**,
nunca mezclado en el propio ID:

- `01_especies.csv`: `ID_ESPECIE` es un entero; el nombre legible ya vive en
  `NOMBRE_CIENTIFICO` (ej. `Ilex guayusa`) — no hace falta una columna de
  código adicional, ese campo ya cumple ese rol.
- `02_muestras.csv`: `ID_MUESTRA` e `ID_MUESTRA_FISICA` son enteros; el
  código real de laboratorio/archivo (ej. `100_MB_53_NEG`, `100_MB_53`) vive
  en las columnas nuevas `CODIGO_MUESTRA` y `CODIGO_MUESTRA_FISICA`.
- Los 25 catálogos ya traían su valor legible en una columna separada
  (`ACTIVIDAD_BIOLOGICA`, `TIPO_PLANTA`, `REINO`...), así que para ellos el
  cambio es solo de tipo de dato en `ID_X`: entero en vez de código con
  prefijo, sin necesidad de agregar ninguna columna.

Todas las tablas de hechos (`02_muestras.csv`, `03_muestra_factor.csv`,
`04_especie_actividad.csv`, `05_picos.csv`) referencian estos catálogos por
ese entero.

---

## Filosofía: ¿catálogo o columna normal?

Regla que se aplicó en todas las tablas: si un valor **se repite entre
filas** y tiene un vocabulario más o menos cerrado (un tipo de muestra, un
solvente, una unidad, una cita, un rango taxonómico...), se saca a su propio
catálogo con ID. Si el valor es **específico de esa fila** y no tiene
sentido reutilizarlo (un código de muestra, una coordenada RT/m-z, el número
exacto de un resultado), se queda como columna normal en la tabla de hechos.

Esto evolucionó en varios pasos dentro de esta misma conversación: primero
`ACTIVIDAD_BIOLOGICA` era un texto libre que mezclaba categoría + objetivo +
método, lo que fragmentaba la misma actividad real en categorías falsas al
agrupar. Se separó en catálogos, luego se aplicó la misma lógica a
`muestras`, `especie_actividad` y `picos`, después a la taxonomía de
`especies` (`REINO`…`GENERO`), luego se agruparon los 25 catálogos en
subcarpetas por dominio, luego se estandarizó que todo ID fuera un entero
puro en vez de un código con prefijo inventado por mí, y finalmente
`EXPOSICION_LUZ`/`ESTADIO_MADUREZ` (columnas fijas específicas del diseño
experimental de *este* estudio) se sacaron a un catálogo de factores +
`03_muestra_factor.csv`, porque otro estudio va a medir factores distintos
(temperatura, tipo de fertilización...) y columnas fijas no escalan.

---

## Resumen de tablas

| Carpeta / archivo | Tipo | Contiene |
|---|---|---|
| `01_especies.csv` | **hub** | una especie (taxonomía y rasgos) |
| `02_muestras.csv` | **hecho** | una corrida/inyección en el equipo |
| `03_muestra_factor.csv` | **hecho** | un factor experimental medido sobre una muestra (luz, edad, temperatura...) |
| `04_especie_actividad.csv` | **hecho** | una actividad biológica reportada para una especie |
| `05_picos.csv` | **hecho** | un pico/metabolito detectado por MS en una corrida |
| `Catalogos_Especie/tipos_planta.csv` | catálogo | árbol, arbusto, liana, hierba |
| `Catalogos_Especie/ciclos_vida.csv` | catálogo | anual, bienal, perenne |
| `Catalogos_Especie/habitos_crecimiento.csv` | catálogo | trepadora, rastrera, erecta |
| `Catalogos_Especie/reinos.csv` | catálogo | reino taxonómico |
| `Catalogos_Especie/filos.csv` | catálogo | filo taxonómico |
| `Catalogos_Especie/clases_taxonomicas.csv` | catálogo | clase taxonómica |
| `Catalogos_Especie/ordenes.csv` | catálogo | orden taxonómico |
| `Catalogos_Especie/familias.csv` | catálogo | familia taxonómica |
| `Catalogos_Especie/generos.csv` | catálogo | género taxonómico |
| `Catalogos_Actividad_Biologica/actividades_biologicas.csv` | catálogo | categoría general de actividad (Antimicrobiano...) |
| `Catalogos_Actividad_Biologica/objetivos_actividad.csv` | catálogo | sobre qué actúa (*S. aureus*, óxido nítrico...) |
| `Catalogos_Actividad_Biologica/metricas_ensayo.csv` | catálogo | tipo de métrica (MIC, LD50, LC50...) |
| `Catalogos_Actividad_Biologica/unidades.csv` | catálogo | unidad de medida (mg/mL, mm, %...) |
| `Catalogos_Actividad_Biologica/condiciones_ensayo.csv` | catálogo | preparación/condición del ensayo |
| `Catalogos_Actividad_Biologica/referencias.csv` | catálogo | citas bibliográficas |
| `Catalogos_Muestras/tipos_muestra.csv` | catálogo | Sample / Blank / QC / SubQC |
| `Catalogos_Muestras/tipos_cultivo.csv` | catálogo | sistema de cultivo |
| `Catalogos_Muestras/partes_planta.csv` | catálogo | órgano del que se sacó el extracto |
| `Catalogos_Muestras/ubicaciones.csv` | catálogo | sitio de recolección |
| `Catalogos_Muestras/metodos_extraccion.csv` | catálogo | técnica de extracción |
| `Catalogos_Muestras/solventes_extraccion.csv` | catálogo | solvente usado |
| `Catalogos_Muestras/columnas_cromatograficas.csv` | catálogo | columna del equipo LC |
| `Catalogos_Muestras/factores_experimentales.csv` | catálogo | nombre de un factor/variable experimental (luz, edad, temperatura...) |
| `Catalogos_Picos/metabolitos.csv` | catálogo | nombre de compuesto + clase/superclase química |
| `Catalogos_Picos/niveles_identificacion.csv` | catálogo | confianza de identificación MS (I–IV) |

---

## Relación entre las tablas

```
                                 ┌─< Catalogos_Especie/tipos_planta
                                 ├─< Catalogos_Especie/ciclos_vida
01_especies ─────────────────────┼─< Catalogos_Especie/habitos_crecimiento
   │  (ID_ESPECIE, entero)       └─< Catalogos_Especie/{reinos,filos,
   │                                  clases_taxonomicas,ordenes,
   │                                  familias,generos}
   │
   ├──< 02_muestras ──< 05_picos
   │        │  │           └─< Catalogos_Picos/{metabolitos,niveles_identificacion}
   │        │  └─< Catalogos_Muestras/{tipos_muestra,tipos_cultivo,partes_planta,
   │        │      ubicaciones,metodos_extraccion,solventes_extraccion,
   │        │      columnas_cromatograficas}
   │        │
   │        └──< 03_muestra_factor
   │                 └─< Catalogos_Muestras/factores_experimentales
   │
   └──< 04_especie_actividad
            └─< Catalogos_Actividad_Biologica/{actividades_biologicas,
                objetivos_actividad,metricas_ensayo,unidades,
                condiciones_ensayo,referencias}
```

Un extracto se inyecta en modo POS y en modo NEG, generando dos `ID_MUESTRA`
distintos que comparten `ID_MUESTRA_FISICA`. La actividad biológica, en
cambio, casi nunca viene del mismo estudio ni del mismo extracto físico que
la metabolómica MS — se reporta en la literatura **a nivel de especie**, en
artículos aparte. Por eso `04_especie_actividad.csv` se enlaza por
`ID_ESPECIE` y no por `ID_MUESTRA_FISICA`. Los factores experimentales
(`03_muestra_factor.csv`) tampoco son columnas fijas de `02_muestras.csv` —
cada estudio define sus propios factores en
`Catalogos_Muestras/factores_experimentales.csv` y los registra ahí, uno por
fila, en vez de forzar columnas como `EXPOSICION_LUZ`/`ESTADIO_MADUREZ` que
solo tienen sentido para el diseño experimental de este estudio en
particular.

---

## 01_especies.csv

Una fila por **especie**. Taxonomía y rasgos biológicos que no cambian entre
extractos de la misma especie.

| Columna | Descripción | Notas |
|---|---|---|
| `ID_ESPECIE` | entero autoincremental | llave que usan `02_muestras.csv` y `04_especie_actividad.csv` |
| `NOMBRE_CIENTIFICO` | género + especie (ej. `Passiflora edulis`) | la etiqueta legible de la especie |
| `ID_REINO` | debe existir en `Catalogos_Especie/reinos.csv` | |
| `ID_FILO` | debe existir en `Catalogos_Especie/filos.csv` | |
| `ID_CLASE_TAXONOMICA` | debe existir en `Catalogos_Especie/clases_taxonomicas.csv` | |
| `ID_ORDEN` | debe existir en `Catalogos_Especie/ordenes.csv` | |
| `ID_FAMILIA` | debe existir en `Catalogos_Especie/familias.csv` | |
| `ID_GENERO` | debe existir en `Catalogos_Especie/generos.csv` | |
| `ESPECIE` | epíteto específico (ej. `edulis` en *Passiflora edulis*) | texto libre, casi 1:1 con la especie, no se catalogó |
| `ID_TIPO_PLANTA` | debe existir en `Catalogos_Especie/tipos_planta.csv` | ej. árbol, arbusto, liana, hierba |
| `ID_CICLO_VIDA` | debe existir en `Catalogos_Especie/ciclos_vida.csv` | ej. anual, bienal, perenne |
| `ID_HABITO_CRECIMIENTO` | debe existir en `Catalogos_Especie/habitos_crecimiento.csv` | ej. trepadora, rastrera, erecta |

Los 6 catálogos de taxonomía no están modelados como jerarquía anidada entre
sí (cada uno es una lista plana independiente) — se van a repetir en cuanto
agregues más especies, pero navegar la jerarquía completa entre ellos no
hacía falta con una sola especie cargada.

**Blank y QC no tienen especie:** no crees una fila en esta tabla para
blancos ni controles de calidad — en `02_muestras.csv` su `ID_ESPECIE`
simplemente queda vacío.

---

## 02_muestras.csv

Una fila por **corrida/inyección**. Metadata fija de la muestra, la planta y
el método analítico — casi todo referenciado por ID entero a un catálogo.

| Columna | Descripción | Notas |
|---|---|---|
| `ID_MUESTRA` | entero autoincremental | llave que usan `03_muestra_factor.csv` y `05_picos.csv` |
| `CODIGO_MUESTRA` | código real de laboratorio/archivo (ej. `100_MB_53_NEG`) | el identificador legible — ya no vive en `ID_MUESTRA` |
| `ID_MUESTRA_FISICA` | entero autoincremental | agrupa las corridas del mismo extracto (comparten el mismo valor) |
| `CODIGO_MUESTRA_FISICA` | código real del extracto físico (ej. `100_MB_53`) | el identificador legible del extracto |
| `ID_TIPO_MUESTRA` | debe existir en `Catalogos_Muestras/tipos_muestra.csv` | Sample / Blank / QC / SubQC — ver nota abajo, **muy importante** |
| `LOTE` | tanda o día en que se procesó esta corrida (o el batch de origen si no hay fecha por muestra) | texto libre, no un catálogo |
| `ID_ESPECIE` | debe existir en `01_especies.csv` | vacío para Blank; ver nota abajo |
| `ID_TIPO_CULTIVO` | debe existir en `Catalogos_Muestras/tipos_cultivo.csv` | vacío para Blank |
| `ID_PARTE_PLANTA` | debe existir en `Catalogos_Muestras/partes_planta.csv` | vacío para Blank |
| `ID_UBICACION` | debe existir en `Catalogos_Muestras/ubicaciones.csv` | vacío para Blank |
| `ID_METODO_EXTRACCION` | debe existir en `Catalogos_Muestras/metodos_extraccion.csv` | |
| `ID_SOLVENTE` | debe existir en `Catalogos_Muestras/solventes_extraccion.csv` | |
| `MODO_IONIZACION` | `POS` o `NEG` | coincide con el sufijo de `CODIGO_MUESTRA` — no se catalogó, es un enum de 2 valores fijo del método analítico |
| `ID_COLUMNA` | debe existir en `Catalogos_Muestras/columnas_cromatograficas.csv` | |

**¿Dónde quedaron `EXPOSICION_LUZ` y `ESTADIO_MADUREZ`?** Ya no son columnas
de esta tabla — eran específicas del diseño experimental de *este* estudio
(chakra × edad × luz), y otro estudio va a medir factores distintos
(temperatura, tipo de fertilización, época de cosecha...). Se movieron a
`03_muestra_factor.csv`, ver esa sección.

**Sobre `ID_TIPO_MUESTRA` — Blank no es "planta"; QC/SubQC depende de qué se
puso en el pool:** un `Blank` (blanco, para medir ruido de fondo, ej. agua
ultrapura) **no tiene especie real** — deja vacíos `ID_ESPECIE`,
`ID_TIPO_CULTIVO`, `ID_UBICACION`, no inventes datos ahí. Un `QC` (pool de
control de calidad, mezcla de todas las muestras) o un `SubQC` (mezcla de
muestras que comparten un nivel de un factor, ej. una sola ubicación) **sí
pueden tener especie real** si el pool se hizo exclusivamente con extracto de
las propias muestras. Si el pool mezcla varias especies o ubicaciones, deja
`ID_UBICACION` vacío o usa una fila de `Catalogos_Muestras/ubicaciones.csv`
tipo "pool multi-ubicación" en vez de forzar un solo valor. Estas filas
(Blank, QC y SubQC) sí van a tener picos en `05_picos.csv`, pero como
`04_especie_actividad.csv` se enlaza por `ID_ESPECIE` (que ellas no tienen),
quedan automáticamente fuera de esa tabla sin necesitar una regla aparte.

---

## 03_muestra_factor.csv

Una fila por cada factor/variable experimental medida sobre una muestra. Si
una muestra se registró bajo 3 factores (ej. ubicación, edad y luz — aunque
`ID_UBICACION` ya vive en `02_muestras.csv` porque casi siempre aplica, los
demás factores del diseño van aquí), son 3 filas.

| Columna | Descripción | Notas |
|---|---|---|
| `ID_MUESTRA` | debe existir en `02_muestras.csv` | |
| `ID_FACTOR` | debe existir en `Catalogos_Muestras/factores_experimentales.csv` | ej. exposición de luz, estadio de madurez, temperatura... |
| `VALOR` | el valor de ese factor para esta muestra (ej. `Sombra`, `Temprana (4-6 años)`) | texto libre — el valor en sí no se catalogó porque varía mucho según el factor |

**Por qué esta tabla existe en vez de columnas fijas como
`EXPOSICION_LUZ`/`ESTADIO_MADUREZ`:** esas columnas eran específicas del
diseño factorial de *este* estudio de Ilex guayusa. Un estudio distinto
podría medir temperatura de cultivo, tipo de fertilización, época de
cosecha, humedad relativa, o cualquier otra variable — columnas fijas en el
molde no escalan a estudios futuros con diseños experimentales diferentes.
Con este catálogo + tabla de hechos, cada estudio registra los factores que
le apliquen sin tocar el esquema, exactamente el mismo patrón que se usó
para `ACTIVIDAD_BIOLOGICA`.

---

## 04_especie_actividad.csv

Una fila por cada actividad biológica reportada para una **especie**, sobre
un objetivo y con una métrica concretos — no por extracto físico individual.

| Columna | Descripción | Notas |
|---|---|---|
| `ID_ESPECIE` | debe existir en `01_especies.csv` | a nivel de especie completa, no del extracto usado en la MS |
| `ID_ACTIVIDAD` | debe existir en `Catalogos_Actividad_Biologica/actividades_biologicas.csv` | categoría general: Antimicrobiano, Antioxidante... |
| `ID_OBJETIVO` | debe existir en `Catalogos_Actividad_Biologica/objetivos_actividad.csv` | sobre qué actúa: el organismo, molécula o compuesto concreto |
| `ID_METRICA` | debe existir en `Catalogos_Actividad_Biologica/metricas_ensayo.csv` | cómo se midió: MIC, LD50, LC50, Halo de inhibición, % remoción... |
| `VALOR_NUMERICO` | el número (o rango, o valor con `>`/`<`) del resultado | ej. `18`, `0.50-1.00`, `>5000` — se deja como texto porque no siempre es un float puro |
| `ID_UNIDAD` | debe existir en `Catalogos_Actividad_Biologica/unidades.csv` | mg/mL, mm, %, µg/mL... |
| `ID_CONDICION_ENSAYO` | debe existir en `Catalogos_Actividad_Biologica/condiciones_ensayo.csv` | preparación/condición bajo la que se midió — puede ir vacío |
| `ID_REFERENCIA` | debe existir en `Catalogos_Actividad_Biologica/referencias.csv` | cita del artículo/estudio de donde sale este valor |

**Por qué se descompuso `VALOR_RESULTADO` en 4 columnas:** antes era un solo
texto como `"MIC = 18 mg/mL (infusion acuosa)"`, que mezclaba la métrica, el
número, la unidad y la condición del ensayo. Con columnas separadas, cada
pregunta se responde con un filtro directo por ID.

---

## 05_picos.csv

Una fila por cada pico/metabolito detectado en una corrida (`ID_MUESTRA`).

| Columna | Descripción | Notas |
|---|---|---|
| `ID_MUESTRA` | debe existir en `02_muestras.csv` (nivel corrida, no extracto) | |
| `ID_FEATURE` | identificador entero del pico/feature | vacío hasta correr el script de alineación m/z+RT |
| `RELACION_MASA_CARGA` ($m/z$) | "peso" de la molécula leído por el equipo | |
| `TIEMPO_RETENCION_MINUTOS` | minuto en que el compuesto salió de la columna | |
| `ALTURA_PICO` | abundancia/concentración relativa del compuesto | sé consistente: si mezclas altura y área entre corridas, anótalo |
| `ID_METABOLITO` | debe existir en `Catalogos_Picos/metabolitos.csv` | vacío si no se identificó — válido y esperado (la mayoría de los picos no se identifican) |
| `ID_NIVEL` | debe existir en `Catalogos_Picos/niveles_identificacion.csv` | confianza de la identificación, I a IV — se llena incluso si `ID_METABOLITO` está vacío (nivel IV = desconocido) |

---

## Catálogos simples (mismo patrón: ID + valor + descripción)

Todos comparten la misma forma: `ID_X` (entero), `X` (el valor legible),
`DESCRIPCION` (el de `Catalogos_Picos/metabolitos.csv` además trae
`CLASE_QUIMICA` y `SUPERCLASE_QUIMICA`). Se crea una fila la primera vez que
aparece ese valor y se reutiliza el ID en todas las filas de hechos que lo
necesiten.

| Archivo | Columna ID | Columna valor | Referenciado desde |
|---|---|---|---|
| `Catalogos_Actividad_Biologica/actividades_biologicas.csv` | `ID_ACTIVIDAD` | `ACTIVIDAD_BIOLOGICA` | `04_especie_actividad.ID_ACTIVIDAD` |
| `Catalogos_Actividad_Biologica/objetivos_actividad.csv` | `ID_OBJETIVO` | `OBJETIVO` | `04_especie_actividad.ID_OBJETIVO` |
| `Catalogos_Actividad_Biologica/metricas_ensayo.csv` | `ID_METRICA` | `METRICA` | `04_especie_actividad.ID_METRICA` |
| `Catalogos_Actividad_Biologica/unidades.csv` | `ID_UNIDAD` | `UNIDAD` | `04_especie_actividad.ID_UNIDAD` |
| `Catalogos_Actividad_Biologica/condiciones_ensayo.csv` | `ID_CONDICION_ENSAYO` | `CONDICION_ENSAYO` | `04_especie_actividad.ID_CONDICION_ENSAYO` |
| `Catalogos_Actividad_Biologica/referencias.csv` | `ID_REFERENCIA` | `REFERENCIA_CITA` | `04_especie_actividad.ID_REFERENCIA` |
| `Catalogos_Muestras/tipos_muestra.csv` | `ID_TIPO_MUESTRA` | `TIPO_MUESTRA` | `02_muestras.ID_TIPO_MUESTRA` |
| `Catalogos_Muestras/tipos_cultivo.csv` | `ID_TIPO_CULTIVO` | `TIPO_CULTIVO` | `02_muestras.ID_TIPO_CULTIVO` |
| `Catalogos_Muestras/partes_planta.csv` | `ID_PARTE_PLANTA` | `PARTE_ESTUDIADA` | `02_muestras.ID_PARTE_PLANTA` |
| `Catalogos_Muestras/ubicaciones.csv` | `ID_UBICACION` | `ORIGEN_GEOGRAFICO` | `02_muestras.ID_UBICACION` |
| `Catalogos_Muestras/metodos_extraccion.csv` | `ID_METODO_EXTRACCION` | `METODO_EXTRACCION` | `02_muestras.ID_METODO_EXTRACCION` |
| `Catalogos_Muestras/solventes_extraccion.csv` | `ID_SOLVENTE` | `SOLVENTE_EXTRACCION` | `02_muestras.ID_SOLVENTE` |
| `Catalogos_Muestras/columnas_cromatograficas.csv` | `ID_COLUMNA` | `COLUMNA_CROMATOGRAFICA` | `02_muestras.ID_COLUMNA` |
| `Catalogos_Muestras/factores_experimentales.csv` | `ID_FACTOR` | `FACTOR` | `03_muestra_factor.ID_FACTOR` |
| `Catalogos_Picos/metabolitos.csv` | `ID_METABOLITO` | `NOMBRE_METABOLITO` (+ `CLASE_QUIMICA`, `SUPERCLASE_QUIMICA`) | `05_picos.ID_METABOLITO` |
| `Catalogos_Picos/niveles_identificacion.csv` | `ID_NIVEL` | `NIVEL_IDENTIFICACION` (I–IV) | `05_picos.ID_NIVEL` |
| `Catalogos_Especie/tipos_planta.csv` | `ID_TIPO_PLANTA` | `TIPO_PLANTA` | `01_especies.ID_TIPO_PLANTA` |
| `Catalogos_Especie/ciclos_vida.csv` | `ID_CICLO_VIDA` | `CICLO_VIDA` | `01_especies.ID_CICLO_VIDA` |
| `Catalogos_Especie/habitos_crecimiento.csv` | `ID_HABITO_CRECIMIENTO` | `HABITO_CRECIMIENTO` | `01_especies.ID_HABITO_CRECIMIENTO` |
| `Catalogos_Especie/reinos.csv` | `ID_REINO` | `REINO` | `01_especies.ID_REINO` |
| `Catalogos_Especie/filos.csv` | `ID_FILO` | `FILO` | `01_especies.ID_FILO` |
| `Catalogos_Especie/clases_taxonomicas.csv` | `ID_CLASE_TAXONOMICA` | `CLASE_TAXONOMICA` | `01_especies.ID_CLASE_TAXONOMICA` |
| `Catalogos_Especie/ordenes.csv` | `ID_ORDEN` | `ORDEN` | `01_especies.ID_ORDEN` |
| `Catalogos_Especie/familias.csv` | `ID_FAMILIA` | `FAMILIA` | `01_especies.ID_FAMILIA` |
| `Catalogos_Especie/generos.csv` | `ID_GENERO` | `GENERO` | `01_especies.ID_GENERO` |

---

## Ejemplo mínimo (una especie, un resultado de actividad, un pico)

**`01_especies.csv`**
```
ID_ESPECIE,NOMBRE_CIENTIFICO,ID_REINO,ID_FILO,ID_CLASE_TAXONOMICA,ID_ORDEN,ID_FAMILIA,ID_GENERO,ESPECIE,ID_TIPO_PLANTA,ID_CICLO_VIDA,ID_HABITO_CRECIMIENTO
1,Passiflora edulis,1,1,1,1,1,1,edulis,1,1,1
```

**`Catalogos_Especie/tipos_planta.csv` / `ciclos_vida.csv` / `reinos.csv` / ... / `generos.csv`**
```
ID_TIPO_PLANTA,TIPO_PLANTA,DESCRIPCION
1,Liana,

ID_REINO,REINO,DESCRIPCION
1,Plantae,
```

**`02_muestras.csv`**
```
ID_MUESTRA,CODIGO_MUESTRA,ID_MUESTRA_FISICA,CODIGO_MUESTRA_FISICA,ID_TIPO_MUESTRA,LOTE,ID_ESPECIE,...,MODO_IONIZACION,...
1,100_MB_53_POS,1,100_MB_53,1,LOTE_2026_02,1,...,POS,...
2,100_MB_53_NEG,1,100_MB_53,1,LOTE_2026_02,1,...,NEG,...
```

**`03_muestra_factor.csv`** (usa `Catalogos_Muestras/factores_experimentales.csv`)
```
ID_MUESTRA,ID_FACTOR,VALOR
1,1,Luz
1,2,Temprana (4-6 años)
2,1,Luz
2,2,Temprana (4-6 años)
```

**`04_especie_actividad.csv`** (usa los 6 catálogos de `Catalogos_Actividad_Biologica/`)
```
ID_ESPECIE,ID_ACTIVIDAD,ID_OBJETIVO,ID_METRICA,VALOR_NUMERICO,ID_UNIDAD,ID_CONDICION_ENSAYO,ID_REFERENCIA
1,1,1,1,42.3,1,1,1
```

**`05_picos.csv`** (usa los 2 catálogos de `Catalogos_Picos/`)
```
ID_MUESTRA,ID_FEATURE,RELACION_MASA_CARGA,TIEMPO_RETENCION_MINUTOS,ALTURA_PICO,ID_METABOLITO,ID_NIVEL
1,,367.10,5.23,845210,2,1
2,,369.12,5.24,612300,,4
```

---

## Siguiente paso (cuando haya datos reales cargados)

Con estas 30 tablas llenas, el pivot a matriz "ancha" lista para modelar
(muestras × metabolitos, combinando POS+NEG por `ID_MUESTRA_FISICA`, con la
taxonomía traída de `01_especies.csv`, más el target de actividad biológica)
requiere hacer join con cada catálogo para volver a tener los nombres
legibles antes de presentarlo — el script de construcción se encarga de eso,
el dataset normalizado no está pensado para leerse "a ojo" fila por fila. Ya
quedó esbozado en la conversación y lo puedo convertir en un `.py` real
dentro de esta carpeta cuando tengas los primeros datos consolidados.

---

## Dónde viven los datos reales

Esta carpeta (`Datos/Base/`) es el **molde vacío**: solo referencia de columnas, sin
filas. Los datos reales, ya llenados a partir de un artículo + sus batches de MZmine, se
guardan en `Datos/Dataset/Consolidados/` — a la fecha de esta reorganización esa carpeta
**todavía sigue con la estructura plana anterior y códigos con prefijo como ID** (sin las
subcarpetas `Catalogos_*`, sin la regla de ID entero), pendiente de aplicarle el mismo
reordenamiento cuando se decida hacerlo.

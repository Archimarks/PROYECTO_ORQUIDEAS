# Código

Todo el código vive en `Script/`, dividido en 4 procesos independientes que se ejecutan en orden, más `Comun/` para lo compartido entre ellos y `main.py` como punto de entrada único. Los 4 procesos comparten la misma taxonomía paradigma → tarea (ver [Modelos.md](Modelos.md)):

```text
Script/
├── Comun/               Config y utilidades compartidas por los 4 procesos
│   ├── config.py           Configuración global: rutas, semillas aleatorias, constantes
│   ├── io_utils.py           Guardar/cargar datos, modelos y resultados
│   ├── taxonomia.py           Tabla canónica paradigma/tarea: nombre humano (con tilde) ↔ nombre interno (snake_case, ver Convenciones)
│   ├── logging_config.py       Configuración centralizada del logger (ver Convenciones)
│   ├── gpu_perfiles.py            Perfiles de GPU preconfigurados por máquina (ver Convenciones)
│   ├── dispositivo.py               Resuelve y activa el perfil de GPU activo (ver Convenciones)
│   └── leaderboard.py                 Compara resultados entre modelos (ver Modelos.md)
├── main.py               Punto de entrada único: CLI que orquesta el pipeline completo
├── Limpieza/             Datos/Batch → Datos/Dataset/Consolidados (general, sin taxonomía) — pendiente de crear
├── Procesamiento/         Consolidados → Datos/Dataset/Por Entrenamiento (por paradigma → tarea) — pendiente de crear
├── Entrenamiento/           Entrena y evalúa cada modelo (por paradigma → tarea → modelo, ver Modelos.md)
└── Visualizacion/            Gráficas — EDA y de resultados (ver Visualizacion.md) — pendiente de crear
```

En disco, hoy solo existen `Comun/` (parcial), `main.py` (vacío) y `Entrenamiento/` (las 12 combinaciones de paradigma/tarea originales, con sus 6 archivos cada una). `Limpieza/`, `Procesamiento/` y `Visualizacion/` todavía no se han creado — lo de abajo es la estructura planeada.

## Limpieza — `Script/Limpieza/`

Primera etapa: toma `Datos/Batch/` (espectros crudos LC-MS/GC-MS) y produce `Datos/Dataset/Consolidados/` (ver [Datos.md](Datos.md)). Es general — no depende de un modelo concreto, el resultado lo reutiliza cualquier experimento futuro.

General, en la raíz (aplica a todos los experimentos):

```text
Limpieza/
├── carga_datos.py       Lee los archivos crudos de Datos/Batch/
├── imputacion.py          k-NN o valor mínimo para datos faltantes
├── transformacion.py        log10 + escalamiento (Pareto / Auto-scaling)
└── principal.py                Orquesta el flujo completo
```

Por paradigma → tarea, solo para ajustes que no aplican a las demás tareas (ej. codificar etiquetas solo tiene sentido en Clasificación):

```text
Limpieza/<paradigma>/<tarea>/
├── ajustes_especificos.py     Ajuste propio de esta tarea, si aplica
└── principal.py                  Limpieza general + ajustes_especificos (si los hay)
```

Pendiente de crear en disco (16 combinaciones de paradigma/tarea cuando se cree, ver [Modelos.md](Modelos.md) para la lista completa). Convención: no es por-modelo, es por-tarea — no depende de un modelo concreto.

## Procesamiento — `Script/Procesamiento/`

Segunda etapa: toma `Datos/Dataset/Consolidados/` y produce el dataset final en `Datos/Dataset/Por Entrenamiento/<paradigma>/<tarea>/`. Sí depende del paradigma/tarea, porque el feature engineering y el split cambian según qué se va a predecir (ej. clasificación necesita las etiquetas codificadas; clustering no necesita split train/val/test).

Por paradigma → tarea, directo (sin subcarpeta de plantilla):

```text
Procesamiento/<paradigma>/<tarea>/
├── seleccion_features.py     Qué metabolitos/variables entran al modelo
├── split.py                    train/val/test (si aplica)
└── principal.py                  Orquesta: Consolidados -> Dataset/Por Entrenamiento
```

Pendiente de crear en disco (16 combinaciones cuando se cree). Convención: si en algún momento hay varios modelos distintos dentro de la misma tarea (ej. dos arquitecturas para Clasificación), crear una subcarpeta por modelo dentro de la tarea en vez de mezclar su código.

> `Entrenamiento/` (tercera etapa) y `Visualizacion/` (cuarta etapa) se documentan en [Modelos.md](Modelos.md) y [Visualizacion.md](Visualizacion.md) respectivamente, junto con lo que producen.

## Convenciones a respetar

- Nombres de carpeta en español, con tilde donde corresponde gramaticalmente — **excepto** dentro de `Script/`, que va en snake_case sin tildes ni espacios (restricción dura de Python: rompe los `import`).
  - Al pasar un nombre a snake_case se mantienen todas las palabras (incluidas preposiciones como "de"/"en"), solo se quitan tildes/espacios/mayúsculas — nunca se elide una palabra a criterio libre. Ej.: "Reducción de dimensionalidad" → `reduccion_de_dimensionalidad`, "Basado en política" → `basado_en_politica`.
- El nombre de un modelo (y su paradigma/tarea) debe coincidir entre `Datos/`, `Script/Procesamiento/`, `Script/Entrenamiento/` y `Resultados/`, para poder rastrear de qué código y qué datos salió cada resultado.
  - Como esto no puede ser una coincidencia literal de string (`Script/` va sin tildes, el resto sí), la conversión nombre-humano ↔ snake_case vive en una única tabla canónica: `Script/Comun/taxonomia.py`. Cualquier código que necesite pasar de uno a otro (`main.py` resolviendo `--paradigma`/`--tarea`, `leaderboard.py` armando el CSV...) importa esa tabla — nunca reimplementa la conversión a mano.
- Código dividido en archivos pequeños por responsabilidad (`config.py`, `carga_datos.py`, `modelo.py`, `entrenamiento.py`, `evaluacion.py`...) — nunca un único script que hace todo.
- No se sobrescribe un entrenamiento anterior en `Resultados/`: cada repetición con otros parámetros es una carpeta nueva.
- **Punto de entrada único (`Script/main.py`):** un CLI en la raíz de `Script/` que corre el pipeline completo (`Limpieza` → `Procesamiento` → `Entrenamiento` → `Visualizacion`) sin tener que ejecutar cada `principal.py` a mano.
  - Recibe por argumentos (`argparse`) qué paradigma/tarea/modelo correr (ej. `--paradigma Supervisado --tarea Clasificación --modelo <nombre>`), qué etapa(s) (`--etapa {limpieza,procesamiento,entrenamiento,visualizacion,comparar,todas}`, por defecto `todas`) y qué perfil de GPU usar (`--perfil-gpu`, por defecto `auto`).
  - Es solo un orquestador: para cada etapa invoca, en orden, el `principal.py` correspondiente (`Script/Limpieza/`, `Script/Procesamiento/<paradigma>/<tarea>/`, `Script/Entrenamiento/<paradigma>/<tarea>/`, `Script/Visualizacion/<paradigma>/<tarea>/`) — la lógica de cada etapa sigue viviendo en su propia carpeta, `main.py` no la reimplementa. `comparar` es la excepción: no toma paradigma/tarea, corre `Script/Comun/leaderboard.py` sobre todos los resultados existentes (ver [Modelos.md](Modelos.md)).
  - Cada `principal.py` debe seguir siendo ejecutable por sí solo (para depurar una sola etapa); `main.py` es la forma recomendada de correr el flujo de punta a punta, no la única forma de correr una etapa suelta.
- **`logging` en vez de `print()`:** prohibido usar `print()` para trazar la ejecución (carga de datos, progreso de entrenamiento, métricas, errores, warnings). Todo el código usa el módulo `logging` de la librería estándar.
  - Configuración del logger centralizada en `Script/Comun/logging_config.py` y reutilizada por los `principal.py` de las 4 etapas — no se reconfigura `logging` en cada archivo.
  - Nivel `INFO` o más alto va a archivo siempre; `DEBUG` es opcional y solo a consola, activable con un flag de `main.py` (ej. `--verbose`).
  - Cada corrida de entrenamiento guarda copia del log en `Resultados/<Paradigma>/<Tarea>/<nombre-del-entrenamiento>/ejecucion.log`, junto a `Modelo/`, `Estadisticas/` y `Graficas/` (ver [Modelos.md](Modelos.md)) — así el log queda trazable al mismo resultado en vez de perderse en la consola.
- **Perfil de GPU por máquina (`Script/Comun/gpu_perfiles.py` + `dispositivo.py`):** el código corre en máquinas con hardware distinto (ej. equipo de escritorio con 2 GPU dedicadas, portátil con 1 GPU integrada) — la GPU a usar nunca se hardcodea en `modelo.py`/`entrenamiento.py`, cada máquina tiene un perfil preconfigurado una sola vez y el código lo carga solo.
  - `gpu_perfiles.py`: diccionario de perfiles con nombre corto (ej. `escritorio`, `portatil`, `cpu`) y, por cada uno, qué GPU(s) exponer (`CUDA_VISIBLE_DEVICES`) y la estrategia (`single_gpu`, `multi_gpu`, `cpu`); más un diccionario `hostname → perfil` para que cada equipo tenga el suyo ya asignado sin tocar código al cambiar de máquina.
  - `dispositivo.py` expone `configurar_gpu(perfil=None)`: usa el `perfil` explícito si se pasa (o el que venga de `--perfil-gpu` en `main.py` / variable de entorno `PERFIL_GPU`); si no se especifica ninguno (`auto`), detecta la máquina por `socket.gethostname()` y busca su perfil en `gpu_perfiles.py`; si el hostname no está registrado, cae a un perfil `cpu` seguro por defecto y deja un `warning` en el log — nunca asume en silencio una GPU que puede no existir.
  - Se llama una sola vez, desde `main.py`, **antes** de invocar cualquier `principal.py` y antes de que se importe el framework de ML (`torch`, `tensorflow`...) dentro de `modelo.py` — `CUDA_VISIBLE_DEVICES` solo tiene efecto si se fija antes de inicializar el framework, fijarlo después no hace nada.
  - Agregar una máquina nueva (ej. un tercer equipo) es solo sumar su hostname y su perfil en `gpu_perfiles.py`; nunca tocar `modelo.py`/`entrenamiento.py`.
- **Dependencias (`requirements.txt`, en la raíz del repositorio):** toda librería externa (framework de ML, `numpy`, `pandas`, `scikit-learn`...) se fija por versión ahí (`paquete==X.Y.Z`) — nunca instalar algo a mano sin dejarlo registrado.
  - Se instala con `pip install -r requirements.txt` antes de correr `Script/main.py`.
  - Cualquier librería nueva que necesite una etapa (`Limpieza`, `Procesamiento`, `Entrenamiento`, `Visualizacion`) se suma ahí en el mismo cambio que la introduce — nunca se asume que ya está instalada en la máquina de quien corre el código.

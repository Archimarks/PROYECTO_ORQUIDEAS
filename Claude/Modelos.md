# Modelos

## Propósito

Desarrollar un modelo predictivo mediante Machine Learning capaz de clasificar, analizar e interpretar perfiles metabólicos de muestras vegetales a partir de datos de espectrometría de masas (LC-MS / GC-MS).

### Objetivo general

Desarrollar un sistema de reconocimiento de patrones basado en Machine Learning y espectros LC-MS de extractos vegetales para la predicción de propiedades biológicas, con potencial de aplicación en la elaboración de biocosméticos por comunidades de mujeres rurales y firmantes de paz.

### Objetivos específicos

- **Base de datos y modelos IA:** recopilar datos espectrales (repositorios públicos + muestras locales) y evaluar modelos de aprendizaje supervisado y no supervisado (redes neuronales, Random Forest, etc.) para la predicción precisa de propiedades biológicas.
- **Análisis de plantas locales:** seleccionar especies de la región (con apoyo de botánicos del Herbario Enrique Forero y saberes de ASMUPROPAZ), caracterizarlas por RP-LCMS-QTOF y validar las predicciones del modelo.
- **Desarrollo del software:** programar una herramienta intuitiva (en Python) que consolide las predicciones e información técnica de las plantas analizadas.

> **La idea es probar diferentes modelos y tipos de entrenamiento** (supervisado, no supervisado, distintas arquitecturas) para ver cuáles predicen mejor cada propiedad biológica — por esto el repositorio está organizado por paradigma → tarea, de forma que cada experimento quede aislado y sea comparable con los demás.

## Paradigma → tarea

Mismo árbol en `Datos/Dataset/Por Entrenamiento/`, los 4 procesos de `Script/` y `Resultados/`:

```text
Supervisado/          Clasificación · Regresión
No supervisado/        Clustering · Reducción de dimensionalidad · Reglas de asociación
Semi-supervisado/       Clasificación · Regresión
Auto-supervisado/        Contrastivo · Generativo
Por refuerzo/              Basado en valor · Basado en política · Actor-Crítico
Ensemble/                    Bagging · Boosting · Apilamiento · Votación-promedio
```

Si dentro de una tarea hay varios modelos distintos, se crea una subcarpeta por modelo dentro de esa tarea (en `Procesamiento/`, `Entrenamiento/` y `Resultados/`).

## Entrenamiento — `Script/Entrenamiento/`

Tercera etapa del pipeline de código: entrena y evalúa cada modelo con el dataset que dejó `Script/Procesamiento/` (ver [Codigo.md](Codigo.md) y [Datos.md](Datos.md)).

Por paradigma → tarea, directo (sin subcarpeta de plantilla):

```text
Entrenamiento/<paradigma>/<tarea>/
├── config.py            Hiperparámetros propios de este modelo
├── carga_datos.py         Carga desde Datos/Dataset/Por Entrenamiento/.../
├── modelo.py                Definición de la arquitectura/modelo
├── entrenamiento.py           Bucle/lógica de entrenamiento
├── evaluacion.py                 Métricas → Resultados/.../Estadisticas/ (incluye metricas.json, ver abajo)
└── principal.py                    Orquesta: carga_datos → entrenamiento → evaluacion
```

No todos los archivos son obligatorios en todo modelo, pero cada uno que exista debe tener una sola responsabilidad clara.

## Resultados — `Resultados/`

Un resultado por entrenamiento, mismo árbol paradigma → tarea:

```text
Resultados/<Paradigma>/<Tarea>/<nombre-del-entrenamiento>/
├── Modelo/               El modelo entrenado (pesos/checkpoint)
├── Estadisticas/          Estadísticas y métricas
├── Graficas/              Gráficas (ver Visualizacion.md)
└── ejecucion.log          Log de la corrida (ver Codigo.md — Convenciones, logging)
```

Cada corrida (entrenamiento) va a una carpeta nueva. Nunca se sobrescribe un entrenamiento anterior — repetir con otros hiperparámetros es una carpeta nueva (convención de nombre: `<nombre-del-modelo>_<fecha-o-version>`).

## Comparar resultados entre modelos (leaderboard)

El objetivo del proyecto es justamente decidir cuál modelo predice mejor cada propiedad biológica (ver Propósito, arriba) — pero cada `Estadisticas/` es libre en formato, así que por sí sola no hay forma sistemática de comparar entre paradigmas y tareas. Dos piezas resuelven esto:

### Esquema común: `Estadisticas/metricas.json`

Además de las métricas propias de la tarea, todo `evaluacion.py` (ver [Codigo.md](Codigo.md)) escribe un `metricas.json` con estos campos siempre presentes (nombres en snake_case, sin tildes — es un archivo que se parsea, mismo criterio que `Script/`):

```json
{
  "paradigma": "Supervisado",
  "tarea": "Clasificación",
  "modelo": "random_forest",
  "entrenamiento": "random_forest_2026-08-18",
  "propiedad_biologica": "antioxidante",
  "fecha": "2026-08-18",
  "metrica_principal": {"nombre": "f1_macro", "valor": 0.87},
  "metricas": {"accuracy": 0.90, "f1_macro": 0.87, "roc_auc": 0.93},
  "n_muestras": 120
}
```

- **`propiedad_biologica`** es el campo clave: el valor Y que se está prediciendo (`antioxidante`, `antiinflamatorio`, `cicatrizante`, `hidratante`...) — es lo que permite agrupar resultados de paradigmas y tareas distintos que atacan la misma pregunta.
- **`metrica_principal`** es una sola métrica, normalizada a [0, 1] donde mayor = mejor, para poder ordenar tareas de tipos distintos en una sola tabla:
  - Clasificación → F1 macro (ya en [0, 1]).
  - Regresión → R².
  - Clustering → Silhouette reescalado: `(silhouette + 1) / 2`.
  - Reducción de dimensionalidad → varianza explicada acumulada.
  - Por refuerzo → retorno normalizado (recompensa obtenida / recompensa máxima teórica).
  - Ensemble → la métrica principal de la tarea que está combinando (ej. un Ensemble de Clasificación reporta F1 macro).
  - Reglas de asociación → se define al implementar esa tarea (ej. confidence o lift promedio de las reglas top-N, normalizado); no tiene un candidato obvio como las demás.
  - Comparar F1 de Clasificación contra R² de Regresión sigue siendo una aproximación — para una comparación rigurosa, comparar primero dentro de la misma tarea.
- `metricas` queda libre para el resto de métricas propias de la tarea — no se estandariza más allá de `metrica_principal`.

### Agregador: `Script/Comun/leaderboard.py`

Recorre todos los `Resultados/**/Estadisticas/metricas.json`, arma una tabla (paradigma, tarea, modelo, entrenamiento, propiedad_biologica, métrica principal, fecha) y la guarda en `Resultados/leaderboard.csv`, ordenada por `propiedad_biologica` y luego por `metrica_principal` descendente. Se corre con `Script/main.py --etapa comparar` (ver [Codigo.md](Codigo.md)) — es la forma de responder "¿qué modelo predice mejor la propiedad X?" sin abrir carpeta por carpeta.

---

Recordatorio del objetivo del proyecto: la idea es probar varios de estos modelos y tipos de entrenamiento para ver cuál predice mejor cada propiedad biológica de los extractos vegetales.

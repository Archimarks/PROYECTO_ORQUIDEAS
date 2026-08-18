# Visualización

Cuarta etapa del pipeline de código (`Script/Visualizacion/`, ver [Codigo.md](Codigo.md)): genera las gráficas, tanto exploratorias (EDA, antes de entrenar) como de resultados (después de entrenar).

## Estructura de archivos

Lo genuinamente compartido vive en la raíz; las gráficas específicas de una tarea van por paradigma/tarea (una matriz de confusión no aplica a clustering, un dendrograma no aplica a clasificación):

```text
Visualizacion/
├── estilos.py                              Colores, formato y guardado — el "look" de todas las gráficas
└── <paradigma>/<tarea>/
    ├── graficas_eda.py                       Distribuciones, correlaciones, PCA exploratorio...
    └── graficas_resultados.py                  Curvas de entrenamiento, matriz de confusión, importancia de features...
```

Pendiente de crear en disco (16 combinaciones de paradigma/tarea cuando se cree, ver [Modelos.md](Modelos.md) para la lista completa) — hoy en `Script/` solo existe `Entrenamiento/`.

## Convención

No es por-modelo, es por-tarea; no depende de un modelo concreto. La llaman tanto `Limpieza/`/`Procesamiento/` (para EDA) como `Entrenamiento/` (para las gráficas que terminan en `Resultados/.../Graficas/`, ver [Modelos.md](Modelos.md)).

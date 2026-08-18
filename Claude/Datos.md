# Datos

Pipeline de datos en 3 carpetas, en el orden en que se generan:

```text
Datos/
├── Batch/                       Datos crudos, tal cual descargados (espectros LC-MS/GC-MS)
└── Dataset/
    ├── Consolidados/            Datos limpios/unificados, de uso general
    └── Por Entrenamiento/       Dataset final, por paradigma → tarea → modelo
```

El código que produce cada paso vive en `Script/Limpieza/` y `Script/Procesamiento/` (ver [Codigo.md](Codigo.md)).

## Qué son los datos: matriz untargeted de metabolómica

- **Variables de entrada / features (X):** abundancia relativa (altura de pico) de compuestos químicos — flavonoides y ácidos orgánicos (ej. ácido clorogénico, rutina, ácido quínico, keracianina).
- **Metadata y factores de estudio (Y):**
  - 
  - 
  - 
  - Control de calidad (QC): muestras agrupadas (Pooled QC) y orden de inyección (`Injection_order`), para corregir desviaciones instrumentales.

## Pipeline de preprocesamiento planeado

1. Separar la matriz en archivos de metadatos, cuantificación y anotación de metabolitos.
2. Imputar valores faltantes (ej. k-NN o valor mínimo).
3. Transformación logarítmica (log₁₀) y escalamiento (Pareto / Auto-scaling) para normalizar intensidades.

`Datos/Batch/` (espectros crudos LC-MS/GC-MS) → `Datos/Dataset/Consolidados/` (matriz separada, imputada, transformada y escalada) → `Datos/Dataset/Por Entrenamiento/<paradigma>/<tarea>/` (dataset final por modelo/experimento).

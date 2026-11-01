# Ojo en órbita

Proyecto con EuroSAT RGB para la asignatura Algoritmos y Arquitecturas para Procesado de Imágenes.
Estudiamos si un algoritmo puede descubrir usos del suelo a partir de imágenes de satélite.

## Cuadernos

- `Ojo_en_orbita.ipynb`: cuaderno principal para la entrega.
- `01_preparacion_eurosat.ipynb`: preparación del subconjunto mediante filtrado con dHash
  y muestreo de 800 imágenes por clase.
- `EDA_EUROSAT.ipynb`: exploración inicial del dataset completo y momentos de color.

## Entorno

Desde la raíz del proyecto:

```bash
uv sync --locked
```

Ejecuta el notebook desde la raíz del proyecto, junto a la carpeta `data/`.
En Google Colab, ejecuta las celdas desde el principio; no es necesario montar Drive.

## Subconjunto del cuaderno principal

El archivo `EuroSAT_RGB_800.zip` se puede descargar desde
[OneDrive](https://upm365-my.sharepoint.com/:u:/g/personal/anton_pogromskyy_alumnos_upm_es/IQB3xUtUzff7SbiYjMNg3unqAT_LrhuWcnpjl_yJclWIbhk?e=zg6BgA&download=1).
Contiene **8.000 imágenes RGB de 64 × 64 píxeles**, 800 por cada una de las diez clases,
y ocupa **26,79 MiB**.

## Datos y rutas del EDA inicial

El EDA utiliza **EuroSAT RGB**: 27.000 imágenes JPG de 64 × 64 píxeles, con
3 canales y 10 clases.

La estructura compartida es:

```text
data/
├── EuroSAT_RGB/
│   ├── AnnualCrop/
│   ├── Forest/
│   └── ... (una carpeta por clase, con sus JPG)
├── EuroSAT_MS/
└── processed/
    └── eurosat_spectral_features.csv
```

El notebook lee directamente `data/EuroSAT_RGB/` con Pillow y NumPy, sin
PyTorch ni descargas automáticas. Cada imagen es un array RGB `uint8` de forma
`(64, 64, 3)`, con valores entre 0 y 255. Las etiquetas numéricas se asignan a
las clases en orden alfabético. Las imágenes se abren conforme se necesitan,
sin cargar todo el dataset en memoria. Para el análisis del brillo, la media
se divide entre 255 para mantener los umbrales en [0, 1].

| Clase                | Imágenes en la descarga local RGB |
| -------------------- | --------------------------------: |
| AnnualCrop           |                             3.000 |
| Forest               |                             3.000 |
| HerbaceousVegetation |                             3.000 |
| Highway              |                             2.500 |
| Industrial           |                             2.500 |
| Pasture              |                             2.000 |
| PermanentCrop        |                             2.500 |
| Residential          |                             3.000 |
| River                |                             2.500 |
| SeaLake              |                             3.000 |
| **Total**            |                        **27.000** |

El CSV contiene 36 características calculadas a partir de RGB, HSV y Lab, más
dos columnas de etiquetas (38 columnas en total). Se guarda siempre en
`data/processed/`. Esta carpeta se crea automáticamente al guardar el CSV.

Fuente: [repositorio oficial de EuroSAT](https://github.com/phelber/EuroSAT).

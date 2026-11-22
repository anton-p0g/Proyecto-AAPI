# Proyecto 3: Ojo en órbita

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/anton-p0g/Proyecto-AAPI/blob/main/Ojo_en_orbita.ipynb)

**Grupo C**

- Anton Pogromskyy Mozharskyi (anton.pogromskyy@alumnos.upm.es)
- Diego Mateo Fernández (diego.mateofer@alumnos.upm.es)

Proyecto con EuroSAT RGB para la asignatura Algoritmos y Arquitecturas para Procesado de Imágenes.
Estudiamos si un algoritmo puede descubrir usos del suelo a partir de imágenes de satélite.

## El reto

¿Puede un algoritmo descubrir los usos del suelo a partir de imágenes de satélite? Para responder, comparamos distintas formas de representar cada imagen y estudiamos qué grupos aparecen cuando aplicamos k-means sin usar las etiquetas. Después usamos kNN como referencia supervisada para comprobar hasta qué punto esas mismas representaciones permiten reconocer las clases cuando disponemos de ejemplos etiquetados.

## Dataset y preparación

Trabajamos con **EuroSAT**, un conjunto de 27.000 imágenes de Sentinel-2 que recoge diez clases de uso y cobertura del suelo. Para el proyecto seleccionamos **8.000 parches RGB de 64 × 64 píxeles, 800 por clase**.

Empezamos con las imágenes multiespectrales porque su georreferenciación nos permite localizar cada parche. Dentro de cada clase, filtramos imágenes con huellas visuales muy parecidas mediante dHash y seleccionamos los parches por cuotas en una cuadrícula de 200 km para conservar, en lo posible, su distribución geográfica. Después recuperamos los JPG de los mismos parches para el cuaderno principal.

- [Descargar el subconjunto RGB desde OneDrive](https://upm365-my.sharepoint.com/:u:/g/personal/anton_pogromskyy_alumnos_upm_es/IQCVHas6nvyzQ5q_0426Mu43AUznmqcDQGyoeKBE_wm5yg8?e=rYR1dK&download=1) (`EuroSAT_RGB_800.zip`, **26,6 MiB**).
- [Descargar las características precalculadas desde OneDrive](https://upm365-my.sharepoint.com/:u:/g/personal/anton_pogromskyy_alumnos_upm_es/IQCNndBnJCd1RaWCgisxMQwTAfoenZTg5pFc5_1RbkJLixM?e=THLtSs&download=1) (**51,5 MiB**).

El conjunto original se distribuye con licencia MIT. [Repositorio oficial de EuroSAT](https://github.com/phelber/EuroSAT) | [Artículo de Helber y colaboradores](https://doi.org/10.1109/JSTARS.2019.2918242)

## Cómo lo hicimos

### A. Extracción de características

Para describir el color calculamos la media, la desviación típica, la asimetría y la curtosis de los canales en RGB, HSV y Lab. Para representar la textura usamos **Haralick**, que resume cómo se relacionan las intensidades de píxeles próximos, y **LBP**, que cuenta patrones locales. Evaluamos estos tres descriptores por separado y también combinamos color y textura en un único vector. Además, extraemos los embeddings de imagen de **ResNet50** y **CLIP ViT-B/32** con sus pesos preentrenados.

### B. Anotación semántica con CLIP

Usamos CLIP en modo _zero-shot_. Comparamos el vector de cada imagen con los vectores de diez frases en inglés y asignamos la clase de la frase más parecida. Partimos de la plantilla `a satellite photo of ...` y probamos tanto nombres de clases como descripciones de lo que se ve desde arriba. Después revisamos las matrices de confusión y comentamos cinco fallos para entender qué descripciones ayudaban y qué clases seguían confundiéndose.

| Versión de los prompts        |   Exactitud |
| ----------------------------- | ----------: |
| V1 con el nombre de la clase  |     41,69 % |
| V2 con descripciones visuales |     45,62 % |
| V2 revisada                   | **52,60 %** |

### C. Agrupamiento y búsqueda con FAISS

Aplicamos **k-means** con diez grupos a cada representación. Usamos la **silueta** para medir la separación relativa de los grupos y el **ARI** para comprobar cuánto coinciden con las clases oficiales, ajustando por azar. Con los embeddings de ResNet50 dibujamos un mapa t-SNE y lo coloreamos por clase y por grupo para localizar las mezclas.

Para la referencia supervisada, separamos **6.400 parches de referencia y 1.600 de consulta**, manteniendo la proporción de cada clase. Predecimos por votación entre los cinco vecinos más cercanos y comparamos dos índices de FAISS. `IndexFlatL2` busca de forma exhaustiva e `IndexIVFFlat` reparte los vectores en 64 celdas y consulta las cuatro más próximas.

| Descriptor        |   Silueta |       ARI | Exactitud kNN exhaustivo | Exactitud kNN con IVF |
| ----------------- | --------: | --------: | -----------------------: | --------------------: |
| Momentos de color |     0,170 |     0,204 |                  74,19 % |               73,56 % |
| Haralick          | **0,328** |     0,276 |                  68,19 % |               68,19 % |
| LBP               |     0,194 |     0,327 |                  69,31 % |               68,56 % |
| Color + textura   |     0,167 |     0,340 |                  85,13 % |               84,75 % |
| ResNet50          |     0,104 | **0,536** |              **90,69 %** |           **90,63 %** |
| CLIP              |     0,115 |     0,450 |                  88,13 % |               87,13 % |

En nuestra ejecución en **Google Colab**, IVF redujo el tiempo medio por consulta de **10,675 a 0,860 ms** con ResNet50 y de **1,520 a 0,181 ms** con CLIP. Esto supone aceleraciones de **12,41×** y **8,39×**, respectivamente. Los tiempos se midieron con consultas individuales y pueden variar según el entorno.

## Qué descubrimos

ResNet50 obtiene los grupos más parecidos a las clases oficiales y la mayor exactitud con kNN, aunque las clases solo se recuperan parcialmente mediante agrupamiento. Por ejemplo, un grupo de ResNet50 reúne **561 parches de River y 472 de Highway**. Al revisar las imágenes, vimos que ambos tipos de parche suelen mostrar una franja alargada que atraviesa el paisaje. También encontramos un grupo que mezcla **487 parches de Forest y 224 de SeaLake**, con ejemplos en los que predominan superficies visualmente uniformes. Estas mezclas muestran que un rasgo visual compartido puede unir parches que EuroSAT clasifica por separado.

## Cómo reproducir los resultados

### En Google Colab

1. Abrimos el cuaderno principal con el botón **Open in Colab** y ejecutamos la celda inicial, que instala las dependencias de [`requirements.txt`](requirements.txt).
2. El cuaderno descarga y descomprime los parches en `data/` y recupera las características precalculadas en `results/`. Si alguna característica sigue faltando, el apartado A la calcula y la guarda. El cuaderno del repositorio conserva las tablas y figuras de la ejecución de referencia.

### En local

Desde la raíz del repositorio instalamos el entorno con `uv` y Python 3.13.

```bash
uv sync --locked
```

## Cuadernos del repositorio

| Cuaderno                                                       | Contenido                                                                    |
| -------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| [`Ojo_en_orbita.ipynb`](Ojo_en_orbita.ipynb)                   | Análisis principal con los apartados A, B y C y sus resultados.              |
| [`01_preparacion_eurosat.ipynb`](01_preparacion_eurosat.ipynb) | Filtrado visual, muestreo geográfico y exportación del subconjunto RGB.      |
| [`EDA_EUROSAT.ipynb`](EDA_EUROSAT.ipynb)                       | Exploración del conjunto RGB y de sus espacios de color.                     |
| [`EDA_Multiespectral.ipynb`](EDA_Multiespectral.ipynb)         | Exploración de las bandas de Sentinel-2, incluido el infrarrojo cercano B08. |

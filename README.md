# Modelo predictivo de precipitaciones en Cajamarca

Proyecto académico orientado a la **clasificación del nivel de riesgo ante precipitaciones en los distritos de la provincia de Cajamarca**, mediante técnicas de Machine Learning.

El cuaderno `Cajamarca_E1_E4_Colab.ipynb` implementa un prototipo que compara cuatro clasificadores para anticipar la banda mensual de un índice construido con precipitación y susceptibilidad territorial. Los resultados se presentan en un mapa interactivo de los 12 distritos.

**La salida del prototipo corresponde a bandas estimadas E1–E4 del índice, no a categorías oficiales de riesgo.** El modelo clasifica estas bandas; no estima directamente la cantidad de lluvia del mes objetivo.

## Contenido

- [Objetivo](#objetivo)
- [Datos y variables](#datos-y-variables)
- [Índice y clases de referencia](#índice-y-clases-de-referencia)
- [Metodología](#metodología)
- [Tecnologías](#tecnologías)
- [Ejecución en Google Colab](#ejecución-en-google-colab)
- [Consulta y visualización](#consulta-y-visualización)
- [Resultados y archivos exportados](#resultados-y-archivos-exportados)
- [Fuentes de información](#fuentes-de-información)

## Objetivo

Desarrollar un modelo predictivo para la clasificación del nivel de riesgo ante precipitaciones en los distritos de la provincia de Cajamarca.

El prototipo aborda este objetivo mediante la comparación de clasificadores sobre bandas estimadas de un índice de precipitación y susceptibilidad, su evaluación temporal y la representación geográfica de sus predicciones.

## Datos y variables

La unidad de análisis es el **distrito-mes**. El cuaderno utiliza un panel con 6.480 combinaciones y un conjunto de modelado de 6.408 observaciones, después de excluir los seis meses iniciales de cada distrito para disponer de antecedentes completos de precipitación.

El calendario de trabajo del cuaderno abarca de enero de 1981 a diciembre de 2025. El conjunto de modelado comienza en julio de 1981.

### Variables de entrada

Para clasificar el mes objetivo **t**, se utilizan cinco entradas:

| Variable | Descripción |
| --- | --- |
| `mes` | Mes calendario del periodo objetivo, de 1 a 12. |
| `susceptibilidad_mm_cenepred` | Valor fijo de susceptibilidad territorial asignado al distrito. |
| `precip_mes_previo_mm` | Precipitación del mes anterior, t−1, en milímetros. |
| `precip_acum_3meses_previos_mm` | Suma de precipitación desde t−3 hasta t−1, en milímetros. |
| `precip_acum_6meses_previos_mm` | Suma de precipitación desde t−6 hasta t−1, en milímetros. |

La variable objetivo es `banda_proxy_estimada`, con las clases **E1, E2, E3 y E4**. La precipitación del mes t se utiliza para construir la etiqueta de referencia, pero no se incorpora como entrada al clasificador.

Los identificadores UBIGEO se conservan como texto para mantener sus ceros iniciales. Aunque los archivos contienen información adicional para auditoría, únicamente las cinco variables anteriores se incorporan al estimador.

## Índice y clases de referencia

Las etiquetas se calculan mediante la siguiente regla, definida en el cuaderno:

```text
I = (precipitación del mes t / 191,47574105)
    × (susceptibilidad del distrito / 0,5003)
```

El normalizador de precipitación corresponde al percentil 95 de las 3.168 observaciones del periodo de referencia utilizado por el cuaderno: enero de 1981 a diciembre de 2002.

| Banda | Intervalo del índice |
| --- | --- |
| E1 | 0 ≤ I < 0,25 |
| E2 | 0,25 ≤ I < 0,50 |
| E3 | 0,50 ≤ I < 0,75 |
| E4 | I ≥ 0,75 |

El índice no se redondea antes de asignar la banda ni se restringe a un máximo de uno. La evaluación compara las predicciones del modelo con estas etiquetas calculadas.

## Metodología

El cuaderno organiza el flujo de trabajo bajo el enfoque **CRISP-DM**:

1. **Comprensión del problema y los datos:** definición de la unidad distrito-mes, las entradas y el objetivo de clasificación.
2. **Preparación y auditoría:** verificación de archivos mediante SHA-256, revisión de claves, cobertura mensual, valores numéricos, antecedentes de lluvia y geometrías distritales; reproducción del índice y sus etiquetas.
3. **Exploración:** análisis de distribuciones y correlaciones utilizando exclusivamente entrenamiento.
4. **Modelado:** comparación de cuatro algoritmos con las mismas cinco entradas.
5. **Evaluación:** selección mediante validación y evaluación final del modelo elegido sobre prueba.
6. **Presentación y exportación:** consulta interactiva, mapas distritales y almacenamiento de evidencias de la ejecución.

### Algoritmos comparados

| Algoritmo | Implementación |
| --- | --- |
| Árbol de Decisión | Clasificador de referencia. |
| Random Forest | Ensamble de árboles de decisión. |
| XGBoost | Clasificador multiclase con árboles potenciados. |
| SVM | Máquina de vectores de soporte con kernel RBF y escalamiento mediante `StandardScaler`. |

Los modelos emplean pesos equilibrados recalculados en cada ajuste. El escalamiento de SVM se realiza dentro del pipeline de entrenamiento.

### Partición temporal

| Conjunto | Periodo de trabajo | Observaciones |
| --- | --- | ---: |
| Entrenamiento | Julio de 1981 a diciembre de 2017 | 5.256 |
| Validación | Enero de 2018 a diciembre de 2021 | 576 |
| Prueba | Enero de 2022 a diciembre de 2025 | 576 |

Los 12 distritos de un mismo mes permanecen juntos en cada partición. Dentro del entrenamiento se utiliza validación temporal con ventana de entrenamiento creciente y tres bloques de evaluación: **2003–2007, 2008–2012 y 2013–2017**.

Se comparan cuatro configuraciones por algoritmo y tres cortes temporales por configuración, para un total de 48 ajustes internos. La configuración de cada algoritmo se elige por F1-macro medio de validación cruzada. Después, el algoritmo final se selecciona por F1-macro en validación, con exactitud balanceada como siguiente criterio.

El modelo seleccionado conserva su ajuste realizado únicamente con entrenamiento. El conjunto de prueba se reserva para la evaluación final.

### Métricas e interpretación

- F1-macro.
- Exactitud balanceada y exactitud global.
- Precisión, sensibilidad y F1 por banda.
- Sensibilidad de la banda E4.
- Matriz de confusión.
- AUC multiclase One-vs-Rest cuando el modelo proporciona probabilidades compatibles.

El cuaderno comprueba las metas **F1-macro ≥ 0,70** y **sensibilidad de E4 ≥ 0,75** con los resultados de cada ejecución. También calcula la importancia por permutación en validación y un diagnóstico de sensibilidad de la regla del índice ante distintos cuantiles de normalización.

## Tecnologías

| Herramienta | Uso |
| --- | --- |
| Python | Procesamiento, modelado y evaluación. |
| Google Colab | Ejecución del cuaderno y consulta interactiva. |
| NumPy y pandas | Cálculo numérico y manejo de datos tabulares. |
| scikit-learn 1.7.2 | Pipelines, clasificadores, métricas e importancia por permutación. |
| XGBoost 3.1.1 | Clasificación mediante árboles potenciados. |
| Matplotlib 3.10.7 y Seaborn 0.13.2 | Gráficos exploratorios y de evaluación. |
| Folium 0.20.0 | Mapas interactivos con geometrías GeoJSON. |
| ipywidgets 8.1.7 | Controles de consulta histórica y escenarios manuales. |
| joblib 1.5.2 | Almacenamiento del modelo entrenado. |

El cuaderno instala las versiones declaradas de modelado y visualización, registra las versiones de NumPy y pandas del entorno y utiliza la semilla `42`.

## Ejecución en Google Colab

### Archivos necesarios

- `Cajamarca_E1_E4_Colab.ipynb`.
- `datos_cajamarca.zip`, correspondiente a la versión de datos esperada por el cuaderno.

El ZIP debe contener estos archivos en su raíz:

```text
datos_cajamarca.zip
├── entrenamiento_proxy_6408.csv
├── dataset_cajamarca_6480_etiquetas_estimadas.csv
├── columnas_para_modelo.json
├── LEEME_METODO.md
└── distritos_cajamarca_mascara_poligonal.geojson
```

### Pasos

1. Abre [Google Colab](https://colab.research.google.com/).
2. Selecciona **Archivo → Subir cuaderno** y carga `Cajamarca_E1_E4_Colab.ipynb`.
3. Utiliza un entorno de ejecución **CPU**.
4. Selecciona **Entorno de ejecución → Ejecutar todas**.
5. Cuando aparezca el selector de archivos, carga `datos_cajamarca.zip` **sin descomprimir**.
6. Mantén la sesión activa durante el entrenamiento, la consulta y la exportación.
7. Descarga `resultados_cajamarca.zip` cuando finalice la ejecución.

El cuaderno comprueba las huellas SHA-256 de los archivos antes de leerlos. Los controles interactivos requieren que las celdas correspondientes estén ejecutadas y la sesión continúe conectada. El mapa base utiliza conexión a Internet.

## Consulta y visualización

La interfaz contiene dos modos:

### Histórico de prueba

Permite seleccionar un mes del conjunto de prueba y consultar las predicciones de los 12 distritos, junto con sus bandas de referencia calculadas. También permite filtrar el detalle tabular de un distrito.

### Escenario manual

Permite seleccionar un distrito, el año y mes objetivo e ingresar las precipitaciones de los seis meses anteriores. El cuaderno construye los antecedentes acumulados y utiliza la susceptibilidad fija del distrito para predecir una banda.

El resultado se representa en un **mapa coroplético distrital**: cada polígono se colorea según su banda estimada. En el escenario manual se colorea únicamente el distrito consultado.

| Banda | Color |
| --- | --- |
| E1 | Verde |
| E2 | Amarillo |
| E3 | Naranja |
| E4 | Rojo |
| Sin resultado en el escenario | Gris |

## Resultados y archivos exportados

Las métricas, el algoritmo seleccionado y las predicciones se calculan al ejecutar el cuaderno. El flujo de exportación reúne los siguientes productos en `resultados_cajamarca.zip`:

| Producto | Archivos principales |
| --- | --- |
| Modelo entrenado | `modelo_final.joblib` |
| Comparación de modelos | `comparacion_cv.csv`, `resultados_pliegues.csv`, `comparacion_validacion.csv` |
| Evaluación final | `metricas_prueba.json`, `metricas_por_banda.csv`, `matriz_confusion.csv`, `matriz_confusion.png` |
| Predicciones | `predicciones_prueba.csv` |
| Interpretación | `importancias_validacion.csv`, `importancias_validacion.png` |
| Diagnóstico del índice | `sensibilidad_regla.csv` |
| Exploración | `exploracion_entrenamiento.png`, `correlaciones_entrenamiento.png` |
| Mapas mensuales | `mapa_AAAA-MM.html`, uno por cada mes de prueba |
| Trazabilidad | `auditoria.json`, `protocolo.json`, `versiones.json`, `huellas_fuentes.json` |
| Entorno y guía de resultados | `requirements_ejecucion.txt`, `LEEME_RESULTADOS.txt` |

Los mapas HTML exportados pueden abrirse fuera de Colab. La consulta manual se realiza desde el cuaderno con Python activo.

## Fuentes de información

Las fuentes utilizadas para el proyecto incluyen el recurso publicado en Figshare[^1], el conjunto de emergencias y daños del Portal Nacional de Datos Abiertos[^2], los escenarios de SIGRID/CENEPRED[^3], el visor de ArcGIS Experience Builder[^4] y el servicio de peligros geológicos de GEOCATMIN/INGEMMET[^5].

El estimador del cuaderno utiliza exclusivamente las cinco entradas descritas en [Datos y variables](#datos-y-variables); no incorpora población, pobreza ni registros de daños de INDECI como variables predictoras.

[^1]: Figshare. Recurso de datos del proyecto. https://doi.org/10.6084/m9.figshare.32411886
[^2]: Portal Nacional de Datos Abiertos. Emergencias y daños a nivel nacional por departamento. https://www.datosabiertos.gob.pe/dataset/emergencias-y-daños-nivel-nacional-por-departamento
[^3]: SIGRID/CENEPRED. Escenarios de riesgo. https://sigrid.cenepred.gob.pe/sigridv3/escenarios
[^4]: ArcGIS Experience Builder. Visor geográfico utilizado en el proyecto. https://experience.arcgis.com/experience/a7e51d8de498487b9e58ca31582c5e09#data_s=id%3AdataSource_6-1a159aa4312-layer-47%3A1207%2Cid%3Awidget_453_output%3A0
[^5]: GEOCATMIN/INGEMMET. Servicio de peligros geológicos. https://geocatmin.ingemmet.gob.pe/arcgis/rest/services/SERV_PELIGROS_GEOLOGICOS/MapServer

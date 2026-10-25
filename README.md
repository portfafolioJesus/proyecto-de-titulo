# Stock Eye

Sistema de Visión Artificial integrado a una **aplicación móvil** para **detectar y cuantificar espacios vacíos en góndolas de supermercado** a partir de imágenes fijas, generando alertas automáticas de reposición cuando el nivel de stock cae bajo el umbral definido por Walmart. El objetivo es reducir las pérdidas de venta asociadas a quiebres de stock no detectados a tiempo.

Proyecto Capstone desarrollado en conjunto con **Walmart Chile** — Padre Alonso Ovalle.

**Equipo:** Jesús González · Claudio Galiano · Ignacio Ruz Aguilar

---

## Prioridad del proyecto

> El objetivo principal **no** es desarrollar una interfaz gráfica sofisticada para la captura de imágenes, sino resolver adecuadamente la lógica que permite analizar la imagen y determinar los espacios vacíos en las góndolas.

En esta etapa se prioriza, en este orden:

1. La captura de una imagen que pueda ser procesada.
2. El procesamiento y análisis de dicha imagen.
3. La identificación de los espacios vacíos.
4. La determinación del porcentaje de espacio vacío.
5. La utilización de este resultado para identificar posibles quiebres de stock.

La lógica y los algoritmos de detección de espacios vacíos y quiebre de stock tienen prioridad sobre la capa gráfica o la interfaz de captura.

---

## Arquitectura

> **Nota:** la arquitectura cambió respecto a la propuesta de Fase 1 (plataforma web con Docker, FastAPI, MySQL y Nginx). Se mantienen **Docker** y la **API**, pero el cliente pasa de ser una web a una **app móvil**, y las imágenes llegan desde un **PC de la tienda** con OneDrive. El diagrama de Fase 1 quedó desactualizado y hay que rehacerlo.

```
Cámaras de seguridad
        │  guardan imágenes fijas
        ▼
PC de la tienda ── carpeta sincronizada con OneDrive
        │
        ▼
Script 1 · Listener (Python, 24/7)
        │  detecta imagen nueva
        ▼
Script 2 · Inferencia (Python + pesos .pt)
        │  espacios vacíos, % vacío por repisa, estado del stock
        ▼
        │  envía inferencia y alarma (HTTP)
        ▼
API (FastAPI, en Docker)
        │  registra imagen, fecha y hora, resultado
        │
        ├──► App móvil: estado de las góndolas
        └──► Notificación push al reponedor si el stock < 30%
```

### 1. Captura de imágenes (sucursal)

- Las **cámaras de seguridad** ya instaladas en la sala de venta guardan imágenes fijas (no video) en un **PC de la tienda**.
- Ese PC tiene la carpeta de capturas sincronizada con **OneDrive**, de modo que las imágenes quedan respaldadas y accesibles.

### 2. Procesamiento (PC de la tienda)

Dos scripts de Python, desacoplados (arquitectura modular, RF-14), empaquetados con **Docker**:

| Script | Rol |
|---|---|
| **Listener** | Corre **24/7** y vigila la carpeta de capturas. Cuando se sube una imagen nueva, se la entrega al script de inferencia. |
| **Inferencia** | Carga el archivo de pesos **`.pt`** (YOLO26-seg) y detecta los espacios vacíos. Calcula el **% vacío por repisa y por góndola**, decide el estado del stock según el semáforo y **envía la inferencia y la alarma a la API**. |

La lógica del segundo script ya existe como prototipo en el notebook `StockEye_Demo.ipynb` (celdas 7.0, 8.1 y 8.2). Falta migrarla a un `.py`, escribir el listener y conectar el envío a la API.

### 3. API, aplicación móvil y notificaciones

- La **API** (propuesta: **FastAPI**, empaquetada en **Docker**) recibe la inferencia y la alarma desde el script de inferencia, las registra con trazabilidad (imagen, fecha y hora, resultado) y las expone a la app.
- La **app móvil** consulta la API y muestra el estado de las góndolas.
- Cuando el stock cae bajo el **30%** (semáforo rojo), la API dispara una **notificación push** (propuesta: **Firebase Cloud Messaging**) al reponedor de la zona.
- Roles definidos en Fase 1:

| Rol | Qué hace |
|---|---|
| Reponedor | Recibe alertas de su zona, las marca "en atención" y "resuelta" |
| Supervisor | Ve el estado de toda la tienda, prioriza y asigna alertas, configura umbrales y genera reportes |
| Administrador / Desarrollador | Mantiene el sistema, el modelo y el repositorio |

- *En definición:* dónde se despliega la API (en el PC de la tienda o en un servidor aparte), la base de datos para usuarios, roles e historial de alertas (MySQL fue la propuesta de Fase 1) y el mecanismo de autenticación de la app.

### 4. Entrenamiento del modelo (offline)

- El entrenamiento y la validación se hacen **offline** en **Google Colab** (GPU T4, restricción de hardware RNF-01).
- Al terminar se exporta el archivo de pesos `.pt`, que se copia al PC de la tienda para que lo use el script de inferencia.
- Con la **retroalimentación** del notebook (celdas 8.3 y 8.4) se corrigen predicciones y se reentrena de forma incremental.

El repositorio de **GitHub** es la fuente de verdad del código y del historial de cambios del proyecto.

---

## Tecnologías

> Las filas marcadas **(implementado)** ya existen en el notebook; el resto es propuesta para la etapa de la app móvil.

| Área | Tecnología |
|---|---|
| Lenguaje base (CRISP-DM, exploración, modelado) | **Python** |
| Prototipado / exploración de datos | **Google Colab** |
| Pipeline final de entrenamiento e inferencia | Hoy corre en un **notebook de Colab** (exploratorio, GPU T4). Se migrará a scripts **`.py`** estructurados, empaquetados en Docker |
| Detección de espacios vacíos **(implementado)** | **YOLO26-seg** (segmentación con polígonos). Etiquetado asistido con **SAM 2.1** y **Grounding DINO**; embeddings con **SigLIP**; repisas por geometría (mínimos cuadrados). Se comparó contra la línea base **v4** (cajas YOLO) |
| Procesamiento en la tienda | Dos scripts de **Python** en el PC de la tienda: un listener 24/7 y un script de inferencia que usa el `.pt` y envía el resultado a la API |
| Backend / API | **FastAPI** (Python): recibe la inferencia y la alarma, las registra, las expone a la app y dispara la notificación push |
| App móvil | **En definición**: se evalúa **Ionic + Angular** (TypeScript) para la app que muestra el estado de las góndolas |
| Base de datos | **MySQL** (propuesta de Fase 1) para usuarios, roles e historial de alertas; confirmar |
| Almacenamiento de imágenes | Carpeta del PC de la tienda sincronizada con **OneDrive** |
| Empaquetado / portabilidad | **Docker** (y `docker-compose` si hay más de un contenedor) para scripts y API, para reproducir el entorno en otro equipo (RNF-04) |
| Notificaciones push | **Firebase Cloud Messaging (FCM)** (propuesta) |
| Integración con cámaras | Las cámaras guardan imágenes en el PC de la tienda; **OneDrive** las sincroniza |
| Control de versiones | **GitHub** (repositorio público) |
| Diseño de software | Principios **SOLID**; patrones **Strategy** (intercambiar algoritmo de detección) y **Repository** (desacoplar acceso a datos) |
| Metodología de gestión de datos/ML | **CRISP-DM** |
| Metodología de gestión de proyecto | **Kanban** |

### Restricciones técnicas relevantes (RNF)

- Entrenamiento: máx. **4.0 GB de VRAM** (Google Colab / Kaggle GPU).
- Inferencia: máx. **1.5–2.0 GB de VRAM**.
- Tiempo de procesamiento por imagen: **2 a 5 segundos** (no se requiere tiempo real / streaming).
- El código debe ser reproducible de forma autónoma en el entorno de un ingeniero de Walmart, siguiendo solo la documentación entregada.
- Cualquier dataset externo debe contar con licencia que permita uso comercial/académico.

---

## Requerimientos funcionales clave

- Detección y delimitación (bounding box) de espacios vacíos en góndolas.
- Cálculo de porcentaje/volumen de espacio libre, segmentado por nivel/repisa.
- **Alerta de reposición cuando el stock/espacio disponible cae bajo 30%** (semáforo de Walmart: verde ≥ 70% · amarillo 30–70% · rojo < 30% = alerta; parámetros `stock_verde` y `stock_rojo` en la celda 8.1).
- Diferenciación entre vacío real (quiebre) y vacío no apilable (ej. frascos de vidrio).
- Robustez ante ruido visual real de sala de venta (~90% de las imágenes con ruido, según Walmart) y distorsión de cámara (fisheye).
- Procesamiento sobre imagen fija (no video en tiempo real).
- Registro trazable de cada análisis (imagen, timestamp, resultado).
- Arquitectura modular: pre-procesamiento / modelo / post-procesamiento, con entradas y salidas tipadas.

---

## Estado actual del repositorio

- **Fase 1 (definición):** documentación y evidencias en `Fase 1/`.
- **Fase 2:** carpeta `Fase 2/` con `Evidencias Individuales/` y `Evidencias Grupales/`.
- **Pipeline de visión (`StockEye_Demo.ipynb`):** implementado y evaluado en Google Colab (T4), Fases 0 a 8 del notebook. Las Fases 9 (búsqueda vectorial) y 10 (video) están solo planificadas.
- **Stubs** en `desarrollo/` (`front.tsx`, `back.tsx`, `css.css`), sin implementación. Corresponden a la propuesta web anterior.
- **Listener, script de inferencia en `.py`, API, Docker y app móvil con notificaciones push:** pendientes, se implementarán después.

### Resultados del modelo (set de validación)

| Métrica | v4 (línea base) | Modelo actual |
|---|---|---|
| Recall de vacíos | 0.27 | 0.57 (entrenamiento base) · **0.76 con P ≥ 0.6** (modelo ganador de la Fase 7, mosaicos + TTA) |
| mAP50-95 de vacío | 0.099 | 0.352 |
| F1 de detección de vacíos | 0.48 | 0.57 |
| Tiempo por foto | — | 2.12 s (cumple 2–5 s) |

---

## Cómo probar la demo

La demo corre en **Google Colab** con GPU T4 y usa Google Drive para los archivos pesados (dataset, pesos del modelo y resultados).

1. **Obtener los archivos.** Abre la carpeta compartida de Drive: <https://drive.google.com/drive/folders/1ozXtwQrtDGWj1IqZQBe4jQxkk17QxWuo?usp=sharing>
   - Descárgala completa (Drive la entrega comprimida en `.zip`) y descomprímela, o agrégala a tu Drive con *Organizar → Agregar acceso directo / Hacer una copia*.
   - Debe quedar en tu Drive como **`MyDrive/stockeye_v5/`**, con estos contenidos:
     - `StockEye_Demo.ipynb` (este notebook)
     - `dataset_gondolas_2026-10-13.zip`
     - `stockeye_v4_best.pt`
     - `runs_f6/` (pesos entrenados de `control_v3` y `maestro_v3`) y, si existe, `modelos/vigente.pt`
2. **Abrir el notebook en Colab** (clic derecho sobre `StockEye_Demo.ipynb` → *Abrir con → Google Colaboratory*).
3. **Activar la GPU:** *Entorno de ejecución → Cambiar tipo de entorno → T4 GPU*.
4. **Correr la celda `[RESTAURAR]`** (la primera celda de código). Monta Drive y ejecuta en orden las celdas 0.1 a 0.9, 1.2, 5.0 a 5.4, 6.1, 6.2, 6.7, 7.0 a 7.5, 8.1 y 8.2. Se detiene si falla alguna o si faltan los pesos de las celdas 6.3 y 6.4.
5. **Subir una foto de góndola** cuando la celda **8.2** lo pida. El notebook devuelve:
   - los vacíos como polígonos y el `% vacío` por repisa y por góndola;
   - el semáforo de stock (rojo = alerta bajo 30%);
   - el estado `indeterminado` cuando no hay evidencia suficiente.
   El resultado queda en `stockeye_v5/inferencias/`.

> **Advertencias:**
> - La celda `[RESTAURAR]` también ejecuta la 6.7 (ajuste fino) y la 7.5 (evaluación); pueden tardar o reentrenar si faltan los pesos intermedios. Es lo más pesado de la demo.
> - Para probar otra foto sin repetir todo, vuelve a correr solo la celda **8.2**. Para procesar una carpeta completa, escribe su ruta en `CARPETA_F8`.
> - Fuera de Colab el notebook corre en CPU, mucho más lento.

---

## Referencias

Documento base: *Presentación idea de Proyecto — Stock Eye* (Capstone, Padre Alonso Ovalle).

# Stock Eye

Sistema de Visión Artificial integrado a una plataforma web para **detectar y cuantificar espacios vacíos en góndolas de supermercado** a partir de imágenes fijas, generando alertas automáticas de reposición cuando el nivel de stock cae bajo el umbral definido por Walmart. El objetivo es reducir las pérdidas de venta asociadas a quiebres de stock no detectados a tiempo.

Proyecto Capstone desarrollado en conjunto con **Walmart Chile** — Padre Alonso Ovalle.

**Equipo:** Jesús González · Claudio Galiano · Ignacio Rolando Ruz Aguilar

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

> **Nota:** el siguiente diagrama es una **propuesta de arquitectura objetivo** trabajada por el equipo; a la fecha **no está implementada en este repositorio** (que hoy solo contiene la documentación de Fase 1 y stubs iniciales en `desarrollo/`). Se documenta aquí como referencia de diseño para las siguientes fases.

![Diagrama de arquitectura (propuesta)](https://drive.google.com/file/d/1gU6roqFDJe12fk0QCS46XLC7lxehZDXA/view?usp=sharing)

La arquitectura propuesta se organiza en cuatro grandes bloques: **captura de imágenes** en la sucursal, un **servidor Docker** que aloja todo el stack de procesamiento y la plataforma web, el **acceso de los usuarios** (reponedor, supervisor, administrador), y un **flujo de entrenamiento offline** del modelo de detección.

### 1. Captura de imágenes (sucursal)

- **Sistema de cámaras**: cada góndola/recinto toma 1 foto cada 10 minutos mediante un trigger programado.
- Las imágenes se envían hacia el servidor **vía API / Internet**; el mecanismo puntual de transporte todavía está en definición por el equipo (no se ata a un proveedor o suite específica).
- El contenedor **`captura-imagenes`** es el responsable de recibir/obtener las fotos nuevas y ponerlas a disposición del backend.

### 2. Servidor — Docker Host

Todo el backend corre dentro de una red Docker (`monitoreo_net`) orquestada por `docker-compose.yml`, lo que permite replicar el stack completo en otro servidor con `docker compose up -d`. Contenedores principales:

| Contenedor | Tecnología | Rol |
|---|---|---|
| `captura-imagenes` | Script backend (Python) | Recibe/obtiene las imágenes nuevas vía API e Internet y las deja disponibles para el backend |
| `backend-api` | **FastAPI** (REST API, puerto `:8000`) | Expone la API (`/api`), orquesta la lógica de negocio, solicita inferencia al modelo y gestiona alertas |
| `modelo-deteccion` | **Python — en evaluación (Computer Vision clásico / R-CNN / YOLO)** (servicio interno `:8500`) | Ejecuta la inferencia sobre las imágenes para detectar y cuantificar espacios vacíos |
| `frontend` | **Framework en definición** (ver sección Tecnologías) | Interfaz web de monitoreo para reponedores, supervisores y administradores |
| `nginx` | Reverse proxy (**HTTPS `:443`**) | Sirve el frontend y enruta las peticiones `/api` hacia el backend |
| `mysql` | **MySQL** | Base de datos relacional: usuarios, roles, historial de alertas |

Volúmenes Docker persistentes:

- **`mysql_data`**: persiste la base de datos MySQL.
- **`imagenes_capturas`**: almacena las imágenes descargadas; son leídas/escritas tanto por `captura-imagenes`/`backend-api` como por `modelo-deteccion`.
- **`modelo_artefacto`**: contiene el artefacto del modelo entrenado (`.pt` / `.onnx` / `.pkl`), que `modelo-deteccion` carga al iniciar.

El flujo interno típico es: `captura-imagenes` hace `POST` de la imagen al `backend-api` → `backend-api` solicita inferencia a `modelo-deteccion` → el resultado (espacios vacíos, % de vacío, alerta) se persiste en `mysql` y queda disponible para el `frontend` a través de `nginx`.

### 3. Acceso y notificaciones

- **Reponedor / Supervisor / Administrador** acceden vía navegador (HTTPS) a la plataforma web servida por `nginx`.
- El **teléfono del reponedor** registra su token de dispositivo al iniciar sesión en el móvil.
- Cuando el `backend-api` genera una alerta (quiebre de stock detectado), hace `POST` a un **servicio de push (Firebase Cloud Messaging)**, que entrega la notificación al teléfono del reponedor correspondiente.

### 4. Entrenamiento del modelo (offline)

- El entrenamiento y validación del modelo se realiza **offline**, en **Google Colab / Kaggle** (GPU limitada — restricción de hardware, RNF-01).
- Una vez entrenado y validado, se exporta el artefacto del modelo (`.pt` / `.onnx` / `.pkl`).
- Ese artefacto se carga al volumen `modelo_artefacto` del servidor, quedando disponible para el contenedor `modelo-deteccion` en cada nueva versión.

El repositorio de **GitHub** es la fuente de verdad del código y del historial de cambios del proyecto.

---

## Tecnologías (propuesta)

| Área | Tecnología |
|---|---|
| Lenguaje base (CRISP-DM, exploración, modelado) | **Python** |
| Prototipado / exploración de datos | **Google Colab** |
| Pipeline final de entrenamiento e inferencia | Scripts **`.py`** estructurados en IDE profesional (se evita Jupyter Notebook en los módulos core por incompatibilidad con drivers CUDA/GPU) |
| Detección de espacios vacíos | **En evaluación/experimentación**: Computer Vision clásico (bordes, contornos, transformaciones matriciales) vs. modelos de Deep Learning (**R-CNN** y **YOLO**). El CV clásico aparece en el marco teórico como línea de base a comparar, no como decisión cerrada; se definirá el enfoque final según los resultados de las pruebas |
| Backend / API | **FastAPI** (Python) — expone los resultados del modelo y gestiona la lógica de alertas |
| Frontend | **En definición**: se está evaluando no usar React + TypeScript, ya que a futuro se busca llevar la interfaz a una app móvil con **Ionic + Angular**. Una alternativa en estudio es desarrollar directamente en **Angular + TypeScript** (compartiendo lenguaje y, potencialmente, componentes/lógica con Ionic para la versión móvil) en lugar de React |
| Base de datos | **MySQL** — datos relacionales (usuarios, roles, historial de alertas) |
| Almacenamiento de imágenes | Sistema de archivos (volumen Docker), referenciado por ruta desde la base de datos |
| Empaquetado / portabilidad | **Docker** y `docker-compose` (contenedores para captura, backend, modelo, frontend, proxy y base de datos) |
| Reverse proxy / HTTPS | **Nginx** |
| Notificaciones push | **Firebase Cloud Messaging (FCM)** |
| Integración con cámaras | Envío de imágenes al servidor **vía API / Internet** (mecanismo puntual aún en definición) |
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
- Modo de contingencia: alerta simplificada cuando el espacio disponible cae bajo **30%**.
- Diferenciación entre vacío real (quiebre) y vacío no apilable (ej. frascos de vidrio).
- Robustez ante ruido visual real de sala de venta (~90% de las imágenes con ruido, según Walmart) y distorsión de cámara (fisheye).
- Procesamiento sobre imagen fija (no video en tiempo real).
- Registro trazable de cada análisis (imagen, timestamp, resultado).
- Arquitectura modular: pre-procesamiento / modelo / post-procesamiento, con entradas y salidas tipadas.

---

## Estado actual del repositorio

Este README describe la **arquitectura y tecnologías propuestas** para el proyecto (Fase 1 — definición). El repositorio, a la fecha, contiene:

- Documentación y evidencias de la Fase 1 (`Fase 1/`).
- Stubs iniciales de código en `desarrollo/` (`front.tsx`, `back.tsx`, `css.css`), aún sin implementación.

La arquitectura Docker, la integración de cámaras, el modelo de detección y el frontend descritos arriba **están en diseño** y se irán implementando y ajustando en las siguientes fases del proyecto.

## Referencias

Documento base: *Presentación idea de Proyecto — Stock Eye* (Capstone, Padre Alonso Ovalle).

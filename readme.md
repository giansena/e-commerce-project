# Pipeline Medallón Automatizado en Databricks: De Datos Desestructurados a un Esquema Estrella

> Este proyecto fue desarrollado de forma completamente independiente como pieza clave para mi portfolio personal. No responde a un encargo laboral ni a una propuesta corporativa, sino al desafío técnico de simular un escenario real de ingeniería de datos con fuentes altamente inconsistentes. Para recrear este escenario, generé inicialmente un archivo CSV masivo y deliberadamente caótico utilizando un script de Python local, simulando un volcado de base de datos legacy desestructurado.

---

## Resumen del Proyecto

En el ecosistema de datos actual, lidiar con fuentes de información caóticas es el pan de cada día. Este repositorio implementa un **pipeline de datos end-to-end automatizado** ,orquestado mediante un job en Databricks, diseñado para procesar un flujo de pedidos desastroso y transformarlo, a través de una **arquitectura en capas (Medallion: Bronze $\rightarrow$ Silver $\rightarrow$ Gold)**, en un modelo dimensional limpio de tipo **esquema estrella**. 

Elegí un esquema estrella para la capa final porque representa el estándar de la industria para cargas analíticas y modelado dimensional (OLAP). Al separar los hechos transaccionales de las dimensiones descriptivas (clientes, ubicaciones, productos, estados de pago y tiempo), optimizamos drásticamente el rendimiento de las consultas y facilitamos la integración con herramientas de Business Intelligence.

---

## Arquitectura del Pipeline y Capas

El flujo completo se ejecuta de manera automatizada dentro de Databricks, garantizando idempotencia, trazabilidad y tolerancia a fallos en cada etapa:

### 1. Capa Bronze (Ingesta Cruda)
* **Tecnología:** Delta Lake + Auto Loader (`cloudFiles`) con `Trigger.AvailableNow`.
* **Propósito:** Ingesta incremental e idempotente desde la zona de aterrizaje (`landing_zone/raw_data`) hacia una tabla Delta cruda. 
* **Características clave:** Incorpora evolución automática de esquemas (`addNewColumns`) y una columna de auditoría (`_ingested_at`) para registrar el momento exacto de la ingesta sin perder un solo byte de la fuente original.

### 2. Capa Silver (Limpieza y Normalización)
* **Tecnología:** PySpark.
* **Propósito:** Transformar los datos crudos resolviendo inconsistencias estructurales complejas.
* **Transformaciones aplicadas:**
  * Parseo y separación de cadenas compuestas (`cliente_info` desglosado en nombre, email y ciudad mediante separadores de barras verticales).
  * Limpieza de montos monetarios (eliminación de símbolos como USD, $, ARS, normalización de separadores de miles y decimales, y casteo seguro a float con `try_cast`), he decidido tomar todos los montos como si fuesen de una misma moneda.
  * Estandarización de textos (mayúsculas/minúsculas y control de espacios en blanco, aplicando valores por defecto ante nulos).
  * Explosión de listas de productos anidados (`explode` sobre los ítems comprados extrayendo nombre y cantidad individual con expresiones regulares).
  * Deduplicación estricta del dataset resultante.

### 3. Capa Gold (Modelo Dimensional / Esquema Estrella)
* **Tecnología:** Delta Lake + Funciones de Ventana y Hash en PySpark.
* **Propósito:** Consolidar el negocio en un modelo analítico listo para consumo.
* **Tablas generadas:**
  * `dim_cliente`, `dim_ubicacion`, `dim_producto`, `dim_estado_pago`, `dim_tiempo`.
  * `fact_ventas` (Tabla de hechos que vincula todas las dimensiones a través de Surrogate Keys generadas mediante `xxhash64` sobre atributos naturales).
* **Particularidad analítica:** El motor de fechas maneja tanto formatos estándar (`yyyy-MM-dd`, `MM/dd/yyyy`, etc.) como expresiones relativas del negocio ("Hoy", "Ayer", "Hace 3 dias", "Semana pasada") mapeándolas dinámicamente al calendario.

---

##  Tecnologías y Herramientas Utilizadas

* **Databricks** (Plataforma unificada de análisis)
* **Apache Spark / PySpark** (Procesamiento distribuido)
* **Delta Lake** (Almacenamiento transaccional y confiable en formato columnar)
* **Python** (Generación local del CSV de prueba y scripts de soporte)

---<img width="641" height="430" alt="Captura de pantalla 2026-10-12 161553" src="https://github.com/user-attachments/assets/ecb353ae-82d8-4f9e-9a67-06744dd541c5" />


## Conclusión

Este proyecto refleja la capacidad de diseñar flujos robustos de ingeniería de datos desde cero: desde la gestión de ingestas desacopladas y eficientes con Auto Loader hasta la estructuración rigurosa de almacenes de datos orientados al rendimiento analítico. Si te interesa discutir sobre arquitectura de datos, optimización en Spark o tienes alguna sugerencia, ¡siempre estoy abierto a conectar!

# Actividad 9: Escalabilidad, FinOps y Gobernanza (Responsible AI)
**Proyecto:** Detección Inteligente de Fraude Transaccional (Modelo Dual: ML Clásico + IA Generativa)  
**Materia:** Gestión de Proyectos de Inteligencia Artificial — Universidad Tecmilenio  
**Alumno:** Diego Erando Alejandre Gutiérrez | **Matrícula:** AL07183375  
**Docente:** Luis Ariel Vázquez  

---

## 1. Análisis y Rediseño de Escalabilidad

### Diagnóstico de la Arquitectura Actual (Fase II)
En la Fase II se desplegó el contenedor `api-fraude-dual` (`python:3.9-slim`, servidor `uvicorn`, `app.py` y `model.pkl`) ejecutándose en una sola instancia local/Cloud Shell sobre el puerto `8000`. Aunque esta arquitectura monolítica validó la integración funcional vía peticiones `POST`, presenta cuellos de botella críticos para un entorno bancario en producción:
* **Punto único de fallo (SPOF):** Si el contenedor cae, se interrumpe la autorización de transacciones.
* **Acoplamiento de cargas heterogéneas:** El modelo predictivo de Machine Learning clásico requiere inferencia en milisegundos, mientras que el módulo de IA Generativa (LLM vía RAG) para redactar reportes a analistas tiene mayor latencia y no debe bloquear el flujo transaccional.

### Escalabilidad Vertical vs. Horizontal
Para resolver estas limitaciones, se distingue la aplicación de ambos paradigmas en el sistema dual:
* **Escalabilidad Vertical (*Scale-up*):** Consiste en aumentar la capacidad de CPU y memoria RAM de un solo nodo. En nuestro proyecto, solo resulta útil de forma limitada para el *reentrenamiento* periódico del modelo predictivo o el procesamiento en memoria de lotes históricos, pero tiene un techo físico, genera altos costos y no elimina el punto único de fallo.
* **Escalabilidad Horizontal (*Scale-out*):** Consiste en añadir múltiples réplicas independientes detrás de un balanceador de carga. Es la estrategia idónea para la **inferencia en tiempo real** de tarjetas de crédito, ya que permite absorber picos de demanda estacionales (ej. Buen Fin, quincenas, Navidad) mediante autoescalado automático sin degradar la latencia.

### Propuesta de Rediseño Arquitectónico (Microservicios Desacoplados)
Se propone migrar del contenedor monolítico a una **arquitectura híbrida basada en microservicios y contenedores**:
1. **API Gateway y Balanceador de Carga:** Distribuye las peticiones entrantes cifradas (TLS 1.2+) hacia los nodos disponibles.
2. **Microservicio de Inferencia ML (Autoescalado Horizontal):** Contenedores ligeros con el modelo predictivo (`model.pkl` optimizado) escalando horizontalmente según uso de CPU y peticiones por segundo (RPS) para garantizar respuestas en milisegundos.
3. **Cola de Mensajería y Microservicio GenAI (Asíncrono):** Cuando una transacción supera el umbral de riesgo, se envía un evento a una cola (ej. Pub/Sub o SQS). Un microservicio independiente invoca la API del LLM con contexto RAG y publica el reporte en el *dashboard* del analista sin afectar la latencia de la transacción original.
4. **Capa de Caching (Redis):** Almacena temporalmente perfiles de riesgo recientes y resultados de consultas frecuentes para reducir recomputación y tiempos de respuesta.

---

## 2. Optimización de Costos y Enfoque FinOps

En la Fase I se estimó un gasto operativo recurrente (OPEX) de **$2,700 USD/mes**, dividido en **$1,200 USD/mes** de infraestructura en la nube y **$1,500 USD/mes** por consumo de API de IA Generativa. Bajo un enfoque **FinOps** (gestión financiera continua basada en monitoreo, análisis y alineación con el negocio), se proponen las siguientes estrategias manteniendo el **SLA bancario** (disponibilidad ≥ 99.95% y latencia de inferencia < 100 ms):

1. **Disparo Condicional del LLM (Gobernanza de Consumo):** En lugar de invocar la IA Generativa en todas las transacciones, el microservicio asíncrono solo se activa cuando la probabilidad de fraude supera el umbral de revisión. Dado que el 99.8% de las transacciones son legítimas, esto reduce drásticamente el consumo de tokens de la API.
2. **Right-Sizing e Instancias Spot:** Ajustar CPU y memoria de los contenedores de inferencia a la carga real para evitar sobreaprovisionamiento (*right-sizing*), y utilizar instancias interrumpibles (*spot*) para tareas tolerantes a fallos como el reentrenamiento trimestral del modelo.
3. **Optimización de Inferencia:** Exportar el modelo de *scikit-learn* a formato **ONNX** y aplicar compresión (*pruning* / *quantization*) para reducir el tamaño del artefacto y el consumo de cómputo sin sacrificar precisión.
4. **Almacenamiento por Niveles (*Tiered Storage*) y Optimización de Red:** Clasificar los logs históricos y datasets anonimizados (PCA) en almacenamiento frío de bajo costo, manteniendo todo el tráfico entre microservicios dentro de la misma región y red interna para evitar cargos ocultos de transferencia.

### Comparativa de Optimización FinOps (Antes vs. Después)

| Componente | Antes (Fase I / Fase II) | Después (Rediseño FinOps) | Ahorro Estimado | Impacto en SLA |
| :--- | :--- | :--- | :--- | :--- |
| **Cómputo (Infraestructura)** | Instancias sobredimensionadas 24/7 ($1,200/mes) | *Right-sizing*, autoescalado horizontal y *spot* para reentrenamiento | **30% – 50%** | Mantiene latencia < 100 ms |
| **Inferencia ML** | `model.pkl` sin optimización | Modelo optimizado (ONNX / *pruning*) | **20% – 40%** | Mejora velocidad de respuesta |
| **Consumo API GenAI** | Llamadas frecuentes sin filtro ($1,500/mes) | Invocación asíncrona solo en alertas reales + *caching* de plantillas | **50% – 70%** | Sin impacto en autorización |
| **Almacenamiento y Red** | Uso uniforme de alto costo | *Tiered storage*, compresión y red interna regional | **20% – 30%** | Mantiene disponibilidad ≥ 99.95% |

Para asegurar el cumplimiento del SLA tras la optimización, se integra **observabilidad** (métricas, logs y trazas distribuidas), pruebas de carga periódicas y redundancia selectiva multi-zona.

---

## 3. Auditoría Focalizada: Sesgos y Fairness (Dimensión Ética y Cumplimiento)

Siguiendo la directriz docente de profundizar en un solo eje de auditoría, este análisis se enfoca en la **Auditoría de Sesgos y Fairness en Producción**. Esta elección responde directamente al hallazgo crítico de la Fase II: debido al desbalanceo extremo del dataset de Kaggle (**99.8% transacciones legítimas**), incluso una transacción anómala de $3,500 USD con variables PCA extremas fue clasificada como `"Legitima"` (con solo `0.059%` de confianza de fraude frente a `0.029%` en el caso base).

### Marco Normativo y Clasificación de Riesgo
* **EU AI Act (Reglamento UE 2024/1689):** Al operar en el sector financiero (detección de fraude y bloqueo de medios de pago), el sistema se clasifica como de **Alto Riesgo**. Esto exige gestión continua de riesgos, gobernanza y calidad de datos, trazabilidad, documentación técnica y supervisión humana (*human-in-the-loop*).
* **NIST AI RMF e ISO/IEC 42001:2023:** Se adopta un enfoque de gobernanza *by design* estructurado en las funciones de *Gobernar, Mapear, Medir y Gestionar*, asegurando trazabilidad desde el desarrollo hasta la operación continua.

### Identificación de Sesgos y Métricas de Fairness
En un sistema bancario, el sesgo puede manifestarse en dos direcciones: **sesgo en los datos** (subrepresentación severa de fraudes reales, 0.2%) y **sesgo algorítmico** (el modelo optimiza la exactitud global prediciendo siempre "Legítima" o penalizando desproporcionadamente ciertos patrones geográficos/de monto).

Dado que no existe una única definición universal de equidad, se seleccionan dos métricas contextualizadas para evaluación continua:
1. **Igualdad de Oportunidades (*Equal Opportunity*):** Garantiza la misma tasa de verdaderos positivos (detección efectiva de fraude) entre distintos segmentos de clientes y montos.
2. **Equalized Odds:** Exige igualdad en las tasas de error (falsos positivos y falsos negativos) entre grupos, evitando que un segmento demográfico o geográfico sufra más bloqueos injustificados que otro.

### Ciclo de Auditoría Continua y Mitigación en Producción
Para pasar de principios declarativos a evidencia verificable, se establece el siguiente flujo operativo:
* **Monitoreo y Detección de *Data Drift*:** Dashboards automatizados que vigilan cambios en la distribución de las variables PCA (`V1`–`V28`) y el monto (`Amount`), disparando alertas si el desempeño cae o aparecen disparidades entre grupos.
* **Mitigación Técnica:**
  * *Preprocesamiento:* Aplicar sobremuestreo sintético (**SMOTE**) o balanceo de clases en el entrenamiento para corregir el sesgo del 99.8%.
  * *Posprocesamiento:* Ajustar el umbral de decisión (calibración del *threshold*) y mantener la regla de la Fase I de exigir alta confianza (> 85%) para bloqueos automáticos, derivando casos intermedios a revisión humana.
* **Transparencia y Explicabilidad:** Documentar el propósito, métricas y limitaciones del modelo mediante **Model Cards** y **Datasheets for Datasets**, manteniendo la anonimización PCA y el filtrado de entradas (*input sanitization*) para evitar inyección de prompts en el LLM.

---

## 4. Plan de Escalamiento Responsable

Este plan integra la evolución técnica, la sostenibilidad financiera (FinOps) y los controles de *Responsible AI* en tres horizontes estratégicos:

| Fase y Horizonte | Escalabilidad e Infraestructura | FinOps y Control de Costos | Gobernanza, Ética y Cumplimiento | Indicador Clave (KPI / SLA) |
| :--- | :--- | :--- | :--- | :--- |
| **Fase 1: Consolidación** *(Meses 1–3)* | Desacoplar `api-fraude-dual` en microservicios (ML síncrono + GenAI asíncrono) con balanceo de carga. | Implementar *right-sizing*, disparo condicional del LLM y etiquetado de costos por servicio. | Reentrenar con **SMOTE**, calibrar umbral de decisión y publicar la primera *Model Card*. | Latencia ML < 100 ms; Reducción de gasto API GenAI ≥ 50%. |
| **Fase 2: Automatización** *(Meses 4–6)* | Activar autoescalado horizontal basado en RPS, capa de *caching* (Redis) y exportación del modelo a ONNX. | Migrar reentrenamiento a instancias *spot* y aplicar *tiered storage* a logs históricos. | Implementar monitoreo continuo de *data drift* y métricas de *Equalized Odds* con alertas automatizadas. | Disponibilidad ≥ 99.95%; Reducción de falsos positivos en 30%. |
| **Fase 3: Madurez y Escala** *(Meses 7–12)* | Despliegue multi-zona con pruebas de carga y resiliencia ante caídas de la API externa (plan de contingencia). | Revisiones trimestrales FinOps para validar el *payback* (mes 8–9) y el ROI proyectado. | Adoptar **Governance-as-Code**, auditorías externas trimestrales y alineación certificable con **ISO/IEC 42001**. | Retorno de inversión positivo (mes 8–9); 0 incumplimientos normativos. |

### Roles y Rendición de Cuentas (*Accountability*)
Para evitar la fragmentación organizacional, se asignan responsabilidades claras:
* **Equipo de MLOps / Ingeniería (Proveedor):** Operación del clúster, autoescalado, observabilidad y optimización FinOps.
* **Área de Prevención de Fraudes (Implementador):** Supervisión humana (*human-in-the-loop*) sobre alertas intermedias y validación de reportes del LLM.
* **Comité de Riesgos y Cumplimiento (Auditor Interno):** Ejecución de auditorías trimestrales de *fairness*, revisión de *Model Cards* y verificación de cumplimiento con GDPR y EU AI Act.

---

## Referencias
* Amazon Web Services. (2024). *Cost Optimization Pillar - AWS Well-Architected Framework*.
* European Commission. (2026). *AI Act (Reglamento UE 2024/1689)*.
* FinOps Foundation. (2026). *What is FinOps?*
* Javid, A., & Welch, A. (2025). *AI lifecycle risk management: ISO/IEC 42001:2023 for AI governance*.
* Machine Learning Group - ULB. (2018). *Credit Card Fraud Detection*. Kaggle.
* Manyika, J., & Hassabis, D. (2025). *Responsible AI: Our 2024 report and ongoing work*.
* Papagiannidis, E., Mikalef, P., & Conboy, K. (2025). *Responsible artificial intelligence governance: A review and research framework*. The Journal of Strategic Information Systems.

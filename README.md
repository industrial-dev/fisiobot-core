# 🤖 ClinicBot Core (Agente de Facturación Clínica)

> **Agente inteligente multicanal para automatizar la facturación y contabilidad de cualquier tipo de clínica. Procesa mensajes y tickets, gestiona datos estructurados y genera automáticamente facturas PDF y reportes de Excel (Maestro y Oficial) en la nube.**

✅ **ESTADO DEL PROYECTO: Diseño de arquitectura cerrado. Pendiente de implementación.** Consulta [`ARCHITECTURE.md`](./ARCHITECTURE.md) para el detalle técnico completo (stack, patrones, infraestructura y diagramas).

---

## 📖 Visión General del Proyecto

Este proyecto define el núcleo funcional de un sistema de automatización contable diseñado para eliminar la carga administrativa en clínicas. A través de una interfaz conversacional (como Telegram, WhatsApp o Web), el agente captura la actividad diaria y orquesta el cierre contable. 

El sistema está diseñado para ser **completamente agnóstico y configurable**, permitiendo adaptar las reglas de negocio, nombres de centros asociados y métodos de pago a las necesidades de cualquier clínica.

## 🚀 Características Principales (Requisitos Funcionales)

### 1. Router Cognitivo (Motor de IA por definir)
- **Clasificación de Intenciones:** Analiza si el usuario registra una sesión, un gasto, o lanza un comando del sistema.
- **Validación Dinámica:** Valida que la información contenga los campos mínimos configurados para la clínica (ej. Paciente, Procedencia, Importe, Método de pago). Solicita de forma conversacional los datos faltantes.
- **Extracción de Datos de Imágenes:** Capacidad de extraer fecha, proveedor, concepto e importes desde fotos de tickets de manera automatizada.

### 2. Base de Datos Centralizada (Tecnología por definir)
- Repositorio estructurado que actúa como fuente única de verdad (SSOT). Guarda el histórico inmutable de interacciones validadas, separando el almacenamiento de datos crudos de las exportaciones.

### 3. Arquitectura de Doble Registro (Libros Mayores)
- **📊 Excel Maestro (Tiempo Real):** Documento sincronizado con la base de datos que refleja el 100% de la actividad y el saldo de caja real. Uso estrictamente interno.
- **📊 Excel Oficial (Diferido):** Documento generado bajo demanda en el cierre contable, aplicando las reglas de purga y facturación definidas. Uso para gestoría/auditoría fiscal.

### 4. Motor de Reglas de Negocio (Configurable)
El proceso de "Cierre de Mes" aplica reglas dinámicas basadas en parámetros configurables:
- **Reglas por Método de Pago:** Configurabilidad para determinar qué métodos (ej. Transferencias, Pagos Digitales, Efectivo) se facturan automáticamente al 100%.
- **Facturación Agrupada por Centro/Procedencia:** Capacidad de configurar ciertas entidades, centros derivados o aseguradoras para que sus sesiones se agrupen en una única factura mensual global a nombre de la entidad.
- **Algoritmo de Cuadre de Caja (Aproximación Superior):** Dada una cantidad objetivo ingresada por el usuario, el sistema selecciona aleatoriamente sesiones abonadas en efectivo hasta igualar o superar el objetivo, apartando o "purgando" el resto en la generación del registro Oficial.

### 5. Gestión Documental Cloud (Integración por definir)
- Generación de facturas en PDF con plantillas y numeración adaptables (ej. `[Prefijo]-[Mes]-[Contador]-[Año]`).
- Creación y mantenimiento de la jerarquía de carpetas por años, trimestres y meses en el sistema de almacenamiento definido.

---

## 🏗 Arquitectura Conceptual (Propuesta Lógica)

*Nota: Este esquema refleja flujos lógicos de negocio. Para los diagramas de arquitectura técnica e infraestructura, consulta [`ARCHITECTURE.md`](./ARCHITECTURE.md).*

```mermaid
flowchart TD
    U((Canal de Entrada)) -->|Texto / Imágenes| IA[Capa Cognitiva / Enrutador]
    
    IA -->|Validación / Extracción| DB[(Repositorio Central de Datos)]
    IA -.->|Solicitud de corrección| U
    
    DB -->|Actualización Continua| EX_M[Exportación: Documento Maestro]
    
    C((Comando de Cierre)) --> Motor[Motor de Lógica de Negocio]
    Motor -->|Lectura de periodo| DB
    Motor -->|Ejecución de reglas configurables| DOCS[Motor de Generación Documental]
    
    DOCS --> EX_O[Exportación: Documento Oficial]
    DOCS --> PDF[Facturas PDF]
    DOCS --> NUBE[📁 Almacenamiento Cloud]
```

---

## 🛠 Tecnologías y Decisiones Arquitectónicas (DECIDIDO)

El diseño técnico ha sido cerrado. Resumen de las decisiones — el detalle completo, con diagramas, está en [`ARCHITECTURE.md`](./ARCHITECTURE.md):

- **Modelo de IA:** motor agnóstico vía Azure AI Foundry — se empieza con el modelo más económico disponible (ej. GPT-4o-mini o Mistral Small) para clasificación de intención, validación conversacional y extracción de datos desde imágenes de tickets; el proveedor/modelo se puede cambiar por configuración sin tocar código.
- **Canal de entrada:** Telegram Bot API para el MVP (long polling, sin exposición pública). Portal web (Vue.js) para el staff de las clínicas desde el propio MVP.
- **Backend:** ASP.NET Core (C#/.NET), arquitectura hexagonal/Clean Architecture.
- **Base de Datos:** PostgreSQL relacional — servidor Azure Database for PostgreSQL Flexible Server **compartido** entre clientes (tier Burstable mínimo), con una base de datos separada por cliente, histórico inmutable append-only y columnas `JSONB` para configurabilidad por clínica.
- **Motor de Ejecución Lógica:** lógica de negocio nativa en el backend (no low-code), con Hangfire para jobs idempotentes (cierre de mes) y semilla registrada para el algoritmo de cuadre de caja.
- **Almacenamiento y Archivos:** Azure Blob Storage (dedicado por cliente), ClosedXML para Excel y QuestPDF para PDF.
- **Infraestructura:** Microsoft Azure (crédito Azure for Students — coste minimizado, tiers gratuitos priorizados), región Spain Central, aislamiento físico por cliente en cómputo/storage/secretos, con infraestructura no sensible compartida (frontend, registro de contenedores en GitHub, CI/CD, identidad, servidor de BD). Sin Front Door/WAF por ahora — ver `ARCHITECTURE.md` §11.
- **Cumplimiento:** diseño alineado con RGPD y Veri*Factu desde el modelo de datos (cadena de huellas inmutable por cliente).

---

## 📂 Estructura de Datos (Esquema General)

### Registro de Sesiones (Ejemplo Estructural)
| ID Factura | Fecha | Paciente / ID | Procedencia / Entidad | Método Pago | Importe | Estado Contable |
|---|---|---|---|---|---|---|
| F-1001-202X | 01/10 | Paciente A | Centro Asociado 1 | Digital | 50.00 | Facturado |
| - | 02/10 | Paciente B | Particular | Efectivo | 60.00 | Purgado |

### Registro de Gastos (Ejemplo Estructural)
| Fecha | Proveedor | Concepto | Base Imponible | % Impuesto | Total | URL Recibo |
|---|---|---|---|---|---|---|
| 10/10 | Proveedor X | Suministros | 20.00 | XX% | 24.20 | [Enlace] |

---
*Nota: Este documento describe el producto. Para la arquitectura técnica completa (patrones, infraestructura Azure, modelo de tenancy y diagramas) consulta [`ARCHITECTURE.md`](./ARCHITECTURE.md).*

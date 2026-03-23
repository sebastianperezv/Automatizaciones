# 🛒 Retail Price Intelligence Pipeline | Quadracom

Pipeline analítico automatizado para la monitorización proactiva de competitividad de precios en el sector farmacéutico (retail). 

Este proyecto extrae, procesa y compara diariamente los catálogos de precios de múltiples actores del mercado Costarricense y Chileno para alertar automáticamente a los tomadores de decisiones sobre pérdidas de liderazgo en productos clave.



## 🎯 El Problema de Negocio
En el entorno altamente competitivo de las cadenas de farmacias, comparar manualmente miles de SKUs diarios a través de múltiples archivos Excel es ineficiente y propenso a errores. Las empresas suelen reaccionar tarde a los movimientos agresivos de la competencia, perdiendo margen de ganancia o volumen de ventas.

## 💡 La Solución
Un sistema de orquestación *Serverless* (basado en scripts y demonios de Linux) que:
1. Consolida los datos diarios del mercado.
2. Identifica matemáticamente qué competidor tiene el precio más bajo por producto.
3. Calcula la "Brecha Porcentual" de pérdida de competitividad.
4. Notifica proactivamente al equipo gerencial a través de Slack con los casos más críticos del día.

## 🛠️ Stack Tecnológico
* **Lenguaje:** Python 3
* **Procesamiento de Datos:** `pandas` (Vectorización y cruce de DataFrames)
* **Integraciones:** Slack API (`requests`)
* **Orquestación y Despliegue:** Linux `systemd` (Services & Timers)

---

## ⚙️ Arquitectura y Flujo de Datos

### 1. Ingesta de Datos
El sistema escanea los directorios locales/cloud en busca de las últimas cargas de inventario (`Precios.xlsx`, etc.). Utiliza el *timestamp* del sistema operativo (`os.path.getmtime`) para garantizar que siempre se procese el archivo más reciente de cada competidor, tolerando asimetrías en las fechas de actualización.

### 2. Motor Analítico Central (Pandas)
* **Homologación:** Los catálogos se cruzan (`Outer Merge`) utilizando el código `Primary_key` de la cadena como llave maestra, eliminando la necesidad de reglas difusas de texto.
* **Cálculo de Liderazgo:** Se evalúan los precios lado a lado (`axis=1`) para aislar el `Precio_Minimo` absoluto y etiquetar al `Lider_Actual` del mercado para ese SKU.

### 3. Distribución Multitenant y Alertas (Slack)
El script filtra la "súper-tabla" aislando el contexto de cada cliente (ej. *Solo productos donde la farmacia no es el líder*). Selecciona el Top 5 de brechas porcentuales más críticas y formatea un *payload* nativo para la API de Slack.

**Ejemplo de salida en Slack:**
> 🚨 **ALERTA CRÍTICA: PÉRDIDA DE COMPETITIVIDAD** 🚨
> * 💊 **Paracetamol 500mg Caja x20** (Cod: 7890123)
>   * **Nuestro Precio:** $1.200
>   * **Mejor Precio Mercado:** $990 (Fijado por *Farmacia*)
>   * 📉 **Brecha:** Estamos un *21.2%* más caros.

---

## 🚀 Despliegue en Producción (Linux VM)

La automatización no depende de intervenciones manuales ni de bucles infinitos en Python. Está integrada nativamente en el sistema operativo mediante **systemd**.

**Ejemplo de configuración del Timer (`/etc/systemd/system/pull_exports.timer`):**
```ini
[Unit]
Description=Tue and Thu pull of market exports at 09:55

[Timer]
OnCalendar=Tue,Thu *-*-* 09:55:00
Persistent=false
Unit=pull_exports.service

[Install]
WantedBy=timers.target

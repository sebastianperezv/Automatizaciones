# 🏥 Automated Retail Data Pipeline | B2B Price Intelligence

Pipeline ETL (Extract, Transform, Load) automatizado mediante `systemd` para la gestión diaria de datos de un cliente clave (Top-Tier Retailer) en el sector farmacéutico. 

El sistema extrae *scrapings* desde almacenes en la nube, procesa y cruza matrices de precios con Python, automatiza la subida de los archivos resultantes a nuestra plataforma B2B interna mediante RPA (Playwright), y notifica el estado de cada fase a Slack.

## ⚙️ Arquitectura y Orquestación

El sistema está orquestado enteramente por **systemd (Timers y Services)** en una máquina virtual Linux, asegurando alta disponibilidad y monitoreo nativo.

Se utilizan **Locks (`.lock`)** para prevenir condiciones de carrera y dobles ejecuciones. El flujo está diseñado para ser tolerante a la falta de insumos: si no hay archivo origen un día específico, el sistema cancela la ejecución de forma limpia (`exit_code=2`) sin generar falsos positivos de error.

---

## 🚀 Fases del Pipeline

### 1. Extracción (Pull)
* **Script:** `/scripts/pull_exports_from_drive.sh`
* **Acción:** Utiliza `rclone` para descargar el Excel *raw* del día (`YYYY-MM-DD.xlsx`) desde la nube hacia el entorno local `/scripts/exports/`.

### 2. Motor de Transformación (Python ETL)
* **Script:** `/scripts/data_processor_v1.3.py` (usando `pandas`, `openpyxl`, `xlsxwriter`)
* **Acción:** Este es el núcleo analítico del pipeline. Realiza limpieza profunda, cruces relacionales y formateo corporativo automatizado.

**Lógica de Procesamiento Interno:**
1. **Data Cleansing:** Carga el *scraping* en crudo. Limpia valores nulos/ceros en precios y fuerza la conversión de tipos en los Códigos de Barra (`int64`) para evitar la corrupción por notación científica en Excel.
2. **Payload Generation (Upload Matrix):** Cruza (`Left Join`) los precios limpios contra una matriz estática de carga. Aplica reglas de negocio condicionales basadas en el tipo de empaque (unidad fraccionada vs. entera) para distribuir los precios en columnas separadas, generando el payload exacto que espera la plataforma web.
3. **Market Consumer Reports:** Cruza los datos del cliente contra catálogos de competidores directos. Utiliza `openpyxl` para inyectar estilos corporativos al vuelo (colores hexadecimales, bordes, ajuste dinámico de ancho de columnas y resaltado condicional).
4. **Auxiliary Mapping:** Genera un catálogo extra consolidando IDs internos con descripciones comerciales y precios finales.

* **Outputs Generados:**
  1. `Client_Report DD-MM-YYYY.xlsx` (Data general limpia)
  2. `Upload_Payload DD-MM-YY.xlsx` (Payload para el Bot RPA)
  3. `Consumer_Prices DD-MM-YYYY.xlsx` (Reporte de mercado estilizado)
  4. `Extra_Data DD-MM-YYYY.xlsx` (Catálogo auxiliar)

### 3. Carga RPA (Upload)
* **Script:** `/upload/uploader_client.py` (Wrapper: `run_upload_client.sh`)
* **Acción:** Login automatizado en la plataforma Core B2B mediante Playwright. Busca estrictamente el archivo `Upload_Payload DD-MM-YY.xlsx` del día y ejecuta la carga en la interfaz web.
* **Manejo de Estado:** Escribe el resultado de la operación en `/upload/state/last_run.json`.

---

## 🩺 Monitoreo y Notificaciones (Healthchecks)

El pipeline cuenta con scripts satélite que evalúan el estado de cada fase. Los estados se envían a un Webhook centralizado, el cual formatea la alerta y la publica en un canal de **Slack** dedicado a monitoreo.

**Estados Posibles:**
* ✅ **OK:** Ejecución exitosa y validada.
* ⚠️ **CANCELLED:** Aborto lógico (Ej. no se detectó el archivo de scraping). No representa un fallo técnico.
* 🚨 **FAIL:** Error de ejecución, crash de Python, o falta de outputs obligatorios.

---

## 💻 Comandos de Auditoría y Troubleshooting

Para la administración del servidor, estos son los comandos principales de diagnóstico:

**1. Estado de los Timers (Scheduler):**
```bash
sudo systemctl list-timers --all | grep -E "pull_exports|daily_pipeline|upload_client|healthcheck_"

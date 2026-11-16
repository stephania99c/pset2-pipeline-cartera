# Pipeline de datos end-to-end: cartera de créditos de auto

PSet 2 · Maestría en Ciencia de Datos (USFQ)

Pipeline batch que carga diariamente la cartera de créditos, la limpia y la modela en Snowflake:

```
Fuente (GitHub Release) → Kestra → RAW → dbt → BRONZE → dbt → SILVER → dbt → GOLD (star schema) → Spark → OBT
```

> **Datos:** todos los datos son **anonimizados y sintéticos** (identificadores seudonimizados y montos perturbados).
> Los datos reales nunca salen del equipo de la autora. La fuente pública es el Release
> [`datos-v1`](https://github.com/stephania99c/pset2-pipeline-cartera/releases/tag/datos-v1).

## Estructura del repositorio

| Carpeta / archivo | Contenido |
|---|---|
| `docker-compose.yml` | Servicios: **Kestra** (orquestador), **dbt** y **Spark** |
| `.env.example` | Variables necesarias (copiar a `.env`; el `.env` real nunca se sube) |
| `herramientas/` | `Dockerfile` y `requirements.txt` de la imagen `pset2-tools:1.0` (Python, Java, dbt, Spark) |
| `snowflake/setup.sql` | Crea warehouse, base `CARTERA`, esquemas, rol y usuario de servicio |
| `ingesta/ingesta.py` | Carga un día de la fuente al esquema RAW (idempotente) |
| `kestra/flow_ingesta_cartera.yml` | Flow diario con horario, reintentos y backfill |
| `calidad/diagnostico_calidad.ipynb` | Diagnóstico de calidad sobre RAW y tabla Problema / Evidencia / Acción / Justificación |
| `dbt/` | Modelos Bronze, Silver y Gold (star schema) con tests |
| `spark/job_obt.py` | Construye la OBT desde Gold y la guarda en Snowflake |
| `docs/diagramas.md` | Diagramas de arquitectura, star schema y OBT |

## Requisitos

- Una cuenta de **Snowflake** (sirve la de prueba gratuita).
- **Docker** con Docker Compose. Se probó en **GitHub Codespaces** (4 núcleos), que ya trae Docker.

## Cómo ejecutarlo

**1. Preparar Snowflake.** En un worksheet de Snowflake, ejecutar `snowflake/setup.sql` completo (Ctrl+A → Run) con el rol ACCOUNTADMIN. La última sentencia genera un token (PAT): copiarlo, porque se muestra una sola vez.

**2. Configurar credenciales.**
```bash
cp .env.example .env
```
Completar `.env` con la cuenta de Snowflake y el token del paso 1.

**3. Construir la imagen de herramientas y levantar Kestra.**
```bash
docker build -f herramientas/Dockerfile -t pset2-tools:1.0 .
docker compose up -d
```
Kestra queda en el puerto 8080 (usuario y clave definidos en `.env`).

**4. Cargar el flow y hacer el backfill.** En Kestra: *Flows → Create*, pegar `kestra/flow_ingesta_cartera.yml` y guardar. Luego, en *Triggers → diario → Backfill*, elegir del 2026-10-01 al 2026-10-31. También se puede ejecutar un día puntual:
```bash
curl -u "$KESTRA_USER:$KESTRA_PASSWORD" -X POST \
  "http://localhost:8080/api/v1/main/executions/pset2.cartera/ingesta_cartera" -F fecha=2026-10-31
```

**5. Diagnóstico de calidad.**
```bash
pip install "snowflake-connector-python[pandas]" python-dotenv ipykernel
```
Abrir `calidad/diagnostico_calidad.ipynb` y ejecutar todas las celdas.

**6. Transformaciones con dbt (Bronze → Silver → Gold) y tests.**
```bash
docker compose run --rm dbt debug
docker compose run --rm dbt build
```
`build` construye todos los modelos y ejecuta todos los tests (37 en total).

**7. OBT con Spark.**
```bash
docker compose run --rm spark job_obt.py
```
Imprime las validaciones del grano y guarda `CARTERA.GOLD.OBT_CARTERA_DIARIA`.

## Decisiones principales

| Tema | Decisión |
|---|---|
| Procesamiento | **Batch diario** (7:00, America/Guayaquil): el corte de cartera se genera una vez al día |
| Ingesta | Idempotente: borra e inserta el día (sin duplicados al reintentar); reportes mensuales con recarga completa |
| Kestra | 3 reintentos cada 5 minutos, una ejecución a la vez (`concurrency: 1`) y backfill desde la interfaz |
| Bronze | Tipos y nombres estandarizados; vistas |
| Silver | Reglas de negocio **marcadas** (no se borra nada): `es_auto`, `es_principal`, `es_aplicado`, `es_recaudacion`, `es_call_center`, `es_posible_duplicado` |
| Gold | Star schema filtrado al alcance: créditos de auto, pagos de recaudación aplicados y gestiones de call center |
| OBT | Grano operación + día; se construye con Spark desde Gold |

## Seguridad

- Las credenciales viven solo en `.env` (excluido por `.gitignore`); el repositorio no contiene claves ni datos.
- Snowflake usa un usuario de servicio (`PSET2_SERVICE`) con un rol limitado y un token con vencimiento.

## Uso de IA

Para este trabajo se utilizó un asistente de IA (Claude) como apoyo para escribir y explicar el código. Las decisiones de negocio (alcance, reglas de calidad y su justificación) provienen del conocimiento de la autora sobre el proceso de cobranzas.

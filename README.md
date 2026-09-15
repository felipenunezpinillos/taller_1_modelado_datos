# Taller 1 — Caso VehiAlpes

Modelo dimensional, ELT con arquitectura medallón, consultas analíticas y tableros de control sobre los datos de VehiAlpes.

**MINE-4214 Modelado y Diseño de Datos** · Universidad de los Andes · Facultad de Ingeniería · Departamento de Ingeniería de Sistemas y Computación

---

## El caso

VehiAlpes opera tres líneas de negocio sobre un mismo activo: compra de vehículos, alquiler de flota y venta de usados. Las directivas no piden reportes operativos; piden entender el **ciclo de vida del vehículo** para decidir qué comprar, cuáles conservar en alquiler y por cuánto tiempo, y cuándo venderlos.

La solución es un modelo dimensional de seis dimensiones conformadas y seis tablas de hechos, cargado en Databricks con arquitectura medallón, sobre el que se resuelven dos requerimientos de negocio en SQL y se construyen dos tableros AI/BI.

---

## Contenido del repositorio

| Archivo | Entregable | Descripción |
|---|---|---|
| `Entregable1_VehiAlpes_Modelo_Dimensional.docx` | 1 | Modelo dimensional descrito y analizado: diagramas E/R, matriz de bus, fortalezas, limitaciones OLAP y el modelo lógico en anexo |
| `vehialpes_modelo_dimensional.dbml` | 1 | Modelo lógico para renderizar en dbdiagram.io, con notas de diseño por tabla |
| `er_nucleo_transaccional.png` | 1 | Diagrama E/R del núcleo: tres hechos transaccionales y seis dimensiones |
| `er_capa_derivada.png` | 1 | Diagrama E/R de la capa derivada: snapshots y tabla factless |
| `01_bronze.ipynb` | 2 | Ingesta cruda a Delta con Auto Loader |
| `02_silver.ipynb` | 2 | Estandarización de fechas, calidad de datos y SCD tipo 2 |
| `03_gold_dimensiones.ipynb` | 2 | Seis dimensiones con llaves surrogadas y centinelas |
| `04_gold_hechos.ipynb` | 2 | Seis hechos: tres transaccionales y tres derivados |
| `05_consultas_sql.ipynb` | 3 | Las dos consultas con `%sql`, el rol del usuario y el análisis |
| `entregable3_consultas.sql` | 3 | Las mismas consultas como archivo SQL independiente |
| `vehialpes_dashboard.lvdash.json` | 4 | Tableros AI/BI importables directamente en Databricks |
| `entregable4_especificacion_tableros.md` | 4 | Especificación de cada widget y descripción de los tableros |

Los cinco notebooks llevan las justificaciones de diseño en celdas de markdown, así que se leen como documento además de ejecutarse.

---

## Modelo de datos

![Núcleo transaccional](er_nucleo_transaccional.png)

*Núcleo transaccional. Los atributos en cursiva son medidas; los marcados con asterisco se versionan con SCD tipo 2; (DD) indica dimensión degenerada.*

![Capa derivada](er_capa_derivada.png)

*Capa derivada, construida en gold a partir de los hechos transaccionales.*

| Hecho | Tipo | Grano | Filas |
|---|---|---|---|
| Compra | Transaccional | una compra | 500 |
| Alquiler | Transaccional | un contrato | 4.356 |
| Venta | Transaccional | una venta | 150 |
| Ocupación diaria | Snapshot periódico | un vehículo-día | 470.781 |
| Ciclo de vida | Snapshot acumulativo | una placa | 500 |
| Historia de versiones | Factless | un cambio de versión | 879 |

---

## Cómo reproducirlo

### Requisitos

- Workspace de Databricks con Unity Catalog y un SQL warehouse (serverless sirve).
- DBR 14.3 LTS o superior: el pipeline usa `try_cast`, `try_to_timestamp` y volúmenes de Unity Catalog.
- Los tres CSV de la fuente (`carros.csv`, `clientes.csv`, `transacciones.csv`), entregados con el enunciado del taller. No se incluyen en este repositorio.

### Pasos

1. **Importar los notebooks.** Workspace → clic derecho en la carpeta destino → *Import* → *File*, y cargar los cinco `.ipynb`.

2. **Crear la estructura.** Ejecutar únicamente la primera celda de código de `01_bronze`. Crea el catálogo `vehialpes`, los esquemas `landing`, `bronze`, `silver` y `gold`, el volumen `landing.archivos` y las subcarpetas de aterrizaje. Imprime la ruta de subida.

   > Si el `CREATE CATALOG` falla por permisos, cambiar la variable `CATALOGO` por un catálogo existente y eliminar esa línea. Todas las rutas se derivan de esa variable.

3. **Subir los CSV, cada uno en su carpeta.** En Catalog: `vehialpes` → `landing` → `archivos` → `crm`, y cargar cada archivo en la subcarpeta con su nombre:

   ```
   /Volumes/vehialpes/landing/archivos/crm/carros/carros.csv
   /Volumes/vehialpes/landing/archivos/crm/clientes/clientes.csv
   /Volumes/vehialpes/landing/archivos/crm/transacciones/transacciones.csv
   ```

   Auto Loader vigila un directorio por tabla y mantiene un esquema separado para cada uno. Si los tres archivos caen en la misma carpeta, la ingesta falla.

4. **Ejecutar en orden:** `01_bronze` → `02_silver` → `03_gold_dimensiones` → `04_gold_hechos`. Cada notebook lee las tablas que dejó el anterior. Los tres últimos son idempotentes; bronze no reprocesa un archivo ya ingerido, por el checkpoint de Auto Loader.

5. **Consultas.** Ejecutar `05_consultas_sql`.

6. **Tableros.** Dashboards → menú desplegable junto a *Create dashboard* → *Import dashboard from file* → `vehialpes_dashboard.lvdash.json`. Asignar el SQL warehouse y publicar.

Si el catálogo no se llama `vehialpes`, reemplazar el prefijo `vehialpes.gold.` en `05_consultas_sql`, en `entregable3_consultas.sql` y en los datasets del JSON del tablero.

---

## Resultados esperados

Todos los notebooks incluyen aserciones: si alguna falla, el pipeline se detiene en el punto del problema en lugar de producir resultados silenciosamente incorrectos. Estos son los valores verificados sobre los archivos entregados:

| Capa | Verificación | Valor |
|---|---|---|
| Bronze | carros / clientes / transacciones | 1.379 / 1.560 / 5.059 |
| Silver | tras deduplicar | 5.059 → 5.006 |
| Silver | por proceso (compra / alquiler / venta) | 500 / 4.356 / 150 |
| Silver | SCD tipo 2 | 1.379 versiones sobre 500 placas |
| Silver | banderas de calidad | 53 deduplicados · 7 fechas reparadas · 0 en cuarentena · 432 valor inconsistente |
| Gold | dimensiones | fecha 1.494 · vehículo 1.381 · cliente 1.592 · sucursal 8 · proveedor 7 · pago 32 |
| Gold | hechos | 500 / 4.356 / 150 / 470.781 / 500 / 879 |
| Gold | conciliación entre granos | 37.410 − 37.345 = 65 = días-carro solapados |

La última fila es la validación más valiosa del pipeline: la diferencia entre el snapshot diario y el hecho transaccional debe igualar exactamente los días-carro cubiertos por más de un contrato. Si no coincide, hay un error de fronteras en el snapshot y las métricas de ocupación no son confiables.

---

## Decisiones de diseño

**SCD tipo 2 en la dimensión de vehículo, por necesidad aritmética.** Se verificó que el valor de cada alquiler equivale a los días multiplicados por la tarifa diaria *de la versión del vehículo vigente en la fecha de inicio del contrato*, no de la versión actual. Sin SCD tipo 2, todo alquiler anterior a la última actualización del vehículo queda mal valorado. Una actualización masiva del 1 de noviembre de 2024 afectó las 500 placas, así que el error alcanzaría a la mayoría del histórico.

**Tres hechos en lugar de uno.** La fuente entrega compras, alquileres y ventas en una sola tabla. Cargarla tal cual produciría granos mezclados, atributos nulos en el 90% de las filas y medidas cuyo significado cambia según el tipo de transacción.

**Lectura por nombre de columna, no por esquema posicional.** El orden real de `carros.csv` y `transacciones.csv` no coincide con el orden intuitivo. Con un esquema explícito, Spark mapea por posición e ignora el encabezado, lo que desplaza los valores en silencio: `color` termina con "Gasolina" y las fechas quedan nulas. El pipeline terminaría sin errores y con todas las tarifas en cero.

**`try_cast` en lugar de `cast`.** Con ANSI activo —el default en DBR 14+ y en serverless— `cast('' as int)` lanza excepción y aborta el job. Las columnas numéricas traen cadenas vacías en la cola defectuosa del archivo.

**Cuatro formatos de fecha, cada uno con su guarda.** `fecha_inicio` trae ISO (55,5%), `DD/MM/AAAA` (19,2%), `MM-DD-AAAA` (14,8%) y `DD-mmm-AAAA` con mes en español (10,5%). Un `coalesce` de intentos sucesivos sin guardas produce errores indetectables: `08-07-2024` se leería como 8 de julio en vez de 7 de agosto. Cada formato se identifica por expresión regular antes de parsear.

**Deduplicación por `id_transaccion` menor.** Los 60 registros con identificador ≥ 5000 son la cola defectuosa del archivo, y 53 de ellos duplican un registro previo. Conservar el identificador menor equivale a conservar el original y descartar el anexo. Como efecto colateral desaparecen todos los nulos de `valor_total` y `km_transaccion`, porque viven en las copias: una regla reemplaza cinco.

**Marcar y no corregir el 10% de montos inconsistentes.** 432 de los 4.356 alquileres se desvían de la fórmula esperada. Tres de los cuatro patrones detectados (descuento del 10%, un día menos cobrado, tarifa de una versión anterior) son indistinguibles de reglas de negocio no documentadas; corregirlos sería sustituir el dato transaccional por un supuesto propio. Se conserva el valor de la fuente, se calcula el teórico al lado y se expone la brecha.

**Reparar las 7 fechas invertidas en lugar de descartarlas.** En los 7 casos `fecha_fin` es exactamente `fecha_inicio` menos dos días, y al dividir `valor_total` entre la tarifa vigente el resultado es un entero exacto. La duración es reconstruible y la reparación queda marcada.

---

## Hallazgos de negocio

**La ocupación no depende de la marca.** Va de 14,1% a 16,2% en las siete marcas con masa crítica. Los clientes alquilan lo que está disponible. Pero la tarifa diaria no escala con el precio de compra: un vehículo de 90 M rinde 18,8 K por día en flota y uno de 271 M rinde 32,3 K — tres veces la inversión para menos del doble del ingreso.

**La ventana de rotación óptima son 15 meses.** Ocupación, ingreso diario y valor residual alcanzan su máximo en el mismo tramo de 12 a 15 meses (20,2% · 58,0 K · 70,7%) y después caen a la vez: el vehículo se alquila menos y además vale menos. Las ventas observadas se concentran entre 15 y 24 meses, es decir, VehiAlpes ya vende tarde.

---

## Limitaciones

**La muestra está sesgada hacia 2024.** Hay 3.137 alquileres en 2024, 1.230 en 2025 y 23 en 2026. Las consultas restringen la ocupación a 2024 por esa razón. La ocupación calculada por año (15,3%, 6,6% y 0,2%) refleja el muestreo y no una tendencia del negocio.

**El tramo temprano de la curva de antigüedad está contaminado.** Los alquileres arrancan en enero de 2024 mientras las compras arrancan en diciembre de 2022, así que los vehículos jóvenes conviven con un histórico sin actividad. El tramo desde los 15 meses es el confiable, y es el que sustenta la decisión de rotación.

**El ciclo de vida está incompleto para el 70% de la flota.** Solo 150 de 500 vehículos se han vendido; los promedios que dependen del cierre del ciclo sufren sesgo de supervivencia.

**No hay rentabilidad neta.** La fuente solo aporta un costo de mantenimiento estimado por kilómetro como atributo del vehículo, no el gasto efectivo.

**No se puede segmentar corto y largo plazo.** El enunciado menciona una flota "a corto y largo plazo", pero todos los alquileres de la muestra duran entre 2 y 15 días.

**Dieciséis vehículos (41 versiones) tienen tarifas fuera de escala**, entre 5.000 y 11.000 o entre 3,1 y 4,9 millones por día. Los alquileres se cobraron con esas mismas tarifas, así que el error proviene del origen. Quedan marcados con una bandera y no se corrigen.

---

## Uso de IA generativa

Se utilizó Claude (Anthropic) para apoyar el perfilamiento de los datos, la discusión de alternativas de diseño dimensional, la escritura de los notebooks y la redacción de la documentación. El pipeline completo fue ejecutado y sus cifras verificadas contra los archivos entregados; las decisiones de modelado y su justificación fueron revisadas y adoptadas por el grupo.

---

## Integrantes

- Felipe Nuñez

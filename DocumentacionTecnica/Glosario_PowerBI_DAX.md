# Glosario — Bases de Datos, Modelado y DAX (Power BI)

## 1. Conceptos de arquitectura de datos

**OLTP (Online Transaction Processing)**
Sistema diseñado para registrar transacciones del día a día en tiempo real (ventas, facturación, inventario). Está normalizado (muchas tablas pequeñas relacionadas) para que cada operación (insertar, actualizar, borrar un registro) sea rápida y consistente. En este proyecto, la base `DLBENDICION` en `SRV-DEMOSOFT` (tablas como `tbl_Documento`, `tbl_Transaccion`, `TBL_PRODUCTO`) es el sistema OLTP: es donde el ERP/sistema operativo escribe cada movimiento.

**OLAP (Online Analytical Processing)**
Sistema diseñado para **analizar** grandes volúmenes de datos históricos: agregaciones, comparativos, tendencias. Está desnormalizado (menos tablas, más anchas) para que las consultas de lectura y agregación sean rápidas. `DLBENDICION_DWH` (el modelo estrella que arma el SP) y el propio modelo de Power BI son la capa OLAP.

**DWH (Data Warehouse / Almacén de datos)**
Base de datos separada del sistema transaccional, optimizada para reportes y análisis histórico. Se alimenta del OLTP mediante procesos ETL. `DLBENDICION_DWH` cumple este rol.

**ETL (Extract, Transform, Load)**
Proceso de: 1) **Extraer** datos del origen (OLTP), 2) **Transformar** (limpiar, unir, renombrar, calcular columnas, aplicar filtros de negocio) y 3) **Cargar** el resultado en el destino (DWH). El stored procedure `PaGeneraModeloEstrellaCompleto` es un ETL completo escrito en T-SQL.

**Linked Server**
Configuración de SQL Server que permite que una instancia de SQL Server (donde vive `DLBENDICION_DWH`) consulte directamente tablas de otra instancia remota (`SRV-DEMOSOFT`) usando sintaxis de 4 partes: `[Servidor].[BaseDatos].[esquema].[tabla]`. Permite hacer ETL entre dos servidores distintos sin mover archivos manualmente.

---

## 2. Modelado dimensional (Kimball)

**Modelo Estrella (Star Schema)**
Diseño donde una **tabla de hechos** central se relaciona directamente con varias **tablas de dimensión**, sin pasos intermedios. Es el estándar recomendado por Power BI porque simplifica las relaciones y mejora el rendimiento del motor VertiPaq.

**Modelo Copo de Nieve (Snowflake Schema)**
Variante donde las dimensiones a su vez se relacionan con sub-dimensiones (normalizadas). Más "correcto" desde el punto de vista relacional, pero más lento y complejo de navegar en Power BI. Este proyecto usa estrella pura (las dimensiones ya vienen pre-unidas/desnormalizadas por el SP, ej. `DIM_Bodega` ya incluye Empresa y Localización).

**Tabla de Hechos (Fact Table)**
Tabla que contiene las **métricas numéricas** del negocio (montos, cantidades, costos) y las **llaves foráneas** hacia las dimensiones. Suele ser la tabla más larga (más filas) del modelo. Aquí: `FACT_Movimientos`.

**Tabla de Dimensión (Dimension Table)**
Tabla que contiene los **atributos descriptivos** para filtrar y agrupar los hechos (nombre de producto, nombre de cliente, fecha, bodega). Suelen ser tablas anchas pero con pocas filas comparadas con el fact. Aquí: `DIM_Producto`, `DIM_Cliente`, `Calendario`, etc.

**Grano (Grain)**
El nivel de detalle que representa **una fila** de la tabla de hechos. Definir el grano es la decisión más importante del modelado. En `FACT_Movimientos`, el grano es: una fila por transacción de documento (`Documento` + `Consecutivo`), a nivel de producto.

**Llave Primaria (Primary Key / PK)**
Columna (o combinación de columnas) que identifica de forma única cada fila de una tabla. En dimensiones como `DIM_Bodega`, la PK es compuesta: `IdBodega = Empresa + '-' + Bodega`.

**Llave Foránea (Foreign Key / FK)**
Columna en la tabla de hechos que apunta a la PK de una dimensión, estableciendo la relación. Ej: `FACT_Movimientos[IdProducto]` → `DIM_Producto[IdProducto]`.

**Llave Compuesta (Composite/Surrogate Key)**
Cuando un solo campo no basta para identificar un registro de forma única, se concatenan varios campos en uno solo (ej. `CONCAT(Empresa, '-', Bodega)`). Se usa para evitar ambigüedad y relaciones "muchos a muchos" accidentales (que Power BI resuelve mal por defecto, generando productos cartesianos).

**Producto Cartesiano**
Resultado incorrecto que ocurre cuando una relación entre tablas no es realmente única (1 a muchos) sino ambigua, generando combinaciones de filas que no existen en la realidad y duplicando/multiplicando los valores numéricos. Se evita con llaves únicas bien definidas (por eso las llaves compuestas del SP).

**Cardinalidad de una relación**
Describe cuántas filas de una tabla se relacionan con cuántas filas de otra: `1 a 1`, `1 a muchos (1:*)`, `muchos a muchos (*:*)`. En este modelo, todas las relaciones dimensión→hecho son `1 a muchos`.

**Dirección de filtro (Filter direction)**
Define hacia dónde "viaja" el filtro en una relación: **única** (de dimensión hacia hecho, lo recomendado) o **ambas** (bidireccional, usar con cuidado porque puede generar ambigüedad o afectar el rendimiento).

**Tabla de Fechas / Calendario (Date Table)**
Dimensión especial con una fila por cada día del rango de análisis. Se marca explícitamente en Power BI como "Date Table" para habilitar las funciones de inteligencia de tiempo. Debe tener una columna de tipo fecha, sin huecos ni duplicados.

---

## 3. Power Query (M) vs. DAX — diferencia clave

| | **Power Query (M)** | **DAX** |
|---|---|---|
| ¿Cuándo actúa? | Al **cargar/actualizar** los datos (ETL dentro de Power BI) | Al **consultar/visualizar** los datos (tiempo de reporte) |
| ¿Qué hace? | Limpia, une, filtra, cambia tipos de datos, crea columnas antes de cargar el modelo | Calcula medidas, columnas calculadas, tablas calculadas sobre el modelo ya cargado |
| Lenguaje | Funcional, basado en pasos (`let...in`) | Basado en fórmulas, similar a Excel pero con contexto |

---

## 4. Conceptos fundamentales de DAX

**Medida (Measure)**
Fórmula que calcula un valor **dinámicamente** según el contexto de filtro del visual (fecha, producto, cliente, etc. seleccionados). No ocupa espacio en el modelo hasta que se calcula. Ejemplo: `Total Ventas = SUM(FACT_Movimientos[VtaSinIva])`.

**Columna Calculada**
Fórmula que se calcula **fila por fila** y se guarda físicamente en el modelo (ocupa espacio, se recalcula al actualizar datos). Se usa cuando el resultado debe segmentarse/filtrarse como si fuera una columna normal (ej. en un slicer).

**Tabla Calculada**
Tabla generada completamente por una fórmula DAX (no viene de una fuente externa). Ejemplo en este proyecto: `Calendario`, creada con `CALENDAR()` + `ADDCOLUMNS()`.

**Contexto de Fila (Row Context)**
El "estado" en el que DAX sabe en qué fila específica está parado (por ejemplo, dentro de una columna calculada, o dentro de funciones iteradoras como `SUMX`).

**Contexto de Filtro (Filter Context)**
El conjunto de filtros activos que rodean un cálculo en un momento dado (los filtros de un slicer, las filas/columnas de una tabla o matriz, otra medida que lo envuelve con `CALCULATE`). Es el concepto más importante para entender por qué una misma medida da resultados distintos en distintas celdas de un visual.

**Transición de Contexto (Context Transition)**
Lo que ocurre cuando `CALCULATE` convierte un contexto de fila en un contexto de filtro equivalente. Es clave para entender medidas anidadas.

---

## 5. Funciones DAX usadas en este proyecto (y para qué sirven)

| Función | Categoría | Qué hace |
|---|---|---|
| `SUM(columna)` | Agregación | Suma todos los valores de una columna numérica en el contexto de filtro actual. |
| `DIVIDE(numerador, denominador, [alternativo])` | Aritmética segura | Divide dos valores devolviendo un resultado alternativo (por defecto en blanco) si el denominador es 0, evitando errores `#DIV/0!`. Preferible a usar `/` directamente. |
| `DISTINCTCOUNT(columna)` | Agregación | Cuenta valores **únicos** (no duplicados) de una columna. Usada para contar documentos/transacciones sin duplicar por línea de detalle. |
| `CALCULATE(expresión, filtro1, filtro2, ...)` | Modificador de contexto | La función más importante de DAX: recalcula una expresión modificando el contexto de filtro actual (agregando, quitando o reemplazando filtros). Base de casi toda la inteligencia de tiempo. |
| `CALENDAR(fecha_inicio, fecha_fin)` | Tabla de fecha | Genera una tabla con una columna `Date` con un renglón por cada día entre las dos fechas dadas. |
| `ADDCOLUMNS(tabla, "nombre", expresión, ...)` | Manipulación de tablas | Devuelve una tabla agregando columnas calculadas adicionales a una tabla existente. |
| `YEAR / QUARTER / MONTH / DAY / WEEKNUM(fecha)` | Fecha | Extraen el año, trimestre, mes, día o número de semana de una fecha. |
| `FORMAT(valor, "formato")` | Texto/Formato | Convierte un valor (fecha o número) a texto con un formato específico (ej. `"MMMM"` → nombre completo del mes). |
| `TOTALYTD(expresión, columna_fecha)` | Inteligencia de tiempo | *Year To Date*: acumula la expresión desde el 1 de enero hasta la última fecha del contexto de filtro. |
| `TOTALMTD(expresión, columna_fecha)` | Inteligencia de tiempo | *Month To Date*: acumula desde el día 1 del mes hasta la última fecha del contexto. |
| `TOTALQTD(expresión, columna_fecha)` | Inteligencia de tiempo | *Quarter To Date*: acumula desde el inicio del trimestre hasta la última fecha del contexto. |
| `SAMEPERIODLASTYEAR(columna_fecha)` | Inteligencia de tiempo | Devuelve el mismo período de fechas pero desplazado un año atrás; se usa dentro de `CALCULATE` para comparar YoY. |
| `DATEADD(columna_fecha, número, intervalo)` | Inteligencia de tiempo | Desplaza el período de fechas del contexto hacia adelante o atrás según el intervalo (`DAY`, `MONTH`, `QUARTER`, `YEAR`). Más flexible que `SAMEPERIODLASTYEAR`. |

---

## 6. Siglas y términos frecuentes en reportes

| Sigla | Significado | Uso típico |
|---|---|---|
| **KPI** | Key Performance Indicator (Indicador Clave de Desempeño) | Métrica que resume el desempeño del negocio frente a una meta. |
| **YTD** | Year To Date | Acumulado del año hasta la fecha. |
| **MTD** | Month To Date | Acumulado del mes hasta la fecha. |
| **QTD** | Quarter To Date | Acumulado del trimestre hasta la fecha. |
| **YoY** | Year over Year | Comparación contra el mismo período del año anterior. |
| **MoM** | Month over Month | Comparación contra el mes inmediato anterior. |
| **PK** | Primary Key (Llave Primaria) | Identificador único de una tabla. |
| **FK** | Foreign Key (Llave Foránea) | Campo que referencia la PK de otra tabla. |
| **ERP** | Enterprise Resource Planning | Sistema que administra las operaciones del negocio (el origen OLTP). |
| **IVA** | Impuesto al Valor Agregado | En Guatemala, 12%; por eso `VtaSinIva = monto / 1.12`. |

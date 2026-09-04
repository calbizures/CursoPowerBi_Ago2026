# Documentación — Módulo 2 Power BI (Modelo Estrella DLBendición)

## 1. Origen de los datos

Todo el modelo se alimenta del **stored procedure** `[dbo].[PaGeneraModeloEstrellaCompleto]`, ubicado en la base `DLBENDICION_DWH`, el cual lee los datos transaccionales desde el servidor origen `[SRV-DEMOSOFT].DLBENDICION` (vía **Linked Server**) y construye un **modelo estrella (Star Schema)** físico compuesto por una tabla de hechos y seis dimensiones.

```sql
EXEC PaGeneraModeloEstrellaCompleto
```

### ¿Qué hace el procedimiento, paso a paso?

1. **Limpieza**: elimina (`DROP TABLE IF EXISTS`) las tablas `FACT_Movimientos` y todas las `DIM_*` antes de recrearlas, garantizando una carga limpia en cada ejecución (proceso tipo *truncate & load*).
2. **Carga de dimensiones**: extrae y transforma los datos maestros del sistema origen (ver detalle en la sección 2).
3. **Carga de la tabla de hechos**: cruza documentos (`tbl_Documento`) con transacciones (`tbl_Transaccion`) aplicando reglas de negocio y filtros (ver sección 3).
4. Todo corre dentro de una **transacción** (`BEGIN TRANSACTION` / `COMMIT` / `ROLLBACK` en el `CATCH`), asegurando que si algo falla, no queden tablas a medio construir.

### Decisiones de diseño importantes

- **Llaves compuestas**: como el sistema origen identifica bodegas por `Empresa + Bodega` y series de documento por `Tipo + Serie`, el SP concatena esos campos (`CONCAT(Empresa, '-', Bodega)` y `CONCAT(Tipo, '-', Serie)`) para generar una llave **única** por dimensión. Esto evita relaciones ambiguas o productos cartesianos al conectar las tablas en Power BI.
- **Sin tabla de fechas nativa**: el origen no trae una dimensión de calendario; `FechaKey` llega como una fecha suelta en `FACT_Movimientos`. Por eso el modelo de Power BI construye su propia tabla `Calendario` con DAX (ver sección 5).

---

## 2. Tablas de dimensión

| Tabla | Llave primaria | Columnas | Descripción / origen |
|---|---|---|---|
| `DIM_Bodega` | `IdBodega` (Empresa-Bodega) | BodegaNombre, EmpresaNombre, LocalizacionNombre | Une `tbl_Bodega` + `tbl_Empresa` + `LocalizacionBodega` + `tbl_Localizacion`. Representa la bodega física donde ocurre el movimiento, junto con la empresa y localización a la que pertenece. |
| `DIM_Serie` | `IdSerie` (Tipo-Serie) | SerieCodigo, SerieNombre | Desde `tbl_serie_documento`. Identifica el tipo/serie de documento (ej. facturas, notas de crédito). |
| `DIM_Producto` | `IdProducto` | ProductoNombre, ClaseProducto | Desde `TBL_PRODUCTO` + `tbl_Clase_Producto`. Si el producto no tiene clase asignada, se etiqueta como `"Sin Clase"`. |
| `DIM_UnidadMedida` | `IdUnidadMedida` | Descripcion | Desde `tbl_Unidad_Medida`. Unidad en la que se mide/vende el producto (unidad, caja, libra, etc.). |
| `DIM_Cliente` | `IdCliente` | ClienteNombre, TipoClienteNombre | Desde `tbl_cuenta_Correntista` + `Tbl_Grupo_Cuenta_Cta` + `Tbl_Grupo_Cuenta`. Si el cliente no tiene categoría, se etiqueta `"Sin Categoría"`. |
| `DIM_TipoTransaccion` | `IdTipoTransaccion` | Descripcion | Desde `tbl_tipo_transaccion`. Naturaleza del movimiento (ej. venta, devolución, ajuste). |
| `Calendario` | `Date` | Año, Trimestre, NoMes, NoDia, NoSemana, Mes, Nombre Mes, Mes Corto, Día Semana, Año Mes, MesKey | Tabla de fechas generada en **DAX** dentro de Power BI (no viene del SP). Marcada como *Date Table*. |

---

## 3. Tabla de hechos — `FACT_Movimientos`

Cruza `tbl_Documento` (cabecera) con `tbl_Transaccion` (detalle) por documento, tipo, serie, empresa, localización, estación de trabajo y fecha de registro.

| Columna | Descripción |
|---|---|
| `FechaKey` | Fecha del documento (`Fecha_Documento`). Se relaciona con `Calendario[Date]`. |
| `IdProducto` | FK a `DIM_Producto`. |
| `IdUnidadMedida` | FK a `DIM_UnidadMedida`. |
| `IdCliente` | FK a `DIM_Cliente`. |
| `IdBodega` | FK compuesta (Empresa-Bodega) a `DIM_Bodega`. |
| `IdSerie` | FK compuesta (Tipo-Serie) a `DIM_Serie`. |
| `IdTipoTransaccion` | FK a `DIM_TipoTransaccion`. |
| `Documento` | Número/ID del documento origen (`"N/A"` si es nulo). |
| `Consecutivo` | Consecutivo interno de la transacción. |
| `VtaSinIva` | **Monto de venta sin IVA** = `monto / 1.12` (IVA de Guatemala, 12%). |
| `Cantidad` | Cantidad de unidades del movimiento. |
| `CostoTotal` | Costo total asociado al movimiento. |

**Filtros de negocio aplicados en el SP** (definen la granularidad del fact):
- `Estado_Documento = 1` → solo documentos activos/vigentes.
- `Estado = 1` y `Reversion = 0` → solo transacciones activas y no reversadas.
- `D_Tipo_Documento = 99` → solo un tipo de documento específico (filtra el universo transaccional a un subconjunto de negocio, p. ej. transacciones definitivas de inventario/ventas).
- `Fecha_Documento > '20200101'` → histórico desde el 1 de enero de 2020.

---

## 4. Relaciones del modelo en Power BI

Todas las relaciones son **1 a muchos**, dirección única (dimensión → hecho), filtrando hacia `FACT_Movimientos`:

```
Calendario[Date]                  1 → * FACT_Movimientos[FechaKey]
DIM_Producto[IdProducto]          1 → * FACT_Movimientos[IdProducto]
DIM_UnidadMedida[IdUnidadMedida]  1 → * FACT_Movimientos[IdUnidadMedida]
DIM_Cliente[IdCliente]            1 → * FACT_Movimientos[IdCliente]
DIM_Bodega[IdBodega]              1 → * FACT_Movimientos[IdBodega]
DIM_Serie[IdSerie]                1 → * FACT_Movimientos[IdSerie]
DIM_TipoTransaccion[IdTipoTransaccion] 1 → * FACT_Movimientos[IdTipoTransaccion]
```

---

## 5. Tabla `Calendario` (DAX)

```dax
Calendario =
ADDCOLUMNS (
    CALENDAR ( MIN ( FACT_Movimientos[FechaKey] ), MAX ( FACT_Movimientos[FechaKey] ) ),
    "Año", YEAR ( [Date] ),
    "Trimestre", QUARTER ( [Date] ),
    "NoMes", MONTH ( [Date] ),
    "NoDia", DAY ( [Date] ),
    "NoSemana", WEEKNUM ( [Date], 2 ),
    "Mes", FORMAT ( [Date], "MM" ),
    "Nombre Mes", FORMAT ( [Date], "MMMM" ),
    "Mes Corto", FORMAT ( [Date], "MMM" ),
    "Dia Semana", FORMAT ( [Date], "dddd" ),
    "Año Mes", FORMAT ( [Date], "YYYY-MM" ),
    "MesKey", FORMAT ( [Date], "YYYYMM" )
)
```

Genera un rango continuo de fechas entre el mínimo y máximo de `FechaKey` en el fact, con columnas de apoyo para agrupar por año, mes, trimestre, semana, etc. Debe estar marcada como **Date Table** (columna `Date`) para que funcionen correctamente las funciones de inteligencia de tiempo (`TOTALYTD`, `SAMEPERIODLASTYEAR`, `DATEADD`, etc.).

---

## 6. Tabla `Medidas` — catálogo de fórmulas DAX

Tabla vacía (`{}`) usada como contenedor organizador de todas las medidas del modelo (buena práctica: separar medidas de las tablas de datos).

### 6.1 Medidas base

| Medida | Fórmula DAX | Para qué sirve | Formato sugerido |
|---|---|---|---|
| `Total Ventas` | `SUM ( FACT_Movimientos[VtaSinIva] )` | Suma el monto de ventas sin IVA. Es la medida principal de ingresos del modelo. | Moneda |
| `Total Costo` | `SUM ( FACT_Movimientos[CostoTotal] )` | Suma el costo total de los movimientos. Base para calcular rentabilidad. | Moneda |
| `Margen Bruto` | `[Total Ventas] - [Total Costo]` | Ganancia bruta en valor absoluto (ventas menos costo). | Moneda |
| `% Margen` | `DIVIDE ( [Margen Bruto], [Total Ventas], 0 )` | Rentabilidad relativa: qué porcentaje de la venta es ganancia. `DIVIDE` evita errores de división por cero. | Porcentaje |
| `Total Cantidad` | `SUM ( FACT_Movimientos[Cantidad] )` | Suma de unidades vendidas/movidas. | Entero |
| `# Documentos` | `DISTINCTCOUNT ( FACT_Movimientos[Documento] )` | Cuenta documentos únicos (transacciones/facturas), no líneas de detalle. | Entero |
| `Ticket Promedio` | `DIVIDE ( [Total Ventas], [# Documentos], 0 )` | Valor promedio de venta por documento/factura. | Moneda |
| `Precio Promedio Unitario` | `DIVIDE ( [Total Ventas], [Total Cantidad], 0 )` | Precio de venta promedio por unidad de producto. | Moneda |
| `Costo Promedio Unitario` | `DIVIDE ( [Total Costo], [Total Cantidad], 0 )` | Costo promedio por unidad de producto. | Moneda |

### 6.2 Medidas de inteligencia de tiempo

Requieren la relación activa `Calendario[Date] → FACT_Movimientos[FechaKey]` y `Calendario` marcada como Date Table.

| Medida | Fórmula DAX | Para qué sirve |
|---|---|---|
| `Ventas YTD` | `TOTALYTD ( [Total Ventas], Calendario[Date] )` | Ventas acumuladas desde el 1 de enero hasta la fecha filtrada (*Year to Date*). |
| `Ventas MTD` | `TOTALMTD ( [Total Ventas], Calendario[Date] )` | Ventas acumuladas en lo que va del mes (*Month to Date*). |
| `Ventas QTD` | `TOTALQTD ( [Total Ventas], Calendario[Date] )` | Ventas acumuladas en lo que va del trimestre (*Quarter to Date*). |
| `Margen Bruto YTD` | `TOTALYTD ( [Margen Bruto], Calendario[Date] )` | Margen bruto acumulado en el año. |
| `Ventas Año Anterior` | `CALCULATE ( [Total Ventas], SAMEPERIODLASTYEAR ( Calendario[Date] ) )` | Trae el total de ventas del mismo período pero un año atrás, para comparar. |
| `Ventas Mes Anterior` | `CALCULATE ( [Total Ventas], DATEADD ( Calendario[Date], -1, MONTH ) )` | Trae el total de ventas del mes inmediatamente anterior. |
| `Var % YoY` | `DIVIDE ( [Total Ventas] - [Ventas Año Anterior], [Ventas Año Anterior], 0 )` | Variación porcentual de ventas respecto al mismo período del año pasado (*Year over Year*). |
| `Var % MoM` | `DIVIDE ( [Total Ventas] - [Ventas Mes Anterior], [Ventas Mes Anterior], 0 )` | Variación porcentual de ventas respecto al mes anterior (*Month over Month*). |

### 6.3 Ranking / Top N

Recomendado agrupar en la carpeta de visualización `Ranking`.

| Medida | Fórmula DAX | Para qué sirve |
|---|---|---|
| `Ranking Producto (Ventas)` | `RANKX ( ALL ( DIM_Producto ), [Total Ventas], , DESC, DENSE )` | Posición de cada producto según sus ventas totales (1 = el más vendido). Se usa en tablas con `ProductoNombre` en filas. |
| `Ranking Cliente (Ventas)` | `RANKX ( ALL ( DIM_Cliente ), [Total Ventas], , DESC, DENSE )` | Posición de cada cliente según sus compras totales. |
| `Es Top 10 Producto` | `IF ( [Ranking Producto (Ventas)] <= 10, 1, 0 )` | Bandera (1/0) para usar como filtro de nivel visual y quedarse solo con el Top 10 de productos. |
| `Ventas Top 10 Productos` | `CALCULATE ( [Total Ventas], TOPN ( 10, ALL ( DIM_Producto ), [Total Ventas], DESC ) )` | Suma de ventas concentrada únicamente en los 10 productos más vendidos; KPI de "cuánto representan mis productos estrella". |

### 6.4 Análisis ABC / Pareto 80-20

Recomendado agrupar en la carpeta de visualización `ABC-Pareto`. Clasifica productos según su aporte acumulado a las ventas totales: **A** = productos que en conjunto generan el 80% de la venta, **B** = el siguiente 15%, **C** = el 5% restante (cola larga).

| Medida | Fórmula DAX | Para qué sirve |
|---|---|---|
| `Venta Acumulada Producto` | `VAR RankActual = [Ranking Producto (Ventas)] RETURN CALCULATE ( [Total Ventas], TOPN ( RankActual, ALL ( DIM_Producto ), [Total Ventas], DESC ) )` | Suma acumulada de ventas hasta el producto actual, ordenando de mayor a menor venta. |
| `% Acumulado Producto` | `DIVIDE ( [Venta Acumulada Producto], CALCULATE ( [Total Ventas], ALL ( DIM_Producto ) ) )` | % que representa la venta acumulada hasta ese producto sobre el total general. |
| `Clasificación ABC Producto` | `SWITCH ( TRUE (), ISBLANK ( [% Acumulado Producto] ), "Sin Ventas", [% Acumulado Producto] <= 0.8, "A", [% Acumulado Producto] <= 0.95, "B", "C" )` | Etiqueta A/B/C de cada producto para priorizar control de inventario y reposición. |

⚠️ **Nota de rendimiento**: `Venta Acumulada Producto` ejecuta un `TOPN` por cada producto evaluado, por lo que con catálogos muy grandes (miles de productos) puede volverse lento en visuales con muchas filas.

### 6.5 Participación % (part-to-whole)

Recomendado agrupar en la carpeta de visualización `Participación %`.

| Medida | Fórmula DAX | Para qué sirve |
|---|---|---|
| `% Participación por Producto` | `DIVIDE ( [Total Ventas], CALCULATE ( [Total Ventas], ALL ( DIM_Producto ) ) )` | % que representa un producto sobre el total de ventas de todos los productos (respeta el filtro de fecha activo). |
| `% Participación por Clase Producto` | `DIVIDE ( [Total Ventas], CALCULATE ( [Total Ventas], ALL ( DIM_Producto[ClaseProducto] ) ) )` | % que representa una línea/clase de producto (ej. tornillería, pintura) sobre el total. |
| `% Participación por Bodega` | `DIVIDE ( [Total Ventas], CALCULATE ( [Total Ventas], ALL ( DIM_Bodega ) ) )` | % que representa una bodega/sucursal sobre el total de ventas de todas las bodegas. |
| `% Participación por Cliente` | `DIVIDE ( [Total Ventas], CALCULATE ( [Total Ventas], ALL ( DIM_Cliente ) ) )` | % que representa un cliente sobre el total de ventas de todos los clientes. |
| `% del Total Seleccionado` | `DIVIDE ( [Total Ventas], CALCULATE ( [Total Ventas], ALLSELECTED () ) )` | % de una fila/celda respecto a lo que esté visible según los segmentadores de la página (útil en totales de tabla/matriz). |

### 6.6 Nota de mantenimiento
- El campo `∑ Column` visible en la tabla `Medidas` es la columna por defecto generada al crear la tabla con `{}`; no se usa y puede eliminarse sin afectar el modelo.
- Si se agregan más medidas en el futuro, seguir el mismo patrón: crearlas en la tabla `Medidas`, con nombre descriptivo en español, formato explícito y carpeta de visualización asignada.

---

## 7. Próximos pasos sugeridos

- Medidas de clientes: recencia (última compra), clasificación Activo/En Riesgo/Inactivo, clientes nuevos vs. recurrentes por período — **pendiente por decidir el criterio de fecha de referencia** (`TODAY()` vs. fecha máxima del dataset).
- Segmentaciones (slicers) recomendadas: `Calendario[Año]`, `DIM_Bodega[EmpresaNombre]`, `DIM_TipoTransaccion[Descripcion]`, `DIM_Producto[ClaseProducto]`.
- Visuales sugeridos para las medidas de esta sección: gráfico de Pareto (barras + línea acumulada) para ABC, treemap o gráfico de dona para participación %, tabla con `Ranking Producto (Ventas)` ordenada ascendente para el Top N.

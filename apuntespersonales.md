## Resumen de técnicas SQL

| Técnica                                      | Preguntas       |
| -------------------------------------------- | --------------- |
| `WHERE`, `BETWEEN`, `IN`, `LIKE`             | 1, 3, 4         |
| `NULL` y `COALESCE()`                        | 3, 7, 9, 10, 19 |
| Funciones de agregación y `ROUND()`          | 1, 2, 6, 14     |
| `GROUP BY` y `HAVING`                        | 2, 6            |
| `INNER JOIN`, alias, `USING`                 | 4, 5, 6         |
| `LEFT JOIN`                                  | 7, 9            |
| `SELF JOIN`                                  | 8               |
| `CROSS JOIN`                                 | 9               |
| `FULL JOIN`                                  | 10              |
| `UNION ALL`                                  | 11              |
| `EXCEPT`, `INTERSECT`                        | 12              |
| Semi join y anti join                        | 12, 13          |
| Subconsulta en `WHERE`                       | 13, 14, 16      |
| Subconsulta en `SELECT`                      | 14, 16          |
| Subconsulta en `FROM`                        | 15              |
| Subconsultas correlacionadas                 | 13, 16          |
| CTE (`WITH`)                                 | 17, 18, 19, 20  |
| `CASE WHEN`                                  | 3, 10, 17, 20   |
| `NTILE()`                                    | 17              |
| `RANK()` / `ROW_NUMBER()`                    | 18              |
| `PARTITION BY`                               | 18              |
| Marcos de ventana (`ROWS BETWEEN`)           | 19              |
| `LAG()`                                      | 19              |
| Pivotado y `ROLLUP`                          | 20              |
| Funciones de texto                           | 11              |
| Funciones de fecha (`EXTRACT`, `DATE_TRUNC`) | 9, 19, 20       |


# Explicaciones según pregunta

## Pregunta 2 — Concentración geográfica de la cartera

**Explicación paso a paso:**

* `SELECT pais, ...`: Selecciona el país y calcula las métricas solicitadas con los alias requeridos (`num_clientes` y `num_ciudades`).
* `COUNT(*)`: Cuenta el total de filas (clientes) agrupadas por cada país.
* `COUNT(DISTINCT ciudad)`: Cuenta únicamente las ciudades únicas, sin repetir, que hay dentro de cada país.
* `GROUP BY pais`: Agrupa los registros por cada país distinto.
* `HAVING COUNT(*) >= 5`: Filtra los grupos resultantes para mostrar solo los países que tienen 5 o más clientes. Se utiliza `HAVING` porque el filtro actúa sobre el resultado de una función de agregación como `COUNT`.
* `ORDER BY num_clientes DESC`: Ordena los resultados de mayor a menor cantidad de clientes.

---

## Pregunta 3 — Alerta de reposición

**Explicación de los elementos clave:**

* `FROM products`: Indica que la consulta trabaja sobre la tabla de productos.
* `WHERE discontinued = 0`: Filtra para obtener únicamente los productos activos. En Northwind, un valor `0` en `discontinued` significa que el producto sigue en catálogo.
* `AND units_in_stock <= reorder_level`: Filtra aquellos productos cuyo stock actual es menor o igual al nivel mínimo de reposición.
* `CASE WHEN ... END AS situacion`: Evalúa las unidades en stock. Si el stock es exactamente `0`, muestra `'CRÍTICO'`; para cualquier otro caso dentro de los productos filtrados, muestra `'AVISO'`.
* **Alias:** Los nombres `producto`, `stock`, `nivel_reposicion`, `pedido_a_proveedor` y `situacion` corresponden a las columnas solicitadas.

### ¿Por qué se llama `situacion`?

Por dos razones principales:

1. **Por el enunciado del ejercicio:** La pregunta pide explícitamente que una de las columnas se llame `situacion`.

2. **Por el funcionamiento de `AS`:** El bloque `CASE ... END` genera una columna calculada dinámicamente. Como no es una columna original de la tabla, no tiene un nombre propio. `AS situacion` crea un alias o etiqueta para esa nueva columna.

Esencialmente, `AS situacion` le indica a la base de datos que la columna resultante de evaluar el `CASE` debe aparecer con ese nombre en la tabla final.

---

## Pregunta 4 — Ficha completa de producto

**Explicación paso a paso:**

### Alias de tabla (`p`, `c`, `s`)

Permiten abreviar y referenciar de forma limpia las tablas `products`, `categories` y `suppliers`, evitando ambigüedades en las columnas.

### `INNER JOIN`

* Conecta los productos con sus respectivas categorías mediante `category_id`.
* Conecta los productos con los datos de sus proveedores mediante `supplier_id`.

### `WHERE s.country IN ('Italy', 'France', 'Spain')`

Filtra únicamente los proveedores cuyos países sean Italia, Francia o España.

> En la base de datos Northwind los nombres de los países suelen estar almacenados en inglés. Si en tu base de datos están traducidos al español, habría que utilizar `'Italia'`, `'Francia'` y `'España'`.

### `ORDER BY s.country ASC, p.product_name ASC`

Ordena primero alfabéticamente por el país del proveedor y, dentro de cada país, por el nombre del producto de forma ascendente.

---

## Pregunta 5 — Detalle valorizado de un pedido

**Explicación de los detalles clave:**

### `USING (order_id)` y `USING (product_id)`

Como las columnas tienen exactamente el mismo nombre en las tablas relacionadas, `USING` permite evitar escribir la condición completa:

```sql
ON tabla1.id = tabla2.id
```

Además, evita duplicar las columnas utilizadas para la unión en el resultado.

### Cálculo de `importe_linea`

Multiplica el precio unitario por la cantidad y aplica el descuento:

```text
precio × cantidad × (1 - descuento)
```

### `ROUND(..., 2)` y `::numeric`

Los precios se convierten temporalmente a `numeric` mediante `::numeric` para poder redondear correctamente a dos decimales con `ROUND()`.

### `WHERE o.order_id = 10248`

Filtra estrictamente para mostrar únicamente las líneas correspondientes al pedido `10248`.

---

## Pregunta 6 — Ranking de categorías por facturación

**Explicación de los puntos clave:**

### Unión de tres tablas

Los `INNER JOIN` relacionan:

* `categories` con `products` mediante `category_id`.
* `products` con `order_details` mediante `product_id`.

### `COUNT(*)`

Cuenta cuántas líneas de pedido se han registrado para cada categoría (`num_lineas`).

### `COUNT(DISTINCT p.product_id)`

Cuenta cuántos productos diferentes de cada categoría se han vendido (`num_productos`).

### Cálculo de la facturación

```sql
SUM(od.unit_price * od.quantity * (1 - od.discount))
```

Calcula la facturación real aplicando el precio, la cantidad y el descuento de cada línea y sumando posteriormente los importes por categoría.

El resultado se redondea a dos decimales mediante `ROUND(..., 2)` y `::numeric`.

### El detalle del `HAVING`

En PostgreSQL no se puede utilizar el alias `facturacion` dentro del `HAVING`. Por eso es necesario repetir la fórmula completa:

```sql
HAVING SUM(od.unit_price * od.quantity * (1 - od.discount)) > 100000
```

Así se filtran únicamente las categorías cuya facturación supera los `100.000`.

En cambio, `ORDER BY` sí permite utilizar el alias `facturacion`.

---

## Pregunta 7 — Clientes sin actividad comercial

**Explicación detallada de las técnicas utilizadas:**

### `LEFT JOIN`

Garantiza que se listen todos los registros de la tabla de la izquierda (`customers`), incluso si no encuentran una coincidencia en `orders`.

### `COUNT(o.order_id)` frente a `COUNT(*)`

Si se utilizara `COUNT(*)`, la fila generada por el `LEFT JOIN` para un cliente sin pedidos podría contabilizarse como una fila.

Al utilizar:

```sql
COUNT(o.order_id)
```

los valores `NULL` se ignoran y los clientes sin pedidos obtienen correctamente un `0`.

### `MAX(o.order_date)`

Obtiene la fecha del pedido más reciente de cada cliente.

### `COALESCE(..., 'SIN PEDIDOS')`

Si `MAX(o.order_date)` devuelve `NULL` porque el cliente no tiene pedidos, `COALESCE()` lo sustituye por `'SIN PEDIDOS'`.

Como el resultado mezcla fechas y texto, se utiliza `::text` para convertir la fecha a texto.

### `ORDER BY num_pedidos ASC`

Ordena de menor a mayor cantidad de pedidos, haciendo que los clientes inactivos aparezcan primero.

---

## Pregunta 8 — Organigrama de la fuerza de ventas

**Explicación paso a paso:**

### Alias `emp` y `jefe`

Como se consulta dos veces la misma tabla (`employees`), los alias son necesarios para distinguir las dos referencias.

* `emp`: empleado.
* `jefe`: supervisor directo.

### `LEFT JOIN`

Es fundamental porque el director general no tiene a nadie por encima y, por tanto, su campo `reports_to` es `NULL`.

El `LEFT JOIN` garantiza que ese empleado no desaparezca del resultado.

### Concatenación de texto (`||`)

Une `first_name` y `last_name` separados por un espacio:

```sql
first_name || ' ' || last_name
```

De esta forma se obtiene el nombre completo del empleado y del responsable.

### `COALESCE(..., 'DIRECCIÓN GENERAL')`

Si el responsable es `NULL`, `COALESCE()` sustituye ese valor por `'DIRECCIÓN GENERAL'`, tanto para el nombre del responsable como para su cargo.

---

## Pregunta 9 — Rejilla de cobertura categoría × año

**Explicación detallada de la estrategia:**

### `anos`

Esta CTE extrae dinámicamente los años distintos existentes en `orders` utilizando:

```sql
EXTRACT(YEAR FROM order_date)
```

### `grid` — El esqueleto con `CROSS JOIN`

Cruza todas las categorías de `categories` con todos los años obtenidos.

Esto genera las combinaciones posibles entre categorías y años, garantizando que ninguna combinación quede fuera aunque no tenga ventas.

### `ventas_reales`

Calcula la facturación real agrupando los importes de los detalles de los pedidos por categoría y año.

### Unión final: `LEFT JOIN` + `COALESCE()`

Se parte de la rejilla completa (`grid`) y se realiza un `LEFT JOIN` con las ventas reales.

Si una categoría no tuvo ventas en un año determinado, el `LEFT JOIN` devuelve `NULL`.

`COALESCE(..., 0)` detecta ese `NULL` y lo sustituye por `0`, evitando dejar huecos en el resultado.

Finalmente, `ROUND(..., 2)` muestra las cifras con dos decimales.

---

# Explicación de CTE

Una **CTE** (*Common Table Expression* o Expresión de Tabla Común) es básicamente una consulta temporal con nombre que se crea justo antes de ejecutar la consulta principal.

Se define utilizando la palabra clave `WITH`.

Puede entenderse como un paso intermedio donde guardamos un resultado para utilizarlo posteriormente, sin necesidad de crear una tabla real en la base de datos. La CTE existe durante la ejecución de la consulta.

### ¿Para qué sirve?

**Ordenar consultas complejas:**
En lugar de crear una consulta gigante con muchas subconsultas anidadas, se divide el problema en varios pasos lógicos.

**Reutilizar lógica:**
Permite calcular determinados datos y utilizarlos posteriormente como base para otras operaciones.

**Mejorar la legibilidad:**
Cada bloque tiene una responsabilidad concreta, haciendo que la consulta sea más fácil de entender y mantener.

### ¿Cómo se estructura?

La sintaxis básica es:

```sql
WITH nombre_de_la_cte AS (
    -- Consulta que genera el resultado temporal
    SELECT columna1, columna2
    FROM tabla
    WHERE condicion = true
)
-- Consulta principal
SELECT *
FROM nombre_de_la_cte
WHERE columna1 > 10;
```

### Ejemplo con la Pregunta 9

En lugar de mezclar toda la lógica en una única consulta, se utilizaron tres CTE:

1. **`anos`**: obtiene los años.
2. **`grid`**: genera todas las combinaciones categoría × año.
3. **`ventas_reales`**: calcula las ventas reales.

Finalmente, un `SELECT` combina estos resultados de forma ordenada.

---

## Pregunta 10 — Mapa de países: clientes frente a proveedores

**Explicación detallada de la estrategia:**

### CTE `clientes_pais` y `proveedores_pais`

Calculan por separado cuántos clientes y proveedores existen agrupados por país.

### `FULL JOIN`

Cruza ambos listados.

Si un país solo existe entre los clientes o solo entre los proveedores, el `FULL JOIN` garantiza que la fila no se pierda.

### `COALESCE(c.pais, p.pais)`

Cuando un país solo aparece en uno de los dos listados, el otro lado del `FULL JOIN` tendrá `NULL`.

`COALESCE()` utiliza el país disponible para garantizar que el nombre siempre aparezca.

### `COALESCE(..., 0)` en los conteos

Convierte los valores `NULL` producidos por el `FULL JOIN` en `0`.

### `CASE WHEN`

Clasifica cada país según su presencia:

* `AMBOS`
* `SOLO CLIENTES`
* `SOLO PROVEEDORES`

---

## Pregunta 11 — Directorio unificado de contactos

### `UNION ALL`

Une los resultados de las tres consultas, apilándolos unos encima de otros.

A diferencia de `UNION`, `UNION ALL` no elimina duplicados.

Esto garantiza que todos los contactos aparezcan aunque dos filas tengan exactamente los mismos datos.

### Literales como columna

Expresiones como:

```sql
'CLIENTE' AS origen
```

o:

```sql
'NORTHWIND TRADERS' AS organizacion
```

insertan un texto fijo en cada fila correspondiente.

### `UPPER(...)`

Convierte los nombres de contactos y organizaciones a mayúsculas.

### Concatenación en empleados

Une el nombre y apellido mediante `||` para mostrar el nombre completo.

---

## Pregunta 12 — Mercados con desequilibrio

### `EXCEPT`

Funciona como una resta de conjuntos.

Devuelve únicamente los elementos que aparecen en el primer `SELECT` pero no en el segundo.

### `INTERSECT`

Devuelve exclusivamente los elementos que aparecen en ambas consultas.

### Eliminación de duplicados

A diferencia de `UNION ALL`, los operadores de conjunto `UNION`, `EXCEPT` e `INTERSECT` eliminan duplicados automáticamente.

Por tanto, un país solo aparecerá una vez aunque existan varios clientes o proveedores de ese país.

---

## Pregunta 13 — Clientes que nunca han comprado pescado

**Explicación detallada de la estrategia:**

### Anti Join con `NOT EXISTS`

La cláusula `NOT EXISTS` evalúa la subconsulta interna.

* Si encuentra que el cliente ha comprado al menos un producto de la categoría `'Seafood'`, ese cliente queda excluido.
* Si la subconsulta no devuelve ninguna fila para ese cliente, se mantiene en el resultado.

### Subconsulta correlacionada

La condición:

```sql
WHERE o2.customer_id = c.customer_id
```

conecta la consulta interna con la fila actual de la consulta externa.

De esta forma, la subconsulta se evalúa teniendo en cuenta el cliente actual.

### Recuento de pedidos

El `LEFT JOIN` con `orders` permite calcular todos los pedidos del cliente:

```sql
COUNT(o.order_id)
```

Después se ordenan de mayor a menor mediante:

```sql
ORDER BY pedidos_realizados DESC
```

### `NOT IN` frente a `NOT EXISTS`

`NOT EXISTS` es especialmente útil para los anti joins porque evita problemas derivados de valores `NULL`.

Con `NOT IN`, si la subconsulta devuelve un `NULL`, la lógica de comparación puede producir resultados inesperados debido a la lógica ternaria de SQL.

Por ello, `NOT EXISTS` suele ser una opción más robusta para este tipo de consultas.

---

## Pregunta 14 — Productos por encima de la media

**Explicación detallada de la estrategia:**

### Subconsulta escalar

```sql
SELECT AVG(unit_price)
FROM products
```

Devuelve un único valor: la media global de los precios del catálogo.

Al devolver un solo número, puede utilizarse directamente dentro de operaciones y condiciones.

### Filtro en `WHERE`

```sql
discontinued = 0
```

Asegura que solo se evalúen productos activos.

```sql
unit_price > (...)
```

Compara el precio de cada producto con la media general.

### Columnas calculadas y redondeo

Se muestran el precio del producto y la media general utilizando `ROUND(..., 2)` y `::numeric`.

La diferencia se calcula mediante:

```sql
unit_price - AVG(unit_price)
```

y se ordena de mayor a menor mediante:

```sql
ORDER BY diferencia DESC
```

### Consideración sobre rendimiento

Repetir la misma subconsulta varias veces funciona correctamente, pero en consultas más pesadas podría ser menos eficiente.

Una alternativa sería utilizar una CTE (`WITH`) para calcular la media una sola vez y reutilizarla.

---

## Pregunta 15 — Ticket medio por cliente

**Explicación detallada de la estrategia:**

### Nivel 1 — Subconsulta en `FROM`

Primero es necesario calcular cuánto vale cada pedido individualmente.

La subconsulta interna une `orders` y `order_details` y agrupa por `order_id` para calcular:

```text
importe_pedido
```

La subconsulta recibe el alias `pedidos`.

Este alias es obligatorio en PostgreSQL para una tabla derivada utilizada en `FROM`.

### Nivel 2 — Consulta externa

La consulta externa toma los pedidos ya calculados y los une con `customers` para recuperar el nombre y país del cliente.

Después vuelve a agrupar por cliente.

* `COUNT(pedidos.order_id)`: Cuenta los pedidos realizados.
* `SUM(pedidos.importe_pedido)`: Calcula la facturación total.
* `AVG(pedidos.importe_pedido)`: Calcula el ticket medio por pedido.
* `LIMIT 15`: Muestra únicamente los 15 clientes con mayor ticket medio.

---

## Pregunta 16 — El producto más caro de cada categoría

**Explicación detallada y concepto de correlación:**

### ¿Qué es una subconsulta correlacionada?

A diferencia de una subconsulta normal, una subconsulta correlacionada depende de los valores de la fila actual de la consulta externa.

En este caso, hace referencia a:

```sql
p.category_id
```

dentro de la propia subconsulta.

### Funcionamiento paso a paso

1. PostgreSQL toma un producto de la consulta principal (`p`).
2. Comprueba a qué categoría pertenece mediante `p.category_id`.
3. Ejecuta la subconsulta interna filtrando los productos de esa misma categoría.
4. Obtiene el precio máximo mediante `MAX()`.
5. Si el precio del producto coincide con ese máximo, el producto se mantiene.
6. Si no coincide, se descarta.

La segunda subconsulta correlacionada funciona de forma similar para calcular el precio medio de la categoría.

### Consideración sobre rendimiento

Conceptualmente, este enfoque evalúa la subconsulta para cada fila de la consulta externa.

En una base de datos pequeña como Northwind el coste es reducido, pero con diez millones de filas esta estrategia puede resultar costosa.

En escenarios de gran volumen, podrían utilizarse alternativas como funciones de ventana (`RANK()`, `ROW_NUMBER()`) o agregaciones previas.

---

## Pregunta 17 — Segmentación ABC de la cartera de clientes

**Explicación detallada de la estrategia por bloques:**

### `facturacion_clientes` — Cálculo base

Agrupa y calcula la facturación histórica de cada cliente aplicando precios, cantidades y descuentos.

Se utiliza `LEFT JOIN` para conservar también clientes sin compras y `COALESCE()` para representar su facturación como `0`.

### `cuartiles` — Distribución con `NTILE()`

Utiliza:

```sql
NTILE(4) OVER (ORDER BY facturacion_total DESC)
```

para dividir los clientes en cuatro grupos.

El cuartil `1` contiene los clientes con mayor facturación, mientras que el cuartil `4` contiene los de menor facturación.

### `segmentacion` — Asignación con `CASE`

Traduce los números de los cuartiles a etiquetas de negocio:

* `1` → `A - Estratégico`
* `2` → `B - Consolidado`
* `3` → `C - Ocasional`
* `4` → `D - Marginal`

### Consulta final

Agrupa los clientes por segmento y calcula:

* Número de clientes.
* Facturación total.
* Porcentaje sobre la facturación total de la compañía.

Finalmente, ordena los segmentos alfabéticamente de `A` a `D`.

---

## Pregunta 18 — Los tres productos más vendidos de cada categoría

**Explicación detallada de la estrategia:**

### CTE `metricas_productos`

Primero se unen `categories`, `products` y `order_details`.

Después se agrupa por categoría y producto para calcular:

* Unidades vendidas.
* Facturación.

### Funciones de ventana

#### Ranking por categoría

```sql
RANK() OVER (
    PARTITION BY c.category_name
    ORDER BY ... DESC
)
```

`PARTITION BY` divide los datos en grupos independientes por categoría.

El ranking comienza desde `1` dentro de cada categoría.

#### Ranking global

```sql
RANK() OVER (
    ORDER BY ... DESC
)
```

Al no utilizar `PARTITION BY`, el ranking se calcula comparando todos los productos de la compañía.

Esto permite ver si un producto es líder dentro de su categoría pero tiene una posición más baja en el conjunto global.

### Filtrado exterior

```sql
WHERE posicion_en_categoria <= 3
```

La función de ventana se calcula dentro de la CTE y posteriormente se filtra desde la consulta exterior.

Esto es necesario porque las funciones de ventana no pueden utilizarse directamente en el `WHERE` de la misma consulta donde se calculan.

### `RANK()` frente a `DENSE_RANK()` y `ROW_NUMBER()`

**`RANK()`**

Si dos productos empatan en una posición, ambos reciben el mismo número y la siguiente posición deja un hueco.

Ejemplo:

```text
1
2
2
4
```

**`DENSE_RANK()`**

También comparte posiciones en los empates, pero no deja huecos:

```text
1
2
2
3
```

**`ROW_NUMBER()`**

Asigna un número secuencial a cada fila:

```text
1
2
3
4
```

No comparte posiciones aunque exista un empate.

---

## Pregunta 19 — Evolución mensual con acumulado y media móvil

**Explicación detallada de las técnicas utilizadas:**

### `DATE_TRUNC('month', o.order_date)::date`

Normaliza cualquier fecha al primer día de su mes.

Por ejemplo, cualquier pedido de enero de 1997 pasa a representarse como:

```text
1997-01-01
```

Esto permite agrupar y ordenar cronológicamente los datos.

El filtro:

```sql
EXTRACT(YEAR FROM o.order_date) = 1997
```

limita los resultados al año solicitado.

### Total acumulado

```sql
SUM(facturacion) OVER (
    ORDER BY mes
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

Va sumando la facturación desde enero hasta el mes actual.

### Media móvil de tres meses

```sql
AVG(facturacion) OVER (
    ORDER BY mes
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
)
```

Indica que se deben utilizar la fila actual y las dos filas anteriores.

Por tanto:

* Enero: solo enero.
* Febrero: enero + febrero.
* Marzo en adelante: los tres últimos meses.

### `LAG()`

```sql
LAG(facturacion, 1) OVER (ORDER BY mes)
```

Obtiene la facturación del mes inmediatamente anterior.

En enero devuelve `NULL`, ya que no existe un mes anterior dentro del período analizado.

### Variación porcentual y `NULLIF()`

La variación se calcula como:

```text
(facturación actual - facturación anterior)
× 100
÷ facturación anterior
```

`NULLIF(..., 0)` evita una división por cero en caso de que la facturación del mes anterior fuera `0`.

---

## Pregunta 20 — Cuadro de mando anual por categoría

**Explicación detallada de la estrategia:**

### Pivotado moderno con `FILTER`

En lugar de utilizar:

```sql
SUM(CASE WHEN anio = 1997 THEN importe ELSE 0 END)
```

se utiliza la sintaxis específica de PostgreSQL:

```sql
SUM(importe) FILTER (WHERE anio = 1997)
```

Esto permite transformar las filas correspondientes a cada año en columnas independientes de forma más clara.

### Fila de totales con `ROLLUP`

```sql
GROUP BY ROLLUP (categoria)
```

añade automáticamente una fila con los totales generales de todas las categorías.

### `COALESCE(categoria, 'TOTAL GENERAL')`

El `ROLLUP` genera `NULL` para la categoría de la fila total.

`COALESCE()` sustituye ese `NULL` por:

```text
TOTAL GENERAL
```

### Peso porcentual

La función de ventana:

```sql
SUM(SUM(total)) OVER ()
```

permite obtener la facturación global de todas las categorías.

Después se divide la facturación de cada categoría entre ese total para obtener el porcentaje que representa.

### Tendencia y comentario crítico

El `CASE` compara la facturación de 1997 con la de 1998 y muestra:

* `CRECE`
* `DECRECE`
* `ESTABLE`

Sin embargo, esta comparación **no es estrictamente comparable desde el punto de vista empresarial**, porque 1998 solo contiene datos hasta mayo.

Por tanto, para una comparación real habría que normalizar los períodos o comparar períodos homogéneos.

### Ordenación inteligente

```sql
ORDER BY (categoria IS NULL) ASC, total DESC
```

Hace que las categorías aparezcan ordenadas por facturación total y mantiene la fila `TOTAL GENERAL` al final.



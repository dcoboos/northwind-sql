## Pregunta 1 — Catálogo comercial activo

**Enunciado:** El equipo de ventas prepara la tarifa de la próxima campaña y necesita el catálogo depurado. Obtén los productos que **no** están descatalogados y cuyo precio unitario esté entre 10 y 50 euros, ambos incluidos. Muestra el nombre del producto y su precio redondeado a dos decimales, ordenado de mayor a menor precio.

**Consulta:**

```sql
SELECT product_name,
       ROUND(unit_price::numeric, 2) AS precio
FROM products
WHERE discontinued = 0
  AND ROUND(unit_price::numeric, 2) BETWEEN 10 AND 50
ORDER BY unit_price DESC;
```

**Resultado:**

[Resultado pregunta 1](images/1.1.png)

**Comentario:** He utilizado `discontinued = 0` para obtener únicamente los productos que no están descatalogados. Además, he utilizado `BETWEEN 10 AND 50` para limitar el precio al rango indicado, incluyendo ambos valores. Finalmente, `ROUND` permite mostrar el precio redondeado a dos decimales y `ORDER BY unit_price DESC` ordena los productos de mayor a menor precio.

## Pregunta 2 — Concentración geográfica de la cartera

**Enunciado:** Dirección quiere saber en qué mercados está realmente concentrada la base de clientes antes de decidir dónde abrir delegación. Cuenta cuántos clientes hay en cada país y muestra únicamente aquellos países con **5 o más clientes**, ordenados de mayor a menor. Indica también cuántas ciudades distintas hay en cada uno de esos países.

**Consulta:**

```sql
SELECT 
    country AS pais, 
    COUNT(*) AS num_clientes, 
    COUNT(DISTINCT city) AS num_ciudades
FROM customers
GROUP BY country
HAVING COUNT(*) >= 5
ORDER BY num_clientes DESC;
```

**Resultado:**

[Resultado pregunta 2](images/2.1.png)

**Comentario:** He utilizado `GROUP BY country` para agrupar los clientes por país. Con `COUNT(*)` obtengo el número de clientes de cada país y con `COUNT(DISTINCT city)` el número de ciudades diferentes. Utilizo `HAVING COUNT(*) >= 5` porque la condición se aplica sobre el resultado del conteo después de agrupar. Finalmente, `ORDER BY num_clientes DESC` muestra primero los países con más clientes.

## Pregunta 3 — Alerta de reposición

**Enunciado:** Logística necesita detectar qué referencias están en riesgo de rotura de stock. Localiza los productos activos cuyas unidades en stock sean **inferiores o iguales** a su nivel de reposición. Muestra el nombre, las unidades en stock, el nivel de reposición, las unidades ya pedidas al proveedor y una columna de texto que indique `'CRÍTICO'` cuando el stock sea 0 y `'AVISO'` en el resto de casos.

**Consulta:**

```sql
SELECT 
    product_name AS producto,
    units_in_stock AS stock,
    reorder_level AS nivel_reposicion,
    units_on_order AS pedido_a_proveedor,
    CASE 
        WHEN units_in_stock = 0 THEN 'CRÍTICO'
        ELSE 'AVISO'
    END AS situacion
FROM products
WHERE discontinued = 0
  AND units_in_stock <= reorder_level;
```

**Resultado:**

[Resultado pregunta 3](images/3.1.png)

**Comentario:** He utilizado `WHERE` para obtener únicamente los productos activos cuyo stock sea menor o igual al nivel de reposición. Con `CASE WHEN` creo la columna `situacion`, mostrando `'CRÍTICO'` cuando el stock es 0 y `'AVISO'` para el resto de productos que necesitan reposición.

## Pregunta 4 — Ficha completa de producto

**Enunciado:** Marketing va a rehacer el catálogo impreso y necesita cada producto con su categoría y los datos de contacto de quien lo suministra. Para los productos suministrados por empresas de **Italia, Francia o España**, muestra el nombre del producto, el nombre de la categoría, el nombre del proveedor, su país y su ciudad. Ordena por país y, dentro de cada país, por nombre de producto.

**Consulta:**

```sql
SELECT 
    p.product_name AS producto,
    c.category_name AS categoria,
    s.company_name AS proveedor,
    s.country AS pais,
    s.city AS ciudad
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
INNER JOIN suppliers s ON p.supplier_id = s.supplier_id
WHERE s.country IN ('Italy', 'France', 'Spain')
ORDER BY s.country ASC, p.product_name ASC;
```

**Resultado:**

[Resultado pregunta 4](images/4.1.png)

**Comentario:** He utilizado dos `INNER JOIN` para relacionar las tablas `products`, `categories` y `suppliers`. La tabla `suppliers` es la que contiene la información del país y la ciudad del proveedor. Con `WHERE ... IN` filtro únicamente los proveedores de Italia, Francia y España. Finalmente, ordeno primero por país y después por nombre de producto.

## Pregunta 5 — Detalle valorizado de un pedido

**Enunciado:** Atención al cliente recibe una reclamación sobre el pedido **10248** y necesita reconstruir la factura línea a línea. Muestra, para ese pedido, el nombre del producto, el precio unitario aplicado, la cantidad, el descuento y el importe final de cada línea. Añade el nombre del cliente y la fecha del pedido.

**Consulta:**

```sql id="4jq7gk"
SELECT 
    c.company_name AS cliente,
    o.order_date AS fecha_pedido,
    p.product_name AS producto,
    od.unit_price AS precio_unitario,
    od.quantity AS cantidad,
    od.discount AS descuento,
    ROUND((od.unit_price * od.quantity * (1 - od.discount))::numeric, 2) AS importe_linea
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
INNER JOIN order_details od USING (order_id)
INNER JOIN products p USING (product_id)
WHERE o.order_id = 10248;
```

**Resultado:**

[Resultado pregunta 5](images/5.1.png)

**Comentario:** He utilizado `INNER JOIN` para relacionar el pedido con el cliente, sus líneas de detalle y los productos. En las tablas que comparten el mismo nombre de columna he utilizado `USING`, tal como indica el enunciado. Para calcular el importe de cada línea multiplico el precio por la cantidad y aplico el descuento. Finalmente, `ROUND()` redondea el resultado a dos decimales y filtro únicamente el pedido `10248`.

## Pregunta 7 — Clientes sin actividad comercial

**Enunciado:** Dirección comercial sospecha que hay cuentas abiertas que nunca han llegado a comprar. Lista **todos** los clientes con el número de pedidos que ha realizado cada uno y la fecha de su último pedido. Los clientes sin ningún pedido deben aparecer igualmente, con un 0 en el conteo y el texto `'SIN PEDIDOS'` en lugar de la fecha. Ordena de forma que los clientes inactivos aparezcan primero.

**Consulta:**

```sql id="m2x8qp"
SELECT 
    c.company_name AS cliente,
    c.country AS pais,
    COUNT(o.order_id) AS num_pedidos,
    COALESCE(MAX(o.order_date)::text, 'SIN PEDIDOS') AS ultimo_pedido
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.company_name, c.country
ORDER BY num_pedidos ASC;
```

**Resultado:**

[Resultado pregunta 7](images/7.1.png)

**Comentario:** He utilizado `LEFT JOIN` para incluir todos los clientes, incluso aquellos que no tienen ningún pedido. Con `COUNT(o.order_id)` cuento únicamente los pedidos existentes, ya que los valores `NULL` se ignoran. `MAX()` obtiene la fecha del último pedido y `COALESCE()` sustituye el valor `NULL` por `'SIN PEDIDOS'` para los clientes que nunca han comprado. Finalmente, ordeno por el número de pedidos de forma ascendente para que los clientes inactivos aparezcan primero.

## Pregunta 8 — Organigrama de la fuerza de ventas

**Enunciado:** Recursos Humanos necesita el organigrama del departamento comercial en formato tabla. Muestra cada empleado con su nombre completo, su cargo, el nombre completo de la persona a la que reporta y el cargo de esa persona. El empleado que no reporta a nadie debe aparecer también, con el texto `'DIRECCIÓN GENERAL'` en el campo del responsable.

**Consulta:**

```sql id="f4r8vn"
SELECT 
    emp.first_name || ' ' || emp.last_name AS empleado,
    emp.title AS cargo,
    COALESCE(jefe.first_name || ' ' || jefe.last_name, 'DIRECCIÓN GENERAL') AS responsable,
    COALESCE(jefe.title, 'DIRECCIÓN GENERAL') AS cargo_responsable
FROM employees emp
LEFT JOIN employees jefe 
    ON emp.reports_to = jefe.employee_id;
```

**Resultado:**

[Resultado pregunta 8](images/8.1.png)

**Comentario:** He utilizado un `SELF JOIN` mediante `LEFT JOIN` para relacionar la tabla `employees` consigo misma. El alias `emp` representa al empleado y `jefe` representa a la persona a la que reporta. La concatenación `||` permite obtener el nombre completo y `COALESCE()` muestra `'DIRECCIÓN GENERAL'` cuando el empleado no tiene responsable.

## Pregunta 9 — Rejilla de cobertura categoría × año

**Enunciado:** Control de gestión quiere una rejilla completa de facturación por categoría y año, **sin huecos**: si una categoría no vendió nada en un año concreto, debe aparecer con un 0, no desaparecer de la tabla. Genera todas las combinaciones posibles de las 8 categorías con los 3 años del histórico (24 filas) y asocia a cada combinación su facturación. Ordena por categoría y año.

**Consulta:**

```sql id="x7n2pm"
WITH anos AS (
    SELECT DISTINCT EXTRACT(YEAR FROM order_date)::integer AS anio
    FROM orders
),
grid AS (
    SELECT c.category_name AS categoria, y.anio
    FROM categories c
    CROSS JOIN anos y
),
ventas_reales AS (
    SELECT 
        c.category_name AS categoria,
        EXTRACT(YEAR FROM o.order_date)::integer AS anio,
        SUM(od.unit_price * od.quantity * (1 - od.discount)) AS facturacion
    FROM categories c
    JOIN products p USING (category_id)
    JOIN order_details od USING (product_id)
    JOIN orders o USING (order_id)
    GROUP BY c.category_name, EXTRACT(YEAR FROM o.order_date)
)
SELECT 
    g.categoria,
    g.anio,
    COALESCE(ROUND(vr.facturacion::numeric, 2), 0) AS facturacion
FROM grid g
LEFT JOIN ventas_reales vr 
    ON g.categoria = vr.categoria 
    AND g.anio = vr.anio
ORDER BY g.categoria ASC, g.anio ASC;
```

**Resultado:**

[Resultado pregunta 9](images/9.1.png)

**Comentario:** He utilizado `CROSS JOIN` para generar todas las combinaciones posibles entre las categorías y los años. Después, mediante `LEFT JOIN`, relaciono esta rejilla con las ventas reales. `COALESCE()` sustituye por 0 las combinaciones que no tienen facturación. Por último, `EXTRACT()` permite obtener el año de cada pedido y `ROUND()` redondea la facturación a dos decimales.

## Pregunta 10 — Mapa de países: clientes frente a proveedores

**Enunciado:** Expansión internacional quiere una única tabla que muestre, para cada país en el que la compañía tiene presencia, cuántos clientes y cuántos proveedores hay. Deben aparecer los países que solo tienen clientes, los que solo tienen proveedores y los que tienen ambos.

**Consulta:**

```sql id="r6tq1m"
WITH clientes_pais AS (
    SELECT country AS pais, COUNT(*) AS num_clientes
    FROM customers
    GROUP BY country
),
proveedores_pais AS (
    SELECT country AS pais, COUNT(*) AS num_proveedores
    FROM suppliers
    GROUP BY country
)
SELECT 
    COALESCE(c.pais, p.pais) AS pais,
    COALESCE(c.num_clientes, 0) AS num_clientes,
    COALESCE(p.num_proveedores, 0) AS num_proveedores,
    CASE 
        WHEN COALESCE(c.num_clientes, 0) > 0 
             AND COALESCE(p.num_proveedores, 0) > 0 THEN 'AMBOS'
        WHEN COALESCE(c.num_clientes, 0) > 0 THEN 'SOLO CLIENTES'
        ELSE 'SOLO PROVEEDORES'
    END AS tipo_presencia
FROM clientes_pais c
FULL JOIN proveedores_pais p ON c.pais = p.pais
ORDER BY pais ASC;
```

**Resultado:**

[Resultado pregunta 10](images/10.1.png)

**Comentario:** He utilizado dos subconsultas para obtener el número de clientes y proveedores por país. Después, `FULL JOIN` permite incluir los países que aparecen en cualquiera de las dos tablas. `COALESCE()` evita valores nulos y permite mostrar 0 cuando un país no tiene clientes o proveedores. Finalmente, `CASE WHEN` clasifica cada país como `'SOLO CLIENTES'`, `'SOLO PROVEEDORES'` o `'AMBOS'`.

## Pregunta 11 — Directorio unificado de contactos

**Enunciado:** Sistemas va a migrar el CRM y necesita una exportación única con todos los contactos de la compañía, vengan de donde vengan. Construye una sola tabla que reúna los contactos de clientes, los de proveedores y los empleados. Cada fila debe indicar el origen (`'CLIENTE'`, `'PROVEEDOR'`, `'EMPLEADO'`), el nombre de la persona de contacto **en mayúsculas**, la organización a la que pertenece, la ciudad y el país. Para los empleados, la organización es el literal `'NORTHWIND TRADERS'` y el nombre de contacto se forma concatenando nombre y apellidos.

**Consulta:**

```sql id="n3k8wr"
SELECT 
    'CLIENTE' AS origen,
    UPPER(contact_name) AS contacto,
    UPPER(company_name) AS organizacion,
    city,
    country
FROM customers

UNION ALL

SELECT 
    'PROVEEDOR' AS origen,
    UPPER(contact_name) AS contacto,
    UPPER(company_name) AS organizacion,
    city,
    country
FROM suppliers

UNION ALL

SELECT 
    'EMPLEADO' AS origen,
    UPPER(first_name || ' ' || last_name) AS contacto,
    'NORTHWIND TRADERS' AS organizacion,
    city,
    country
FROM employees

ORDER BY origen ASC, country ASC;
```

**Resultado:**

[Resultado pregunta 11](images/11.1.png)

**Comentario:** He utilizado `UNION ALL` para unir los contactos de clientes, proveedores y empleados manteniendo todas las filas, incluso si existen contactos con datos iguales. `UPPER()` convierte los nombres de contacto y organizaciones a mayúsculas. Para los empleados concateno el nombre y los apellidos con `||` y utilizo el literal `'NORTHWIND TRADERS'` como organización. Finalmente, ordeno por origen y país.

## Pregunta 12 — Mercados con desequilibrio

**Enunciado:** Compras y Ventas mantienen una discusión recurrente: ¿en qué países vendemos sin tener proveedor local, y en cuáles coincidimos? Resuelve las dos preguntas en dos consultas independientes: **a)** países donde hay clientes pero **ningún** proveedor. **b)** países donde hay **a la vez** clientes y proveedores. Ordena ambos resultados alfabéticamente.

**Consulta:**

**a) Países donde hay clientes pero ningún proveedor:**

```sql
SELECT country AS pais
FROM customers
EXCEPT
SELECT country AS pais
FROM suppliers
ORDER BY pais ASC;
```

**b) Países donde hay clientes y proveedores:**

```sql
SELECT country AS pais
FROM customers
INTERSECT
SELECT country AS pais
FROM suppliers
ORDER BY pais ASC;
```

**Resultado:**

[Resultado pregunta 12 — Apartado a](images/12.1.png)

[Resultado pregunta 12 — Apartado b](images/12.2.png)

**Comentario:** En el apartado **a)** he utilizado `EXCEPT` para obtener los países que aparecen en `customers` pero no en `suppliers`. En el apartado **b)** utilizo `INTERSECT` para obtener únicamente los países que aparecen en ambas tablas. Los operadores de conjunto eliminan automáticamente los valores duplicados y `ORDER BY` permite mostrar los países en orden alfabético.

## Pregunta 13 — Clientes que nunca han comprado pescado

**Enunciado:** El responsable de la categoría Seafood quiere una lista de cuentas sobre las que hacer campaña de captación. Localiza los clientes que **nunca** han incluido un producto de la categoría `'Seafood'` en ninguno de sus pedidos. Muestra el nombre del cliente, su país y el número total de pedidos que sí ha realizado, de mayor a menor.

**Consulta:**

```sql id="q2m7vx"
SELECT 
    c.company_name AS cliente,
    c.country AS pais,
    COUNT(o.order_id) AS pedidos_realizados
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o2
    JOIN order_details od USING (order_id)
    JOIN products p USING (product_id)
    JOIN categories cat USING (category_id)
    WHERE o2.customer_id = c.customer_id
      AND cat.category_name = 'Seafood'
)
GROUP BY c.customer_id, c.company_name, c.country
ORDER BY pedidos_realizados DESC;
```

**Resultado:**

[Resultado pregunta 13](images/13.1.png)

**Comentario:** He utilizado `NOT EXISTS` con una subconsulta correlacionada para excluir a los clientes que hayan comprado algún producto de la categoría `'Seafood'`. La subconsulta relaciona `orders`, `order_details`, `products` y `categories` para comprobar la categoría de los productos. Fuera de la subconsulta, utilizo `LEFT JOIN` para contar todos los pedidos realizados por cada cliente, incluidos aquellos que nunca han comprado. Finalmente, ordeno por el número de pedidos de mayor a menor.

## Pregunta 14 — Productos por encima de la media

**Enunciado:** El comité de precios quiere identificar el segmento premium del catálogo. Muestra los productos activos cuyo precio unitario supere el precio medio de **todo** el catálogo. Incluye en cada fila el precio del producto, el precio medio general y la diferencia entre ambos, todo redondeado a dos decimales. Ordena por diferencia descendente.

**Consulta:**

```sql id="v6p3ks"
SELECT 
    product_name AS producto,
    ROUND(unit_price::numeric, 2) AS precio,
    ROUND((SELECT AVG(unit_price) FROM products)::numeric, 2) AS precio_medio_catalogo,
    ROUND((unit_price - (SELECT AVG(unit_price) FROM products))::numeric, 2) AS diferencia
FROM products
WHERE discontinued = 0
  AND unit_price > (SELECT AVG(unit_price) FROM products)
ORDER BY diferencia DESC;
```

**Resultado:**

[Resultado pregunta 14](images/14.1.png)

**Comentario:** He utilizado una subconsulta escalar con `AVG(unit_price)` para calcular el precio medio de todo el catálogo. Esta subconsulta se utiliza tanto en el `WHERE` para seleccionar los productos que están por encima de la media como en el `SELECT` para mostrar el precio medio y calcular la diferencia. `ROUND()` permite redondear los valores a dos decimales y `ORDER BY` ordena los productos por la diferencia de mayor a menor.

## Pregunta 15 — Ticket medio por cliente

**Enunciado:** Dirección comercial quiere segmentar la cartera por valor medio de pedido, no por volumen total. Calcula, para cada cliente que haya comprado alguna vez, el número de pedidos, el importe total acumulado y el importe medio por pedido. Muestra los 15 clientes con mayor ticket medio.

**Consulta:**

```sql id="q8m4tz"
SELECT 
    c.company_name AS cliente,
    c.country AS pais,
    COUNT(pedidos.order_id) AS num_pedidos,
    ROUND(SUM(pedidos.importe_pedido)::numeric, 2) AS importe_total,
    ROUND(AVG(pedidos.importe_pedido)::numeric, 2) AS ticket_medio
FROM customers c
JOIN (
    SELECT 
        o.customer_id,
        o.order_id,
        SUM(od.unit_price * od.quantity * (1 - od.discount)) AS importe_pedido
    FROM orders o
    JOIN order_details od USING (order_id)
    GROUP BY o.customer_id, o.order_id
) pedidos ON c.customer_id = pedidos.customer_id
GROUP BY c.customer_id, c.company_name, c.country
ORDER BY ticket_medio DESC
LIMIT 15;
```

**Resultado:**

[Resultado pregunta 15](images/15.1.png)

**Comentario:** He utilizado una subconsulta en `FROM` para calcular primero el importe total de cada pedido sumando sus líneas. Después, en la consulta principal, agrupo esos pedidos por cliente para obtener el número de pedidos, el importe total acumulado y el ticket medio. De esta forma, el promedio se calcula sobre los importes de los pedidos y no directamente sobre las líneas. Finalmente, `LIMIT 15` muestra los 15 clientes con mayor ticket medio.

## Pregunta 16 — El producto más caro de cada categoría

**Enunciado:** Para cada categoría, muestra el producto con el precio unitario más alto. Incluye el nombre de la categoría, el nombre del producto, su precio y el precio medio de su categoría. Resuélvelo mediante una subconsulta correlacionada.

**Consulta:**

```sql
SELECT 
    c.category_name AS categoria,
    p.product_name AS producto,
    ROUND(p.unit_price::numeric, 2) AS precio,
    ROUND((
        SELECT AVG(p2.unit_price) 
        FROM products p2 
        WHERE p2.category_id = p.category_id
    )::numeric, 2) AS precio_medio_categoria
FROM 
    products p
INNER JOIN 
    categories c USING (category_id)
WHERE 
    p.unit_price = (
        SELECT MAX(p3.unit_price) 
        FROM products p3 
        WHERE p3.category_id = p.category_id
    )
ORDER BY 
    c.category_name ASC;
```

**Resultado:**

[Resultado pregunta 16](images/16.1.png)

**Comentario:** La consulta utiliza un `INNER JOIN` para relacionar productos con sus categorías. La subconsulta correlacionada del `WHERE` obtiene el precio máximo de la categoría del producto actual, por lo que solo se muestran los productos cuyo precio coincide con ese máximo. La segunda subconsulta correlacionada, en el `SELECT`, calcula el precio medio de la categoría correspondiente. Si hubiera varios productos con el mismo precio máximo dentro de una categoría, se mostrarían todos.

## Pregunta 17 — Segmentación ABC de la cartera de clientes

**Enunciado:** Clasifica a los clientes en cuatro segmentos según su facturación, utilizando cuartiles. Devuelve por segmento el número de clientes, la facturación total y el porcentaje sobre la facturación total de la compañía.

**Consulta:**

```sql
WITH facturacion_clientes AS (
    SELECT 
        c.customer_id,
        c.company_name AS cliente,
        COALESCE(SUM(od.unit_price * od.quantity * (1 - od.discount)), 0) AS facturacion_total
    FROM customers c
    LEFT JOIN orders o USING (customer_id)
    LEFT JOIN order_details od USING (order_id)
    GROUP BY c.customer_id, c.company_name
),
cuartiles AS (
    SELECT 
        cliente,
        facturacion_total,
        NTILE(4) OVER (ORDER BY facturacion_total DESC) AS cuartil
    FROM facturacion_clientes
),
segmentacion AS (
    SELECT 
        cliente,
        facturacion_total,
        CASE cuartil
            WHEN 1 THEN 'A - Estratégico'
            WHEN 2 THEN 'B - Consolidado'
            WHEN 3 THEN 'C - Ocasional'
            ELSE 'D - Marginal'
        END AS segmento
    FROM cuartiles
)
SELECT 
    segmento,
    COUNT(cliente) AS num_clientes,
    ROUND(SUM(facturacion_total)::numeric, 2) AS facturacion_segmento,
    ROUND(
        (SUM(facturacion_total) * 100.0 / 
        (SELECT SUM(facturacion_total) FROM segmentacion))::numeric, 
        2
    ) AS porcentaje_sobre_total
FROM segmentacion
GROUP BY segmento
ORDER BY segmento ASC;
```

**Resultado:**

[Resultado pregunta 17](images/17.1.png)

**Comentario:** La primera CTE calcula la facturación total de cada cliente. La segunda utiliza `NTILE(4)` para dividir los clientes en cuatro cuartiles ordenados de mayor a menor facturación. La tercera asigna cada cuartil a un segmento mediante `CASE`. Finalmente, se agrupan los clientes por segmento y se calcula la facturación total y el porcentaje que representa cada segmento sobre el total de la compañía.

## Pregunta 18 — Los tres productos más vendidos de cada categoría

**Enunciado:** Para cada categoría, obtén los tres productos con mayor facturación. Muestra la categoría, la posición dentro de la categoría, el nombre del producto, las unidades vendidas, la facturación y la posición global del producto en el conjunto de la compañía.

**Consulta:**

```sql
WITH metricas_productos AS (
    SELECT 
        c.category_name AS categoria,
        p.product_name AS producto,
        SUM(od.quantity) AS unidades,
        SUM(od.unit_price * od.quantity * (1 - od.discount)) AS facturacion,
        RANK() OVER (
            PARTITION BY c.category_name 
            ORDER BY SUM(od.unit_price * od.quantity * (1 - od.discount)) DESC
        ) AS posicion_en_categoria,
        RANK() OVER (
            ORDER BY SUM(od.unit_price * od.quantity * (1 - od.discount)) DESC
        ) AS posicion_global
    FROM categories c
    JOIN products p USING (category_id)
    JOIN order_details od USING (product_id)
    GROUP BY c.category_name, p.product_name
)
SELECT 
    categoria,
    posicion_en_categoria,
    producto,
    unidades,
    ROUND(facturacion::numeric, 2) AS facturacion,
    posicion_global
FROM metricas_productos
WHERE posicion_en_categoria <= 3
ORDER BY categoria ASC, posicion_en_categoria ASC;
```

**Resultado:**

[Resultado pregunta 18](images/18.1.png)

**Comentario:** La CTE calcula las métricas de cada producto y utiliza `RANK()` para obtener su posición dentro de cada categoría mediante `PARTITION BY`, y también su posición global sin particionar. Después, la consulta externa filtra los productos cuya posición en su categoría es como máximo 3. Se utiliza `RANK()` en lugar de `ROW_NUMBER()` porque permite conservar los empates: si varios productos comparten una posición, pueden aparecer más de tres productos en una categoría.

## Pregunta 19 — Evolución mensual con acumulado y media móvil

**Enunciado:** Para cada mes de 1997, calcula la facturación del mes, el total acumulado desde enero, la media móvil de los tres últimos meses, la facturación del mes anterior y la variación porcentual respecto al mes anterior.

**Consulta:**

```sql id="k7x2qm"
WITH ventas_mensuales AS (
    SELECT 
        DATE_TRUNC('month', o.order_date)::date AS mes,
        SUM(od.unit_price * od.quantity * (1 - od.discount)) AS facturacion
    FROM orders o
    JOIN order_details od USING (order_id)
    WHERE EXTRACT(YEAR FROM o.order_date) = 1997
    GROUP BY DATE_TRUNC('month', o.order_date)
)
SELECT 
    mes,
    ROUND(facturacion::numeric, 2) AS facturacion,
    -- Total acumulado desde enero
    ROUND(
        SUM(facturacion) OVER (
            ORDER BY mes 
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        )::numeric, 
        2
    ) AS acumulado,
    -- Media móvil de los 3 últimos meses (mes actual + 2 anteriores)
    ROUND(
        AVG(facturacion) OVER (
            ORDER BY mes 
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        )::numeric, 
        2
    ) AS media_movil_3m,
    -- Facturación del mes anterior
    ROUND(
        LAG(facturacion, 1) OVER (ORDER BY mes)::numeric, 
        2
    ) AS mes_anterior,
    -- Variación porcentual respecto al mes anterior
    ROUND((
        (facturacion - LAG(facturacion, 1) OVER (ORDER BY mes)) * 100.0 
        / NULLIF(LAG(facturacion, 1) OVER (ORDER BY mes), 0)
    )::numeric, 2) AS variacion_pct
FROM ventas_mensuales
ORDER BY mes ASC;
```

**Resultado:**

[Resultado pregunta 19](images/19.1.png)

**Comentario:** La CTE `ventas_mensuales` agrupa la facturación por mes de 1997 utilizando `DATE_TRUNC()`. Después, `SUM() OVER` calcula el acumulado desde enero, mientras que `AVG() OVER` con `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` obtiene la media móvil de tres meses. `LAG()` permite recuperar la facturación del mes anterior y calcular la variación porcentual. En enero, `mes_anterior` y `variacion_pct` aparecen como `NULL` porque no existe un mes anterior en el período analizado.

## Pregunta 20 — Cuadro de mando anual por categoría

**Enunciado:** Construye un cuadro de mando por categoría con la facturación de 1996, 1997 y 1998 en columnas separadas, el total de los tres años, el peso de cada categoría sobre la facturación total y una tendencia entre 1997 y 1998. Añade una fila de totales generales.

**Consulta:**

```sql id="n5r8cw"
WITH base_ventas AS (
    SELECT 
        c.category_name AS categoria,
        EXTRACT(YEAR FROM o.order_date)::integer AS anio,
        od.unit_price * od.quantity * (1 - od.discount) AS importe
    FROM categories c
    JOIN products p USING (category_id)
    JOIN order_details od USING (product_id)
    JOIN orders o USING (order_id)
),
resumen_anual AS (
    SELECT 
        categoria,
        SUM(importe) FILTER (WHERE anio = 1996) AS f_1996,
        SUM(importe) FILTER (WHERE anio = 1997) AS f_1997,
        SUM(importe) FILTER (WHERE anio = 1998) AS f_1998,
        SUM(importe) AS total
    FROM base_ventas
    GROUP BY categoria
)
SELECT 
    COALESCE(categoria, 'TOTAL GENERAL') AS categoria,
    ROUND(COALESCE(SUM(f_1996), 0)::numeric, 2) AS f_1996,
    ROUND(COALESCE(SUM(f_1997), 0)::numeric, 2) AS f_1997,
    ROUND(COALESCE(SUM(f_1998), 0)::numeric, 2) AS f_1998,
    ROUND(COALESCE(SUM(total), 0)::numeric, 2) AS total,
    ROUND(
        (SUM(total) * 100.0 / SUM(SUM(total)) OVER ())::numeric, 
        2
    ) AS peso_pct,
    /*
       NOTA DE CONTROL DE GESTIÓN:
       La tendencia entre 1997 y 1998 no es estrictamente comparable de forma directa,
       ya que 1996 solo contiene datos desde julio y 1998 abarca únicamente hasta mayo.
       Para una lectura de negocio real, se requeriría normalizar los datos por meses
       operativos completos o comparar periodos homogéneos (YoY ponderado).
    */
    CASE 
        WHEN categoria IS NULL THEN '-'
        WHEN SUM(f_1998) > SUM(f_1997) THEN 'CRECE'
        WHEN SUM(f_1998) < SUM(f_1997) THEN 'DECRECE'
        ELSE 'ESTABLE'
    END AS tendencia
FROM resumen_anual
GROUP BY ROLLUP (categoria)
ORDER BY (categoria IS NULL) ASC, total DESC;
```

**Resultado:**

[Resultado pregunta 20](images/20.1.png)

**Comentario:** La primera CTE obtiene la facturación de cada línea de pedido junto con su año y categoría. La segunda realiza el pivotado manual mediante `FILTER`, convirtiendo los años 1996, 1997 y 1998 en columnas. `ROLLUP` añade la fila de `TOTAL GENERAL`, mientras que `COALESCE()` sustituye los valores nulos. La función de ventana `SUM(SUM(total)) OVER ()` permite calcular el peso porcentual de cada categoría sobre el total. La columna `tendencia` compara 1997 con 1998, aunque esta comparación debe interpretarse con cautela porque 1998 solo contiene datos hasta mayo y, por tanto, no representa un año completo.




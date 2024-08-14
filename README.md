# PRUEBA TECNICA SOPORTE

Este documento describe el código utilizado para realizar la prueba tecnica para el cargo de soporte tecnico , en donde se pidio gestionar y actualizar las tablas de pedidos en una base de datos PostgreSQL.

# CODIGO UTILIZADO

```sql
--Elimina Triggers Existentes
DROP TRIGGER IF EXISTS update_inf ON cpedidos;
DROP TRIGGER IF EXISTS update_allprice ON dpedidos;

--Crea o reemplaza la funcion update_info
CREATE OR REPLACE FUNCTION update_info()
RETURNS TRIGGER AS
$$
BEGIN
    NEW.cpe_subtotal = (SELECT SUM(dpedidos.dpe_total) 
                        FROM dpedidos 
                        WHERE dpedidos.cpe_id = NEW.cpe_id);
    NEW.cpe_impuesto = (SELECT (NEW.cpe_subtotal * impuesto.imp_porcentaje::int / 100) 
                        FROM impuesto 
                        WHERE impuesto.imp_id = NEW.imp_id);
    NEW.cpe_total = NEW.cpe_subtotal + NEW.cpe_impuesto;

    RETURN NEW;
END;
$$
LANGUAGE plpgsql;

--Crea el trigger
CREATE TRIGGER update_inf
    BEFORE UPDATE ON cpedidos
    FOR EACH ROW
    EXECUTE FUNCTION update_info();

--Crea o reemplaza la funcion update_price
CREATE OR REPLACE FUNCTION update_price()
    RETURNS TRIGGER AS
    $$
    BEGIN
        NEW.dpe_total = NEW.dpe_precio * NEW.dpe_cantidad;
        RETURN NEW;
    END;
    $$
    LANGUAGE plpgsql;

--Crea el triger
CREATE TRIGGER update_allprice
    BEFORE UPDATE ON dpedidos
    FOR EACH ROW
    EXECUTE FUNCTION update_price();

--Actualizar totales de la cabecera de pedidos
 UPDATE cpedidos
    SET cpe_subtotal = (SELECT SUM(dpedidos.dpe_total) 
                        FROM dpedidos 
                        WHERE dpedidos.cpe_id = cpedidos.cpe_id),
        cpe_impuesto = (SELECT ((cpe_subtotal * impuesto.imp_porcentaje::int)/100) 
                        FROM impuesto 
                        WHERE impuesto.imp_id = cpedidos.imp_id),
        cpe_total= cpe_subtotal + cpedidos.cpe_impuesto;

--Actualizar el impuesto a 15% de los pedidos del mes Julio y agosto
UPDATE cpedidos
    SET imp_id = 1
    WHERE cpe_fecha BETWEEN '2024-07-01' AND '2024-08-31';

--Reporte de pedidos de clientes
SELECT cliente.cli_nombre AS cliente,
       EXTRACT(MONTH FROM cpedidos.cpe_fecha) AS mes,
       cpedidos.cpe_total AS total_impuesto
FROM cpedidos
JOIN cliente ON cpedidos.cli_id = cliente.cli_id
ORDER BY mes, total_impuesto DESC;

--Reporte con clientes que mas compraron en los ultimos 4 meses
SELECT cliente.cli_nombre AS cliente,
        EXTRACT(MONTH FROM cpedidos.cpe_fecha) AS mes,
        SUM(cpedidos.cpe_total) AS total_compras
FROM cpedidos
JOIN cliente ON cpedidos.cli_id = cliente.cli_id
WHERE cpedidos.cpe_fecha between '2024-05-01' and '2024-08-31'
GROUP BY cliente.cli_nombre, cpedidos.cpe_fecha
ORDER BY total_compras DESC
LIMIT 3;

--Actualizar la cantidad de unidades a 2 en todos los productos
UPDATE dpedidos
SET dpe_cantidad = 2
WHERE cpe_id IN (28, 29, 30);

UPDATE cpedidos
SET cpe_subtotal = (SELECT SUM(dpedidos.dpe_total)
                    FROM dpedidos
                    WHERE dpedidos.cpe_id = cpedidos.cpe_id),
    cpe_impuesto = (SELECT ((cpe_subtotal * impuesto.imp_porcentaje::int)/100)
                    FROM impuesto
                    WHERE impuesto.imp_id = cpedidos.imp_id),
    cpe_total= cpe_subtotal + cpedidos.cpe_impuesto;

-- Consultar todos los registros en dpedidos y cpedidos
SELECT * FROM dpedidos;
SELECT * FROM cpedidos;

```

# DESCRIPCION DEL CODIGO 

## 1. Eliminar Triggers Existentes

- Antes de crear nuevos triggers, es importante eliminar los que ya existen para evitar conflictos.

    ```sql
    DROP TRIGGER IF EXISTS update_inf ON cpedidos;
    DROP TRIGGER IF EXISTS update_allprice ON dpedidos;
    ```
## 2. Función para Actualizar `cpedidos`

- La función `update_info`  se encarga de recalcular los campos `cpe_subtotal`,`cpe_impuesto` , y `cpe_total` en la tabla `cpedidos` utilizando los datos de la tabla `dpedidos`y la tabla `impuesto`.


    ```sql
    CREATE OR REPLACE FUNCTION update_info()
    RETURNS TRIGGER AS
    $$
    BEGIN
        NEW.cpe_subtotal = (SELECT SUM(dpedidos.dpe_total) 
                            FROM dpedidos 
                            WHERE dpedidos.cpe_id = NEW.cpe_id);
        NEW.cpe_impuesto = (SELECT (NEW.cpe_subtotal * impuesto.imp_porcentaje::int / 100) 
                            FROM impuesto 
                            WHERE impuesto.imp_id = NEW.imp_id);
        NEW.cpe_total = NEW.cpe_subtotal + NEW.cpe_impuesto;

        RETURN NEW;
    END;
    $$
    LANGUAGE plpgsql;
    ```

-  El trigger `update_inf` se ejecuta antes de cada actualización en la tabla `cpedidos`, para garantizar que los valores de `cpe_subtotal`, `cpe_impuesto`, y `cpe_total` se recalculen automáticamente

    ````sql
    CREATE TRIGGER update_inf
    BEFORE UPDATE ON cpedidos
    FOR EACH ROW
    EXECUTE FUNCTION update_info();
    ````

## 3. Función para Actualizar el Precio en `dpedidos`

- La función `update_price` se encarga de recalcular el campo `dpe_total` en la tabla dpedidos basándose en el precio y la cantidad de cada producto.

    ```sql
    CREATE OR REPLACE FUNCTION update_price()
    RETURNS TRIGGER AS
    $$
    BEGIN
        NEW.dpe_total = NEW.dpe_precio * NEW.dpe_cantidad;
        RETURN NEW;
    END;
    $$
    LANGUAGE plpgsql;

    ```
- El trigger `update_allprice` se ejecuta antes de cada actualización en la tabla dpedidos para recalcular automáticamente el total de cada pedido.

    ```sql
    CREATE TRIGGER update_allprice
    BEFORE UPDATE ON dpedidos
    FOR EACH ROW
    EXECUTE FUNCTION update_price();
    ```


## 4. Actualizar totales de la cabecera de pedidos: `cpe_subtotal` `cpe_impuesto` `cpe_total`

- Esta consulta actualiza los campos `cpe_subtotal`, `cpe_impuesto`, y `cpe_total` en la tabla cpedidos utilizando los datos más recientes de las tablas dpedidos e impuesto.

    ```sql
    UPDATE cpedidos
    SET cpe_subtotal = (SELECT SUM(dpedidos.dpe_total) 
                        FROM dpedidos 
                        WHERE dpedidos.cpe_id = cpedidos.cpe_id),
        cpe_impuesto = (SELECT ((cpe_subtotal * impuesto.imp_porcentaje::int)/100) 
                        FROM impuesto 
                        WHERE impuesto.imp_id = cpedidos.imp_id),
        cpe_total= cpe_subtotal + cpedidos.cpe_impuesto;
    ```


## 5.Actualizar el impuesto a 15% de los pedidos del mes Julio y agosto

- Se actualiza el impuesto al 15% para los pedidos realizados entre julio y agosto de 2024. Posterior a esta consulta se ejectua el tigger `update_inf` para actualizar los campos en `cpedidos`

    ```sql
    UPDATE cpedidos
    SET imp_id = 1
    WHERE cpe_fecha BETWEEN '2024-07-01' AND '2024-08-31';
    ```

## 6.Generar Reportes de Pedidos de Clientes

- Genera  un reporte tabular de pedidos de clientes, muestra el cliente, mes, y valor total con impuesto, ordenado por mes y valor descendente.


    ```sql
    SELECT cliente.cli_nombre AS cliente,
       EXTRACT(MONTH FROM cpedidos.cpe_fecha) AS mes,
       cpedidos.cpe_total AS total_impuesto
    FROM cpedidos
    JOIN cliente ON cpedidos.cli_id = cliente.cli_id
    ORDER BY mes, total_impuesto DESC;
    ```
- Genera un reporte tabular del top 3 clientes que mas compras realizaron en los ultimos 4 meses .


    ```sql
    SELECT cliente.cli_nombre AS cliente,
        EXTRACT(MONTH FROM cpedidos.cpe_fecha) AS mes,
        SUM(cpedidos.cpe_total) AS total_compras
    FROM cpedidos
    JOIN cliente ON cpedidos.cli_id = cliente.cli_id
    WHERE cpedidos.cpe_fecha between '2024-05-01' and '2024-08-31'
    GROUP BY cliente.cli_nombre, cpedidos.cpe_fecha
    ORDER BY total_compras DESC
    LIMIT 3;
    ```

## 7.Actualizar la Cantidad de Unidades en dpedidos
- Se actualiza la cantidad de unidades a 2 al valor actual en dpedidos para los pedidos con `cpe_id` `28`, `29` y `30`


    ```sql
        UPDATE dpedidos
        SET dpe_cantidad = 2
        WHERE cpe_id IN (28, 29, 30);
    ```
- Considerado el trigger inicial  de `update_allprice` , se acciona actualizando automatica los valores de `dpe_total` en los `cpe_id` requeridos.

- Se realiza nuevamente la actualizacion de  cpedidos considerando el nuevo valor de los subtotales:
    ```sql
    UPDATE cpedidos
    SET cpe_subtotal = (SELECT SUM(dpedidos.dpe_total) 
                        FROM dpedidos 
                        WHERE dpedidos.cpe_id = cpedidos.cpe_id),
        cpe_impuesto = (SELECT ((cpe_subtotal * impuesto.imp_porcentaje::int)/100) 
                        FROM impuesto 
                        WHERE impuesto.imp_id = cpedidos.imp_id),
        cpe_total= cpe_subtotal + cpedidos.cpe_impuesto;
        
    ```
## 8. Herramientas ultilizadas
- pgAdmin
- PostgreSQL
- DataGrip

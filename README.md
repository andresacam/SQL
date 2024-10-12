# SQL
Codigo de SQL
```sql
--ESTE CÓDIGO SE DESARROLLO ESPECIFICAMENTE PARA UN PROYECTO DE ANÁLISIS DE DATOS,
PREVIO A REALIZAR EL MODELO DE LA BASE DE DATOS HICE UN ANÁLISIS EXPLORATORIO PARA
IDENTIFICAR ALGUNOS PROBLEMAS. POSTERIORMENTE, SE ESTABLECIERON LAS RELACIONES ENTRE
LAS TABLAS Y SE INTEGRÓ TODA LA INFORMACIÓN CON POWER BI.

-- Modificamos los nombres de las columnas para tenerlos en un solo formato
-- Eliminamos la columna ok comercial por ser irrelevante

ALTER TABLE Ventas1 RENAME COLUMN Suc TO Sucursal;

ALTER TABLE Ventas1 DROP COLUMN OK_Comercial;

--Ahora unimos todas las tablas en una final llamada ventasf

CREATE TABLE ventasf (
    ID_Cliente INT,
    ID_Producto INT,
    Fecha DATE,
    Sucursal VARCHAR(50),
    Monto_Venta DECIMAL,
    Vendedor INT
);

INSERT INTO ventasf (ID_Cliente, ID_Producto, Fecha, Sucursal, Monto_Venta, Vendedor)
SELECT ID_Cliente, ID_Producto, Fecha, Sucursal, Monto_Venta, Vendedor FROM ventas
UNION
SELECT ID_Cliente, ID_Producto, Fecha, Sucursal, Monto_Venta, Vendedor FROM ventas1
UNION
SELECT ID_Cliente, ID_Producto, Fecha, Sucursal, Monto_Venta, Vendedor FROM ventas2;

--Es importante aclarar que la funcion UNION elimina las filas duplicadas en caso de que existan
--si no queremos esto tenemos que usar UNION ALL

-- Verificamos el paso anterior--

SELECT * FROM ventasf LIMIT 10;

--Exportamos como CSV la tabla combinada 


--ANALISIS DE LA TABLA PRODUCTOS

--Forma de contar la cantidad de valores nulos por calumna
SELECT 
    COUNT(ID_PRODUCTO) as total_id_producto,
    COUNT(Concepto) as total_concepto,
    COUNT(Tipo) as total_tipo,
    COUNT(Precio) as total_precio
FROM productos
WHERE 
    ID_PRODUCTO IS NULL OR
    Concepto IS NULL OR
    Tipo IS NULL OR
    Precio IS NULL;
    
--Verificamos si existen productos con conceptos duplicados--
SELECT Concepto, COUNT(Concepto) FROM productos  GROUP BY Concepto HAVING COUNT(Concepto) > 1;

--Verificamos si existen productos con IDPRODUCTO duplicados--
SELECT ID_PRODUCTO, COUNT(ID_PRODUCTO) FROM productos  GROUP BY ID_PRODUCTO HAVING COUNT(ID_PRODUCTO) > 1;

--Ahora sabemos que solo se repiten los conceptos, entonces queremos eliminar los duplicados--

CREATE TABLE productos_sin_duplicados AS
SELECT DISTINCT Concepto, ID_PRODUCTO, Tipo, Precio FROM productos;

--Verificamos que la nueva tabla no tiene conceptos duplicados--
SELECT Concepto, COUNT(Concepto) FROM productos_sin_duplicados  GROUP BY Concepto HAVING COUNT(Concepto) > 1;
select * from productos_sin_duplicados where concepto = 'CD DVD+-RW 24X SAMSUNG SATA BLACK' or concepto = 'MEM USB 64GB KINGSTON DATA 3.1';


--Los datos siguien repitiendose, esto podria deberse a que tienen diferentes espacios entre palabras
--para verificarlo proponemos la siguiente consulta.
select ID_PRODUCTO, length(Concepto) from productos_sin_duplicados
where ID_PRODUCTO = '42786' or ID_PRODUCTO = '42790' or ID_PRODUCTO = '42813' or ID_PRODUCTO = '42814'; 

--Vemos que no es el caso, entonces procedemos a eliminarlos individualmente--
DELETE FROM productos_sin_duplicados
WHERE ID_PRODUCTO = '42786' or ID_PRODUCTO = '42813' ;

--Volvemos a verificar--
SELECT Concepto, COUNT(Concepto) FROM productos_sin_duplicados  GROUP BY Concepto HAVING COUNT(Concepto) > 1;

--Analizamos los precios de los productos
SELECT avg(Precio), min(Precio), max(Precio) from productos_sin_duplicados;

--Bucamos los productos con precio igual a cero
SELECT * FROM productos_sin_duplicados
WHERE Precio = '0,00';
--Encontramos 4 productos sin precio

--Ahora necesitamos saber si fueron vendidos
SELECT * FROM ventasf
WHERE 
    ID_PRODUCTO = '42802' OR 
    ID_PRODUCTO = '42806' OR
    ID_PRODUCTO = '42807' OR
    ID_PRODUCTO = '42809'
ORDER BY ID_PRODUCTO;
--Vemos que existen ventas de estos prouctos y que ademas tienen distintos precios

--Consultamos las ventas totales de estos productos
SELECT ID_Producto, sum(Monto_Venta) FROM ventasf
WHERE 
    ID_PRODUCTO = '42802' OR 
    ID_PRODUCTO = '42806' OR
    ID_PRODUCTO = '42807' OR
    ID_PRODUCTO = '42809'
GROUP BY ID_PRODUCTO
ORDER BY ID_PRODUCTO;
--Los montos son coinciderables


--PARA UN PROYECTO DE ANÁLISIS DE DATOS DE BIENES RAÍCES SE DESARROLLARON
CONSULTAS PARA COMPRENDER LA INFORMACIÓN DISPONIBLE.

select * from rtate limit 10;

--CONSULTE EL PRECIO PROMEDIO SEGUN EL BARRIO EN EL QUE SE ENCUENTRA LA PROPIEDAD

select suburb, round(avg(price)) from rtate 
group by suburb 
order by round(avg(price)) desc;

--CONSULTE LA DISTANCIA PROMEDIO DE LAS PROPIEDADES DE CADA BARRIO RESPECTO DEL CENTRO

select suburb, round(avg(distance),2) as distance from rtate 
group by suburb;

--CUENTE LA CANTIDAD DE PROPIEDADES POR CADA REGION

select Regionname, count(address) from rtate 
group by Regionname;

--CUAL ES LA SUPERFICIE PROMEDIO DE CADA BARRIO

select suburb, round(avg(landsize)) as superficie from rtate 
group by suburb;

-- CONSULTE LAS CARACTERISTICAS PRINCIPALES DE LOS 10 BARRIOS MAS BARATOS DE MELBOURNE

select suburb, 
       round(avg(price)) as precio_promedio, 
       round(avg(landsize),2) as superficie, 
       round(avg(distance),2) as d_centro, 
       round(avg(car)) as garage, 
       round(avg(bathroom)) as banios 
       from rtate
group by suburb 
order by round(avg(price)) asc
limit 10;

--CONSULTE LAS PROPIEDADES CON EL PRECIO MAS BAJO QUE CUMPLAN LOS REQUERIMIENTOS DEL CLIENTE. ENTRE USD 300000 Y USD 450000 CON MAS DE UN BANIO Y A MENOS DE 15 KM DEL CENTRO

select address, price, distance, bathroom from rtate 
where price > 300000 and price < 450000 and bathroom > 1 and distance < 15
order by price asc
limit 5;


--CANTIDAD DE PROPIEDADES CON UN PRECIO MAYOR AL PROMEDIO Y UNA SUPERFICIE INFERIOR AL PROMEDIO DE CADA ZONA

select suburb, count(address) as q_particular from rtate 
where price > (select avg(price) from rtate group by suburb) and landsize < (select avg(landsize) from rtate group by suburb)
group by suburb
order by q_particular desc;


--CLASIFICACION DE PROPIEDADES SEGUN PRECIO E INFORMACION DISPONIBLE

select address, price, landsize, distance, case when price < (select avg(price) from rtate ) and landsize != 0 then 'oportunidad'
                                                when price < (select avg(price) from rtate) and landsize = 0 then 'reelevar propiedad'
                                                when price > (select avg(price) from rtate) then 'no conveniente'
                                                end as estado
from rtate
where suburb = 'Richmond' and method = 'S'
order by address desc;

--CONTEO DE CEROS PARA LANDSIZE

select suburb, 
       count(landsize) as ceros, 
       min(price) as minp,
       round(avg(price),2) as prom,
       max(price) as maxp 
from rtate 
where landsize = 0  and method = 'S'
group by suburb
order by ceros desc
;

--VALOR TOTAL DE CADA REGION

select count(address) as cantidad, 
       regionname, 
       sum(price) as valor_total, 
       round(avg(price),2) as prom 
from rtate group by regionname
order by cantidad desc;


select suburb, Propertycount, councilarea from rtate group by suburb order by councilarea ;

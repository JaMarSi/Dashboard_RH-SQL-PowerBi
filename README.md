# Dashboard RH
### SQL + POWER BI

Informe de dos páginas para el departamento de Recursos Humanos sobre los empleados de la empresa.

Limpieza y extracción de datos con MySQL, visualización con Power Bi.

Código de SQL:

-- CREAR LA BASE DE DATOS

CREATE DATABASE Proyecto_RH;

-- SELECCIONAR LA BASE DE DATOS

USE proyecto_rh;

-- IMPORTAR LA TABLA DE DATOS DEL ARCHIVO .CSV

-- Visualizacion de los datos importados

SELECT * FROM hr;

-- CREAR COPIA DE LA TABLA ORIGINAL:

CREATE TABLE hr_copy SELECT * FROM hr;

-- LIMPIEZA DE DATOS:

-- MODIFICAR NOMBRE DE LA COLUMNA ID

ALTER TABLE hr
CHANGE COLUMN ï»¿id emp_id VARCHAR(20) NOT NULL;

-- VER TIPO DE VARIABLES

DESC hr;

-- CAMBIAR EL FORMATO DE LA FECHA DE birthdate (NO LO RECONOCE)

UPDATE hr
SET birthdate = CASE
	WHEN birthdate LIKE '%/%' THEN DATE_FORMAT(str_to_date(birthdate, '%m/%d/%Y'), '%Y-%m-%d')
    WHEN birthdate LIKE '%-%' THEN DATE_FORMAT(str_to_date(birthdate, '%m-%d-%Y'), '%Y-%m-%d')
    ELSE null
END;

-- SI LA CONSULTA ANTERIOR NO FUNCIONA, DESACTIVAR LAS FUNCIONES DE SEGURIDAD DE SQL:

SET sql_safe_updates = 0;

-- VER LOS CAMBIOS EN LAS FECHAS REALIZADOS CORRECTAMENTE

select * from hr;

-- CAMBIAMOS EL TIPO DE LA COMUNMA birthdate PARA RECONOCERLO COMO FECHA

ALTER TABLE hr MODIFY COLUMN birthdate DATE;

-- CAMBIAR EL FORMATO DE hire_date (ES DE TIPO TEXTO)

UPDATE hr
SET hire_date = CASE
	WHEN hire_date LIKE '%/%' THEN DATE_FORMAT(str_to_date(hire_date, '%m/%d/%Y'), '%Y-%m-%d')
    WHEN hire_date LIKE '%-%' THEN DATE_FORMAT(str_to_date(hire_date, '%m-%d-%Y'), '%Y-%m-%d')
    ELSE null
END;
SELECT hire_date FROM hr;

-- CAMBIAR EL FORMATO DE termdate (INCLUYE FECHA Y HORA)

UPDATE hr
SET termdate = date(str_to_date(termdate,'%Y-%m-%d %H:%i:%s UTC'))
WHERE termdate IS NOT NULL AND termdate !=' ';

-- SI NO FUNCIONA str_to_date:

SET @@SESSION.sql_mode='NO_ZERO_DATE,NO_ZERO_IN_DATE';

-- cambiar valores NULL por 0s

UPDATE hr
SET termdate='0000-00-00'
WHERE termdate IS NULL;

-- MODIFICAR TIPO DE VARIABLE A DATE

ALTER TABLE hr
MODIFY COLUMN termdate DATE;
ALTER TABLE hr
MODIFY COLUMN hire_date DATE;

-- AÑADIR COLUMNA CON LA EDAD (PARA NO TENER QUE CALCULARLA EN CADA CONSULTA)

ALTER TABLE hr ADD COLUMN age INT;
UPDATE hr SET age = TIMESTAMPDIFF(YEAR, birthdate, CURDATE());

-- VER LOS RESULTADOS:

SELECT birthdate, age FROM hr;

-- VISUALIZAR VALORES ATIPICOS: aparecen edades en negativo

SELECT MIN(age) AS edad_min,
MAX(age) AS edad_max
FROM hr;

-- EDAD MENOR DE 18 AÑOS: no puede haber trabajadores menores de edad

SELECT COUNT(*) FROM hr
WHERE age<18;

-- PREGUNTAS PARA MOSTRAR EN EL INFORME

-- DIVISION POR GENEROS DE LOS EMPLEADOS (guardar la consulta en archivo .csv)

SELECT gender, COUNT(*) AS recuento
FROM hr 
WHERE age>=18 AND termdate = 0000-00-00
GROUP BY gender;

-- DIVISION POR RAZAS

SELECT race, COUNT(*) AS recuento
FROM hr
WHERE age>=18 AND termdate = 0000-00-00
GROUP BY race
ORDER BY COUNT(*) DESC;

-- GRUPOS DE EDADES: ver que grupos hacer y que rangos de edades de trabajadores tenemos

SELECT 
	MIN(age) AS edad_minima,
    MAX(age) AS edad_maxima
    FROM hr
    WHERE age>=18 AND termdate = 0000-00-00;

SELECT
	CASE
		WHEN age >=18 AND age <= 24 THEN '18-24'
        WHEN age >=25 AND age <= 34 THEN '25-34'
        WHEN age >=35 AND age <= 44 THEN '35-44'
        WHEN age >=45 AND age <= 54 THEN '44-54'
        WHEN age >=55 AND age <= 64 THEN '55-64'
        ELSE '+65'
    END AS grupo_edad,
    COUNT(*) AS recuento
FROM hr
WHERE age>=18 AND termdate = 0000-00-00
GROUP BY grupo_edad
ORDER BY grupo_edad;

-- GRUPO DE EDAD POR GENEROS:

SELECT
	CASE
		WHEN age >=18 AND age <= 24 THEN '18-24'
        WHEN age >=25 AND age <= 34 THEN '25-34'
        WHEN age >=35 AND age <= 44 THEN '35-44'
        WHEN age >=45 AND age <= 54 THEN '44-54'
        WHEN age >=55 AND age <= 64 THEN '55-64'
        ELSE '+65'
    END AS grupo_edad,
    gender,
    COUNT(*) AS recuento
FROM hr
WHERE age>=18 AND termdate = 0000-00-00
GROUP BY grupo_edad, gender
ORDER BY grupo_edad, gender;

-- NUMERO DE TRABAJADORES EN REMOTO Y PRESENCIAL

SELECT location,
	COUNT(*) AS recuento
FROM hr
WHERE age>=18 AND termdate = 0000-00-00
GROUP BY location;

-- TIEMPO DE DURACIÓN EN LA EMPRESA

SELECT
	ROUND(AVG(datediff(termdate, hire_date))/365,0) AS media_duracion
FROM hr
WHERE termdate <= curdate() AND termdate <> 0000-00-00 AND age>=18;

-- DISTRIBUCION POR GENERO EN LOS DEPARTAMENTOS Y PUESTOS DE TRABAJO

SELECT department,
	gender,
    COUNT(*) AS recuento
FROM hr
WHERE age>=18 AND termdate = 0000-00-00
GROUP BY department, gender
ORDER BY department;

-- DISTRIBUCION DE LOS PUESTOS DE TRABAJO EN LA EMPRESA

SELECT jobtitle,
	COUNT(*) AS recuento
FROM hr
WHERE age>=18 AND termdate = 0000-00-00
GROUP BY jobtitle
ORDER BY jobtitle DESC;

-- DEPARTAMENTOS CON LA TASA DE ROTACION MAS ALTA (subconsulta para calcular los valores de la tasa de rotacion)

SELECT department,
	recuento_total,
    recuento_despedidos,
    recuento_despedidos/recuento_total AS tasa_rotacion
FROM (
	SELECT department,
		COUNT(*) AS recuento_total,
        SUM(CASE WHEN termdate <> 0000-00-00 AND termdate<=curdate() THEN 1 ELSE 0 END) AS recuento_despedidos
        FROM hr
        WHERE age>= 18
        GROUP BY department
        ) AS subquery
	ORDER BY tasa_rotacion DESC;
    
    -- CANTIDAD DE EMPLEADOS POR ESTADO
    
    SELECT location_state,
		COUNT(*) AS recuento
	FROM hr
    WHERE age>=18 AND termdate = 0000-00-00
    GROUP BY location_state
    ORDER BY recuento DESC;
    
    -- CAMBIO DEL TOTAL DE EMPLEADOS CON LOS AÑOS
    
    SELECT
		year,
		contrataciones,
        despidos,
        contrataciones - despidos AS diferencia_plantilla,
        ROUND((contrataciones - despidos)/contrataciones *100,2) AS porcentaje_cambio
	FROM(
		SELECT YEAR(hire_date) AS year,
			COUNT(*) AS contrataciones,
			SUM(CASE WHEN termdate<>0000-00-00 AND termdate<= curdate() THEN 1 ELSE 0 END) AS despidos
        FROM hr
        WHERE age>=18
        GROUP BY YEAR(hire_date)
        ) AS subquery
	ORDER BY year ASC;
	
-- TENDENCIA DE ROTACION POR DEPARTAMENTO EN AÑOS

SELECT department,
	ROUND(AVG(datediff(termdate, hire_date)/365),0) AS promedio_rotacion
FROM hr
WHERE termdate<=curdate() AND termdate<> 0000-00-00 AND age >=18
GROUP BY department;

Cargamos todos los archivos .csv en Power Bi y representamos los datos extraidos:

![Dashboard Hoja1](https://user-images.githubusercontent.com/122362266/236689561-a95c81a5-5182-466c-b201-2f4f3c9b9d2f.PNG)

![Dashboard Hoja2](https://user-images.githubusercontent.com/122362266/236689567-6a1b750a-a099-4676-b7b4-93318be36994.PNG)

# proyecto_8_webscraping_SQL
Recopilación y almacenamiento de datos (SQL) | Zuber Taxis Chicago | Preferencias de pasajeros ante factores externos

# **ZUBER: ANÁLISIS DE VIAJES COMPARTIDOS EN CHICAGO**

## **RESUMEN DEL PROYECTO**

Este proyecto se enfoca en el análisis de datos de viajes compartidos en Chicago para la nueva empresa **Zuber**. 
El objetivo principal es descubrir:
- Patrones en la información disponible
- Entender las preferencias de los pasajeros
- Evaluar el impacto de factores externos (clima,frecuencia y duración de los viajes).

## **OBJETIVO**

El objetivo es proporcionar a Zuber información estratégica para su lanzamiento en Chicago. Esto se logrará mediante:

* Análisis de los viajes de los competidores para identificar las empresas líderes y los barrios más populares.  
* Exploración de la relación entre las condiciones climáticas y el tiempo de viaje.  
* Prueba de una hipótesis sobre el impacto del clima en la duración de los viajes.

## **METODOLOGÍA DE ANÁLISIS**

### **1\. DESCRIPCIÓN DE LOS DATOS**

Se analizaron datos de viajes en taxi en Chicago, provenientes de cuatro tablas en una base de datos:

* neighborhoods: Información de los barrios de la ciudad.  
* cabs: Datos sobre los taxis y sus empresas propietarias.  
* trips: Registros detallados de cada viaje, incluyendo duración, distancia y ubicación.  
* weather\_records: Datos meteorológicos por hora, como temperatura y una breve descripción de las condiciones.

### **2\. PREPROCESAMIENTO DE DATOS**

#### **Web Scraping de Datos Climáticos**

Se utilizó la biblioteca pandas para extraer los datos climáticos de un sitio web y guardarlos en un DataFrame para su posterior análisis.

import pandas as pd

\# URL del sitio web con los datos climáticos  
url \= 'https://practicum-content.s3.us-west-1.amazonaws.com/data-analyst-eng/moved\_chicago\_weather\_2017.html'

try:  
    \# Lee la tabla HTML y la carga en un DataFrame de pandas  
    weather\_records \= pd.read\_html(url, attrs={'id': 'weather\_records'})\[0\]

    \# Muestra el DataFrame cargado  
    print(weather\_records)  
except Exception as e:  
    print(f"Ocurrió un error al intentar leer la URL: {e}")

#### **Análisis Exploratorio de Datos (SQL)**

Se ejecutaron varias consultas SQL para el análisis inicial de los datos y la preparación de los archivos CSV:

* **Viajes por Compañía:** Se calculó el número de viajes para cada empresa de taxis en dos fechas específicas.  
  SELECT  
      c.company\_name,  
      COUNT(t.trip\_id) AS trips\_amount  
  FROM  
      trips AS t  
  JOIN  
      cabs AS c ON t.cab\_id \= c.cab\_id  
  WHERE  
      CAST(t.start\_ts AS DATE) BETWEEN '2017-11-15' AND '2017-11-16'  
  GROUP BY  
      c.company\_name  
  ORDER BY  
      trips\_amount DESC;

* **Viajes de "Yellow" o "Blue"**: Se filtraron los viajes por nombre de compañía para un rango de fechas.  
  SELECT  
      c.company\_name,  
      COUNT(t.trip\_id) AS trips\_amount  
  FROM  
      trips AS t  
  JOIN  
      cabs AS c ON t.cab\_id \= c.cab\_id  
  WHERE  
      CAST(t.start\_ts AS DATE) BETWEEN '2017-11-01' AND '2017-11-07'  
      AND (  
          c.company\_name ILIKE '%Yellow%' OR c.company\_name ILIKE '%Blue%')  
  GROUP BY  
      c.company\_name  
  ORDER BY  
      c.company\_name ASC;

* **Empresas Populares vs. Otras**: Se agruparon los viajes de las dos compañías más populares (Flash Cab y Taxi Affiliation Services) y se consolidaron las demás en un grupo Other.  
  SELECT  
      CASE  
          WHEN c.company\_name \= 'Flash Cab' THEN 'Flash Cab'  
          WHEN c.company\_name \= 'Taxi Affiliation Services' THEN 'Taxi Affiliation Services'  
          ELSE 'Other'  
      END AS company,  
      COUNT(t.trip\_id) AS trips\_amount  
  FROM  
      trips AS t  
  JOIN  
      cabs AS c ON t.cab\_id \= c.cab\_id  
  WHERE  
      CAST(t.start\_ts AS DATE) BETWEEN '2017-11-01' AND '2017-11-07'  
  GROUP BY  
      company  
  ORDER BY  
      trips\_amount DESC;

### **3\. PRUEBA DE HIPÓTESIS**

#### **Preparación de Datos (SQL)**

Se utilizaron consultas SQL para aislar los datos necesarios para la prueba de hipótesis:

* **Identificadores de Barrios**: Se obtuvieron los IDs de los barrios "Loop" y "O'Hare International Airport".  
  SELECT  
      neighborhood\_id,  
      name  
  FROM  
      neighborhoods  
  WHERE  
      name LIKE '%Hare' OR name LIKE 'Loop';

* **Clasificación de Condiciones Climáticas**: Se clasificaron las condiciones climáticas por hora como Good o Bad.  
  SELECT  
      ts,  
      CASE  
          WHEN description LIKE '%rain%' OR description LIKE '%storm%' THEN 'Bad'  
          ELSE 'Good'  
      END AS weather\_conditions  
  FROM  
      weather\_records  
  ORDER BY  
      ts ASC;

* **Recuperación de Viajes Específicos**: Se recuperaron todos los viajes desde el Loop hasta O'Hare en sábados, uniéndolos con los datos climáticos para obtener la duración y las condiciones meteorológicas.  
  SELECT  
      t.start\_ts,  
      CASE  
          WHEN wr.description LIKE '%rain%' OR wr.description LIKE '%storm%' THEN 'Bad'  
          ELSE 'Good'  
      END AS weather\_conditions,  
      t.duration\_seconds  
  FROM  
      trips AS t  
  INNER JOIN  
      weather\_records AS wr ON CAST(t.start\_ts AS DATE) \= CAST(wr.ts AS DATE) AND EXTRACT(HOUR FROM t.start\_ts) \= EXTRACT(HOUR FROM wr.ts)  
  WHERE  
      t.pickup\_location\_id \= 50  
      AND t.dropoff\_location\_id \= 63  
      AND EXTRACT(DOW FROM t.start\_ts) \= 6  
  ORDER BY  
      t.trip\_id;

#### **Prueba de Hipótesis (Python)**

Se utilizó el archivo CSV generado a partir de la última consulta para probar la hipótesis con Python y la biblioteca scipy.stats.

* **Hipótesis Nula (**H\_0**):** La duración promedio de los viajes desde el Loop hasta O'Hare no cambia los sábados lluviosos.  
* **Hipótesis Alternativa (**H\_1**):** La duración promedio de los viajes desde el Loop hasta O'Hare sí cambia los sábados lluviosos.

Se utilizó la prueba **t de Student para muestras independientes** con un nivel de significación (alpha) de 0.05. Previamente, se usó la **prueba de Levene** para determinar si las varianzas de las muestras eran iguales, lo que guió la configuración de la prueba t.

## **CONCLUSIONES PRINCIPALES**

**La duración promedio de los viajes en taxi desde el Loop hasta el Aeropuerto Internacional O'Hare varía considerablemente los sábados cuando las condiciones climáticas son 'Bad' (lluviosas/tormentosas) en comparación con los sábados de 'Good' (buen clima).**

Se rechazó la hipótesis nula con evidencia estadística sobre la incidencia de las condiciones climáticas adversas y su impacto en el tiempo de los viajes. Esto sugiere que elementos como el tráfico, la visibilidad reducida o las precauciones de seguridad durante el mal tiempo alargan el trayecto en promedio.

La diferencia, según la hipótesis planteada, no es algo al azar: los viajes en sábados lluviosos tienen una duración promedio de **2427.21 segundos** (aproximadamente 40.45 minutos), mientras que los viajes en sábados con buen clima tienen una duración promedio de **1999.68 segundos** (aproximadamente 33.33 minutos). Esta diferencia de más de 7 minutos en promedio sugiere que las condiciones meteorológicas adversas de los sábados alargan el tiempo de viaje entre el Loop y O'Hare.

**Implicaciones:** Los usuarios deben salir con más tiempo los sábados lluviosos si se dirigen al aeropuerto, y las empresas de taxis podrían ajustar sus estimaciones de tiempo en base a las previsiones meteorológicas.

**Limitaciones:** El estudio se basa en los datos proporcionados y no en todos los viajes posibles. Además, solo se consideraron dos categorías de clima.

## **TECNOLOGÍAS UTILIZADAS**

* **SQL:** Consultas de la base de datos para el análisis y la preparación de los datos.  
* **Python:** Para el web scraping, el análisis exploratorio, la visualización de datos y la prueba de hipótesis.  
* **Pandas:** Manipulación y análisis de datos en Python.  
* **Matplotlib y Seaborn:** Creación de visualizaciones para el análisis exploratorio.  
* **SciPy:** Implementación de la prueba de hipótesis (prueba t de Student y prueba de Levene).

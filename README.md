### Introducción del Proyecto

Zuber es una nueva empresa de viajes compartidos en Chicago, y el objetivo del análisis de datos es comprender los patrones de los pasajeros y el impacto de factores externos, como el clima, en la demanda de viajes. A través del análisis de información sobre competidores y condiciones meteorológicas, se evaluará cómo los días lluviosos afectan la duración y frecuencia de los viajes. Este proyecto tiene como fin proporcionar a Zuber insights clave para optimizar su estrategia, mejorando tanto la experiencia del usuario como la eficiencia operativa en un mercado altamente competitivo.

### Desarrollo del Proyecto
#### Paso 1
En el primer paso del proyecto, se solicitó el análisis de datos meteorológicos en Chicago durante noviembre de 2017, específicamente desde un archivo HTML que contiene información sobre las condiciones climáticas. Para completar esta tarea, se utilizó un enfoque de web scraping con las bibliotecas requests y BeautifulSoup en Python para extraer los datos de la página web. El proceso comienza con la descarga de la página utilizando la URL proporcionada, luego se empleó BeautifulSoup para parsear el contenido HTML y localizar la tabla con la información relevante (Dataframe weather_records).

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
URL='https://practicum-content.s3.us-west-1.amazonaws.com/data-analyst-eng/moved_chicago_weather_2017.html'
req = requests.get(URL)
soup = BeautifulSoup(req.text, 'lxml')
table = soup.find('table', attrs={"id": "weather_records"})
heading_table=[]
for row in table.find_all('th'):
    heading_table.append(row.text)   
content=[]
for row in table.find_all('tr'):
    if not row.find_all('th'):
        content.append([element.text for element in row.find_all('td')])
weather_records = pd.DataFrame(content, columns = heading_table)
print(weather_records)

```
Resultado: 

```python
Date and time Temperature       Description
0    2017-11-01 00:00:00     276.150     broken clouds
1    2017-11-01 01:00:00     275.700  scattered clouds
2    2017-11-01 02:00:00     275.610   overcast clouds
3    2017-11-01 03:00:00     275.350     broken clouds
4    2017-11-01 04:00:00     275.240     broken clouds
5    2017-11-01 05:00:00     275.050   overcast clouds
6    2017-11-01 06:00:00     275.140   overcast clouds
7    2017-11-01 07:00:00     275.230   overcast clouds
8    2017-11-01 08:00:00     275.230   overcast clouds
...
```

Este conjunto de datos se puede utilizar para correlacionar las condiciones climáticas con otros factores de interés, como la demanda de viajes de Zuber en Chicago, y es un paso crucial para probar la hipótesis sobre el impacto del clima en la duración y frecuencia de los viajes.

#### Paso 2: Análisis exploratorio de datos (Consultas SQL)
Se realizaron varios análisis clave para entender mejor la información sobre los viajes en taxi en Chicago durante noviembre de 2017. El primero análisis se centró en encontrar el número total de viajes realizados por cada empresa de taxis entre el 15 y 16 de noviembre de 2017.

```sql

SELECT 
    cabs.company_name AS company_name,
    COUNT(DISTINCT trips.trip_id) AS trip_amount
FROM 
    cabs
    INNER JOIN trips ON trips.cab_id = cabs.cab_id
    
WHERE 
CAST(trips.start_ts AS date) BETWEEN '2017-11-15' AND '2017-11-16'
    
GROUP BY 
    cabs.company_name
ORDER BY 
    trip_amount DESC;
```
Resultado:

```sql
company_name	         trip_amount
Flash Cab	                19558
Taxi Affiliation Services	11422
Medallion Leasin	        10367
Yellow Cab	               9888
Taxi Affiliation Service Yellow	9299
Chicago Carriage Cab Corp 	9181
City Service             	8448
...
```


Los resultados mostraron que las empresas más populares en ese período fueron Flash Cab con 19.558 viajes y Taxi Affiliation Services con 11.422 viajes, seguidas por otras compañías como Medallion Leasing y Yellow Cab. Estos datos se agruparon y ordenaron en función de la cantidad de viajes (trips_amount), proporcionando una visión clara de qué compañías dominaban el mercado durante ese período específico.

A continuación, se realizó un análisis para identificar las empresas de taxis cuyos nombres contenían las palabras "Yellow" o "Blue" entre el 1 y el 7 de noviembre de 2017. Este análisis agrupó los resultados por company_name y calculó el total de viajes (trips_amount) para cada empresa.

```sql

SELECT 
    cabs.company_name AS company_name,
    COUNT(DISTINCT trips.trip_id) AS trip_amount
FROM 
    cabs
    INNER JOIN trips ON trips.cab_id = cabs.cab_id
    
WHERE 
    (cabs.company_name LIKE '%Blue%' OR cabs.company_name LIKE'%Yellow%') AND
    CAST(trips.start_ts AS date) BETWEEN '2017-11-01' AND '2017-11-07'
    
GROUP BY 
    cabs.company_name
```
Resultado:

```sql
company_name	                  trip_amount
Blue Diamond	                    6764
Blue Ribbon Taxi Association Inc.	17675
Taxi Affiliation Service Yellow	  29213
Yellow Cab	                      33668

```


Las empresas más destacadas en este grupo fueron Blue Diamond y Yellow Cab, con un total significativo de 33.668 y 29.213 viajes respectivamente.

El último análisis exploratorio del paso 2 consistió en agrupar los viajes de Flash Cab y Taxi Affiliation Services en su propia categoría, mientras que todos los demás viajes se agruparon bajo el grupo "Other". 
```sql

SELECT 
    
    CASE 
        WHEN cabs.company_name IN ('Flash Cab', 'Taxi Affiliation Services') 
        THEN company_name
        ELSE 'Other'
        END AS company,
    COUNT(DISTINCT trips.trip_id) AS trips_amount
FROM 
    cabs
    INNER JOIN trips ON trips.cab_id = cabs.cab_id
    
WHERE 
    CAST(trips.start_ts AS date) BETWEEN '2017-11-01' AND '2017-11-07'
    
GROUP BY 
    CASE 
        WHEN cabs.company_name IN ('Flash Cab', 'Taxi Affiliation Services') 
        THEN company_name
        ELSE 'Other'
    END 
ORDER BY 
    trips_amount DESC
```
Resultado:

```sql
company	                 trips_amount
Other	                    335771
Flash Cab	                64084
Taxi Affiliation Services	37583


```


Al hacer esto, se obtuvo un total de 335.771 viajes en la categoría "Other", con Flash Cab y Taxi Affiliation Services como las empresas con el mayor número de viajes individuales. Este análisis es clave para entender las dos compañías más competitivas en el mercado de taxis durante noviembre de 2017.


#### Paso 3 
Esta etapa consistió en probar la hipótesis de que la duración de los viajes desde el Loop hasta el Aeropuerto Internacional O'Hare cambia durante los sábados lluviosos. Para ello, se recuperaron los identificadores de los barrios de O'Hare y Loop,
```sql
SELECT 
    neighborhood_id,
    name
FROM 
    neighborhoods

WHERE 
    name LIKE '%Hare' OR name LIKE 'Loop'

```
Resultado:
```sql
neighborhood_id	name
50	            Loop
63	            O'Hare

```

y se segmentaron las horas del día en dos grupos: "Bad" (si las condiciones climáticas eran lluviosas o tormentosas) y "Good" (para el resto de las condiciones).
```sql
SELECT 
    ts,
    CASE 
        WHEN description LIKE '%rain%' OR description LIKE '%storm%' 
        THEN 'Bad'
        ELSE 'Good' 
    END AS WEATHER_RECORDS

FROM  
    weather_records
```
Resultado:
```sql
ts	weather_records
2017-11-01 00:00:00	Good
2017-11-01 01:00:00	Good
2017-11-01 02:00:00	Good
2017-11-01 03:00:00	Good
2017-11-01 04:00:00	Good
...
```

Se obtuvo un conjunto de datos que relaciona cada hora con su condición climática, lo que permitió vincular estas condiciones con la duración de los viajes de los sábados.

Finalmente, se filtraron los viajes que comenzaron en el Loop y finalizaron en O'Hare durante los sábados.
```sql
SELECT 
    trips.start_ts AS start_ts,
    CASE 
        WHEN weather_records.description LIKE '%rain%' OR 
        weather_records.description LIKE '%storm%' 
        THEN 'Bad'
        ELSE 'Good' 
    END AS weather_records,
    trips.duration_seconds AS duration_seconds
    
FROM 
    trips
    INNER JOIN weather_records ON weather_records.ts = trips.start_ts
WHERE 
    trips.pickup_location_id = 50 AND trips.dropoff_location_id = 63 AND EXTRACT(DOW from trips.start_ts) = 6
ORDER BY 
    trips.trip_id
```
Resultado:
```sql
start_ts	weather_records	duration_seconds
2017-11-25 12:00:00	Good	1380
2017-11-25 16:00:00	Good	2410
2017-11-25 14:00:00	Good	1920
2017-11-25 12:00:00	Good	1543
2017-11-04 10:00:00	Good	2512
2017-11-11 07:00:00	Good	1440
2017-11-11 04:00:00	Good	1320
```

Para estos viajes, se recuperaron tanto las condiciones climáticas específicas de cada hora como la duración de cada viaje, lo que será fundamental para la prueba de hipótesis posterior.

[Continuación pasos siguientes y conclusión final]()

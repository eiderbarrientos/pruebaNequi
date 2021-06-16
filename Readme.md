# PRUEBA NEQUI. 
## DATASET consumo de energía eléctrica en un hogar.
**pregunta de negocio**
¿Queremos saber la tendencia de consumo de energía en el hogar en el año 2008 y cual de  submedición de “cocina, zona de ropas y calefacción” presento mayor  en el mismo año?

**Informacion del Dataset:**

**URL:** https://archive.ics.uci.edu/ml/datasets/Individual+household+electric+power+consumption

Mediciones del consumo de energía eléctrica en un hogar con una frecuencia de muestreo de un minuto durante un período de casi 4 años. Se encuentran disponibles diferentes magnitudes eléctricas y algunos valores de submedición.

Este archivo contiene 2075259 mediciones recopiladas en una casa ubicada en Sceaux (7 km de París, Francia) entre diciembre de 2006 y noviembre de 2010 (47 meses).

Notas:
 (potencia activa global * 1000/60 - sub_metering_1 - sub_metering_2 - sub_metering_3) representa la energía activa consumida cada minuto (en vatios hora) en el hogar por equipos eléctricos no medidos en los submedidores 1, 2 y 3.
 
 **Información de atributos:**
1. date: fecha en formato dd / mm / aaaa
2. time: time en formato hh: mm: ss
3. global_active_power: potencia activa promedio por minuto global del hogar (en kilovatios)
4. potencia_reactiva_global: potencia reactiva promediada por minuto global del hogar (en kilovatios)
5. voltaje: voltaje promediado por minutos (en voltios)
6. global_intensity: intensidad de corriente promedio por minuto global del hogar (en amperios)
7. sub_metering_1: submedición de energía nº 1 (en vatios-hora de energía activa). Corresponde a la cocina, que contiene principalmente lavavajillas, horno y microondas (los fogones no son eléctricos sino de gas).
8. sub_metering_2: submedición de energía nº 2 (en vatios-hora de energía activa). Corresponde al lavadero, que contiene lavadora, secadora, frigorífico y luz.
9. sub_metering_3: submedición de energía nº 3 (en vatios-hora de energía activa). Corresponde a un calentador de agua eléctrico y un acondicionador de aire.

**nota**
*el dataset fue subido a github para que pueda ser utilizado desde este notebook*

### Arquitectura propuesta para desarrollar el ejercicio.

1.	Se propone disponer de dos zonas (Buckets) en el S3 de aws :
    * prueba-nequi-raw : zona cruda donde se realizará ingestas de archivos orginales.
    * prueba-nequi-stage zona stage donde se realizará ingestas de archivos limpios y transformados en formato parquet para su consumo.
2.	Se consumen tres servicios de aws para consumo la data:
    * Glue como catálogo de datos.
    * Athena para realizar querys.
    * Quicksight para realizar los dashboard con los que se consumen la data.
3.	Se utilizan dos librearías de interacción entre el notebook y el AWS para realizar las tareas de carga a cada zona y las transformaciones correspondientes.
    * Boto3
    * AWS Data Wrangler
    
**Diagrama **
![diagrama.jpg](https://github.com/eiderbarrientos/pruebaNequi/blob/main/diagrama.jpg)

instalación de librerias:

* seaborn -> Necesaria para realizar graficos y visualizar data.
* Boto3 -> Necesaria para realizar sesiones entre el notebook y AWS.
* awswrangler -> Libreria extendida de pandas de AWS para realizar conexión entre data frames y servicios de AWS.


```python
pip install seaborn
pip install boto3
pip install awswrangler
```

    Requirement already satisfied: boto3 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (1.17.49)
    Requirement already satisfied: jmespath<1.0.0,>=0.7.1 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from boto3) (0.10.0)
    Requirement already satisfied: s3transfer<0.4.0,>=0.3.0 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from boto3) (0.3.6)
    Requirement already satisfied: botocore<1.21.0,>=1.20.49 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from boto3) (1.20.49)
    Requirement already satisfied: python-dateutil<3.0.0,>=2.1 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from botocore<1.21.0,>=1.20.49->boto3) (2.8.1)
    Requirement already satisfied: urllib3<1.27,>=1.25.4 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from botocore<1.21.0,>=1.20.49->boto3) (1.25.8)
    Requirement already satisfied: six>=1.5 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from python-dateutil<3.0.0,>=2.1->botocore<1.21.0,>=1.20.49->boto3) (1.15.0)
    Note: you may need to restart the kernel to use updated packages.
    

importamos librearias necesarias


```python
import urllib.request, json
import pandas as pd
import zipfile
import os
import io
import boto3
import logging
from botocore.exceptions import ClientError
import seaborn as sns
import awswrangler as wr
```

Creamos directorio para logs de carga


```python
logger = logging.getLogger('ejemplo_Log')
logger.setLevel(logging.DEBUG)
fh = logging.FileHandler('debug.log')
fh.setLevel(logging.DEBUG)
logger.addHandler(fh)
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
fh.setFormatter(formatter)
logger.addHandler(fh)
logger.info('Inicio del proceso')
```

Leemos dataset consumo de energia desde github.


```python
url='https://github.com/eiderbarrientos/pruebaNequi/raw/main/household_power_consumption.zip'
df = pd.read_csv(url, compression='zip', sep=';',low_memory=False)
```


```python
df.head(10)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Date</th>
      <th>Time</th>
      <th>Global_active_power</th>
      <th>Global_reactive_power</th>
      <th>Voltage</th>
      <th>Global_intensity</th>
      <th>Sub_metering_1</th>
      <th>Sub_metering_2</th>
      <th>Sub_metering_3</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>16/12/2006</td>
      <td>17:24:00</td>
      <td>4.216</td>
      <td>0.418</td>
      <td>234.840</td>
      <td>18.400</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>16/12/2006</td>
      <td>17:25:00</td>
      <td>5.360</td>
      <td>0.436</td>
      <td>233.630</td>
      <td>23.000</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>16.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>16/12/2006</td>
      <td>17:26:00</td>
      <td>5.374</td>
      <td>0.498</td>
      <td>233.290</td>
      <td>23.000</td>
      <td>0.000</td>
      <td>2.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>16/12/2006</td>
      <td>17:27:00</td>
      <td>5.388</td>
      <td>0.502</td>
      <td>233.740</td>
      <td>23.000</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>16/12/2006</td>
      <td>17:28:00</td>
      <td>3.666</td>
      <td>0.528</td>
      <td>235.680</td>
      <td>15.800</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>16/12/2006</td>
      <td>17:29:00</td>
      <td>3.520</td>
      <td>0.522</td>
      <td>235.020</td>
      <td>15.000</td>
      <td>0.000</td>
      <td>2.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>16/12/2006</td>
      <td>17:30:00</td>
      <td>3.702</td>
      <td>0.520</td>
      <td>235.090</td>
      <td>15.800</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>16/12/2006</td>
      <td>17:31:00</td>
      <td>3.700</td>
      <td>0.520</td>
      <td>235.220</td>
      <td>15.800</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>16/12/2006</td>
      <td>17:32:00</td>
      <td>3.668</td>
      <td>0.510</td>
      <td>233.990</td>
      <td>15.800</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>16/12/2006</td>
      <td>17:33:00</td>
      <td>3.662</td>
      <td>0.510</td>
      <td>233.860</td>
      <td>15.800</td>
      <td>0.000</td>
      <td>2.000</td>
      <td>16.0</td>
    </tr>
  </tbody>
</table>
</div>




```python
df.shape
```




    (2075259, 9)



Credenciales de AWS de nuestra cuenta.


```python
AWS_ACCESS_KEY_ID = "AKIAWH3A7HI4IQG4TV6I"
AWS_SECRET_ACCESS_KEY = "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"
```


```python
file_name='household_power_consumption.csv'
AWS_S3_BUCKET = 'prueba-nequi-raw'
```

Subimos archivo a la zona cruda de AWS BUCKET *prueba-nequi-raw*


```python
csv_buffer=io.StringIO()
df.to_csv(csv_buffer, sep=";", index=False)

s3_client = boto3.client(
    's3',
    aws_access_key_id=AWS_ACCESS_KEY_ID,
    aws_secret_access_key=AWS_SECRET_ACCESS_KEY
)
    
response = s3_client.put_object(
    ACL='private',
    Body=csv_buffer.getvalue(),
    Bucket=AWS_S3_BUCKET,
    Key = file_name
)

status = response.get("ResponseMetadata", {}).get("HTTPStatusCode")

if status == 200:
    print(f"Successful S3 put_object response. Status - {status} " + file_name)
    logger.info(f"Successful S3 put_object response. Status - {status} - " + file_name)
else:
    logger.info(f"Successful S3 put_object response. Status - {status} - " + file_name)
```

    Successful S3 put_object response. Status - 200 household_power_consumption.csv
    

evidenciamos como se carga el archivo en la zona raw.

![raw.jpg](attachment:raw.jpg)

Desde bucket 'prueba-nequi-raw' cargamos el dataset para tranformarlos y limpiarlo.


```python
session = boto3.session.Session(
    aws_access_key_id=AWS_ACCESS_KEY_ID,
    aws_secret_access_key=AWS_SECRET_ACCESS_KEY
)

raw_s3_bucket ='prueba-nequi-raw'
raw_path = f"s3://{raw_s3_bucket}/{file_name}"
raw_df = wr.s3.read_csv(path=raw_path,
                        boto3_session=session,
                        sep=';',
                        low_memory=False
                       )
```


```python
raw_df.head(10)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Date</th>
      <th>Time</th>
      <th>Global_active_power</th>
      <th>Global_reactive_power</th>
      <th>Voltage</th>
      <th>Global_intensity</th>
      <th>Sub_metering_1</th>
      <th>Sub_metering_2</th>
      <th>Sub_metering_3</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>16/12/2006</td>
      <td>17:24:00</td>
      <td>4.216</td>
      <td>0.418</td>
      <td>234.840</td>
      <td>18.400</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>16/12/2006</td>
      <td>17:25:00</td>
      <td>5.360</td>
      <td>0.436</td>
      <td>233.630</td>
      <td>23.000</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>16.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>16/12/2006</td>
      <td>17:26:00</td>
      <td>5.374</td>
      <td>0.498</td>
      <td>233.290</td>
      <td>23.000</td>
      <td>0.000</td>
      <td>2.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>16/12/2006</td>
      <td>17:27:00</td>
      <td>5.388</td>
      <td>0.502</td>
      <td>233.740</td>
      <td>23.000</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>16/12/2006</td>
      <td>17:28:00</td>
      <td>3.666</td>
      <td>0.528</td>
      <td>235.680</td>
      <td>15.800</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>16/12/2006</td>
      <td>17:29:00</td>
      <td>3.520</td>
      <td>0.522</td>
      <td>235.020</td>
      <td>15.000</td>
      <td>0.000</td>
      <td>2.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>16/12/2006</td>
      <td>17:30:00</td>
      <td>3.702</td>
      <td>0.520</td>
      <td>235.090</td>
      <td>15.800</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>16/12/2006</td>
      <td>17:31:00</td>
      <td>3.700</td>
      <td>0.520</td>
      <td>235.220</td>
      <td>15.800</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>16/12/2006</td>
      <td>17:32:00</td>
      <td>3.668</td>
      <td>0.510</td>
      <td>233.990</td>
      <td>15.800</td>
      <td>0.000</td>
      <td>1.000</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>16/12/2006</td>
      <td>17:33:00</td>
      <td>3.662</td>
      <td>0.510</td>
      <td>233.860</td>
      <td>15.800</td>
      <td>0.000</td>
      <td>2.000</td>
      <td>16.0</td>
    </tr>
  </tbody>
</table>
</div>




```python
raw_df.shape
```




    (2075259, 9)




```python
raw_df.duplicated().sum()
```




    0




```python
raw_df.isna().sum()
```




    Date                         0
    Time                         0
    Global_active_power          0
    Global_reactive_power        0
    Voltage                      0
    Global_intensity             0
    Sub_metering_1               0
    Sub_metering_2               0
    Sub_metering_3           25979
    dtype: int64



Eliminamos valores que se encuentran vacios.


```python
df_cleaned = raw_df.dropna()
```


```python
df_cleaned.shape
```




    (2049280, 9)




```python
df_cleaned.isna().sum()
```




    Date                     0
    Time                     0
    Global_active_power      0
    Global_reactive_power    0
    Voltage                  0
    Global_intensity         0
    Sub_metering_1           0
    Sub_metering_2           0
    Sub_metering_3           0
    dtype: int64



transformamos tipos de datos.


```python
df_cleaned['Date']= pd.to_datetime(df_cleaned['Date'],format='%d/%m/%Y')
```


```python
df_cleaned['Time']= pd.to_datetime(df_cleaned['Time'],format='%H:%M:%S').dt.time
```


```python
df_cleaned['Global_active_power']= pd.to_numeric(df_cleaned['Global_active_power'])
df_cleaned['Global_reactive_power']= pd.to_numeric(df_cleaned['Global_reactive_power'])
df_cleaned['Voltage']= pd.to_numeric(df_cleaned['Voltage'])
df_cleaned['Global_intensity']= pd.to_numeric(df_cleaned['Global_intensity'])
df_cleaned['Sub_metering_1']= pd.to_numeric(df_cleaned['Sub_metering_1'])
df_cleaned['Sub_metering_2']= pd.to_numeric(df_cleaned['Sub_metering_2'])
```


```python
df_cleaned.dtypes
```




    Date                     datetime64[ns]
    Time                             object
    Global_active_power             float64
    Global_reactive_power           float64
    Voltage                         float64
    Global_intensity                float64
    Sub_metering_1                  float64
    Sub_metering_2                  float64
    Sub_metering_3                  float64
    dtype: object



Renombramos Columnas


```python
df_cleaned = df_cleaned.rename(columns={'Date':'fecha',
                                        'Time':'hora',
                                        'Global_active_power':'potencia_activa_kw',
                                        'Global_reactive_power':'potencia_reactiva_kw',
                                        'Voltage':'voltaje',
                                        'Global_intensity':'intensidad_corriente_amp',
                                        'Sub_metering_1':'submedicion_cocina_w_hora',
                                        'Sub_metering_2':'submedicion_zona_ropa_w_hora',
                                        'Sub_metering_3':'submedicion_calefacion_w_hora'
                                       }
                              )
```


```python
df_cleaned.head(10)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>fecha</th>
      <th>hora</th>
      <th>potencia_activa_kw</th>
      <th>potencia_reactiva_kw</th>
      <th>voltaje</th>
      <th>intensidad_corriente_amp</th>
      <th>submedicion_cocina_w_hora</th>
      <th>submedicion_zona_ropa_w_hora</th>
      <th>submedicion_calefacion_w_hora</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2006-12-16</td>
      <td>17:24:00</td>
      <td>4.216</td>
      <td>0.418</td>
      <td>234.84</td>
      <td>18.4</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2006-12-16</td>
      <td>17:25:00</td>
      <td>5.360</td>
      <td>0.436</td>
      <td>233.63</td>
      <td>23.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>16.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2006-12-16</td>
      <td>17:26:00</td>
      <td>5.374</td>
      <td>0.498</td>
      <td>233.29</td>
      <td>23.0</td>
      <td>0.0</td>
      <td>2.0</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2006-12-16</td>
      <td>17:27:00</td>
      <td>5.388</td>
      <td>0.502</td>
      <td>233.74</td>
      <td>23.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2006-12-16</td>
      <td>17:28:00</td>
      <td>3.666</td>
      <td>0.528</td>
      <td>235.68</td>
      <td>15.8</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>2006-12-16</td>
      <td>17:29:00</td>
      <td>3.520</td>
      <td>0.522</td>
      <td>235.02</td>
      <td>15.0</td>
      <td>0.0</td>
      <td>2.0</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>2006-12-16</td>
      <td>17:30:00</td>
      <td>3.702</td>
      <td>0.520</td>
      <td>235.09</td>
      <td>15.8</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>2006-12-16</td>
      <td>17:31:00</td>
      <td>3.700</td>
      <td>0.520</td>
      <td>235.22</td>
      <td>15.8</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>2006-12-16</td>
      <td>17:32:00</td>
      <td>3.668</td>
      <td>0.510</td>
      <td>233.99</td>
      <td>15.8</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>2006-12-16</td>
      <td>17:33:00</td>
      <td>3.662</td>
      <td>0.510</td>
      <td>233.86</td>
      <td>15.8</td>
      <td>0.0</td>
      <td>2.0</td>
      <td>16.0</td>
    </tr>
  </tbody>
</table>
</div>



## Análisis exploratorio de los datos EDA


```python
df_cleaned.describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>potencia_activa_kw</th>
      <th>potencia_reactiva_kw</th>
      <th>voltaje</th>
      <th>intensidad_corriente_amp</th>
      <th>submedicion_cocina_w_hora</th>
      <th>submedicion_zona_ropa_w_hora</th>
      <th>submedicion_calefacion_w_hora</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>2.049280e+06</td>
      <td>2.049280e+06</td>
      <td>2.049280e+06</td>
      <td>2.049280e+06</td>
      <td>2.049280e+06</td>
      <td>2.049280e+06</td>
      <td>2.049280e+06</td>
    </tr>
    <tr>
      <th>mean</th>
      <td>1.091615e+00</td>
      <td>1.237145e-01</td>
      <td>2.408399e+02</td>
      <td>4.627759e+00</td>
      <td>1.121923e+00</td>
      <td>1.298520e+00</td>
      <td>6.458447e+00</td>
    </tr>
    <tr>
      <th>std</th>
      <td>1.057294e+00</td>
      <td>1.127220e-01</td>
      <td>3.239987e+00</td>
      <td>4.444396e+00</td>
      <td>6.153031e+00</td>
      <td>5.822026e+00</td>
      <td>8.437154e+00</td>
    </tr>
    <tr>
      <th>min</th>
      <td>7.600000e-02</td>
      <td>0.000000e+00</td>
      <td>2.232000e+02</td>
      <td>2.000000e-01</td>
      <td>0.000000e+00</td>
      <td>0.000000e+00</td>
      <td>0.000000e+00</td>
    </tr>
    <tr>
      <th>25%</th>
      <td>3.080000e-01</td>
      <td>4.800000e-02</td>
      <td>2.389900e+02</td>
      <td>1.400000e+00</td>
      <td>0.000000e+00</td>
      <td>0.000000e+00</td>
      <td>0.000000e+00</td>
    </tr>
    <tr>
      <th>50%</th>
      <td>6.020000e-01</td>
      <td>1.000000e-01</td>
      <td>2.410100e+02</td>
      <td>2.600000e+00</td>
      <td>0.000000e+00</td>
      <td>0.000000e+00</td>
      <td>1.000000e+00</td>
    </tr>
    <tr>
      <th>75%</th>
      <td>1.528000e+00</td>
      <td>1.940000e-01</td>
      <td>2.428900e+02</td>
      <td>6.400000e+00</td>
      <td>0.000000e+00</td>
      <td>1.000000e+00</td>
      <td>1.700000e+01</td>
    </tr>
    <tr>
      <th>max</th>
      <td>1.112200e+01</td>
      <td>1.390000e+00</td>
      <td>2.541500e+02</td>
      <td>4.840000e+01</td>
      <td>8.800000e+01</td>
      <td>8.000000e+01</td>
      <td>3.100000e+01</td>
    </tr>
  </tbody>
</table>
</div>




```python
sns.pairplot(df_cleaned)
```




    <seaborn.axisgrid.PairGrid at 0x1e5823a3780>




    
![png](output_40_1.png)
    


Correlacion de los datos entre las columnas Potencia Activa en KW y Intensidad corriente de Amp


```python
sns.heatmap(df_cleaned.corr(),annot=True,cmap='magma')
```




    <AxesSubplot:>




    
![png](output_42_1.png)
    


Parametros para cargar el archivo parquet y crear el catalogo en Glu


```python
stage_s3_bucket ='prueba-nequi-stage'
stage_path_dir = 'Consumo_energia_hogar/'
stage_path = f"s3://{stage_s3_bucket}/{stage_path_dir}"


database='consumo_energia'
table='consumo_energia_hogar_x_min'
dtype={
    'fecha':'date',
    'hora':'string',
    'potencia_activa_kw':'double',
    'potencia_reactiva_kw':'double',
    'voltaje':'double',
    'intensidad_corriente_amp':'double',
    'submedicion_cocina_w_hora':'double',
    'submedicion_zona_ropa_w_hora':'double',
    'submedicion_calefacion_w_hora':'double'
}
description = 'Mediciones del consumo de energía eléctrica en un hogar con una frecuencia de muestreo de un minuto'
comments = {
    'fecha':'fecha de captura de medicion',
    'hora':'fecha de captura de medicion',
    'potencia_activa_kw':'potencia activa promedio por minuto global del hogar - en kilovatios',
    'potencia_reactiva_kw':': potencia reactiva promediada por minuto global del hogar - en kilovatios',
    'voltaje':'voltaje promediado por minutos en voltios',
    'intensidad_corriente_amp':'intensidad de corriente promedio por minuto global del hogar - en amperios',
    'submedicion_cocina_w_hora':'submedición en vatios-hora de energía activa. Corresponde a la cocina, que contiene lavavajillas, horno y microondas',
    'submedicion_zona_ropa_w_hora':'submedición en vatios-hora de energía activa. Corresponde al lavadero, que contiene lavadora, secadora, frigorífico y luz',
    'submedicion_calefacion_w_hora':'submedición en vatios-hora de energía activa. Corresponde a un calentador de agua eléctrico y un acondicionador de aire'    
}

```

Carga del data set transformado a la zona Stage 'prueba-nequi-stage'



```python
session = boto3.session.Session(
    aws_access_key_id=AWS_ACCESS_KEY_ID,
    aws_secret_access_key=AWS_SECRET_ACCESS_KEY,
    region_name='sa-east-1'
)

wr.s3.to_parquet(
    df=df_cleaned,
    path=stage_path,
    boto3_session=session,
    dataset = True,
    mode ='overwrite',
    database = database,
    table = table,   
    dtype =dtype,
    description = description,
    columns_comments = comments  
)
```




    {'paths': ['s3://prueba-nequi-stage/Consumo_energia_hogar/e56c58c71d5d4f94a5a2d1058b3f3cff.snappy.parquet'],
     'partitions_values': {}}



![parquet_energia.jpg](attachment:parquet_energia.jpg)

tablas en GLU Creadas

![glue1.jpg](attachment:glue1.jpg)

Query de consumo desde athena.

![athenaQuery.jpg](attachment:athenaQuery.jpg)

consumimos datos para responder la pregunta de negocio. **Agosto** es la respuesta a la pregunta de negocio y la submedicion que mas consume es la de la calefaccion con color azul.



![dash2.jpg](attachment:dash2.jpg)

![dash.jpg](attachment:dash.jpg)

# DATASET - 2  Spotify PlayList

Conjunto de datos de listas de reproducción de Spotify

URL: https://www.kaggle.com/ashwinik/spotify-playlist

Este conjunto de datos se basa en el subconjunto de usuarios que publican sus tweets #nowplaying a través de Spotify. En principio, el conjunto de datos contiene a los usuarios, sus listas de reproducción y las pistas contenidas en estas listas de reproducción.

El archivo csv que contiene el conjunto de datos contiene las siguientes columnas:
* user_id: es un hash del nombre de usuario de Spotify del usuario
* artistname: es el nombre del artista
* trackname: es el título de la pista y
* playlistname: es el nombre de la lista de reproducción que contiene esta pista.

**Pregunta de nogocio**
*Cuales son las 20 canciones mas utilizadas en las listas de reproducción creadas por los usuarios.
*Cuales son los 20 Artistas mas utilizadas en las listas de reproducción creadas por los usuarios.

*nota este data se consmunio desde el api de kaggle en notebook*

Instalamos el api de Kaggle en notebook.


```python
!pip install kaggle
!mkdir .kaggle
```

    Requirement already satisfied: kaggle in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (1.5.12)
    Requirement already satisfied: python-dateutil in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from kaggle) (2.8.1)
    Requirement already satisfied: python-slugify in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from kaggle) (5.0.2)
    Requirement already satisfied: six>=1.10 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from kaggle) (1.15.0)
    Requirement already satisfied: certifi in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from kaggle) (2021.5.30)
    Requirement already satisfied: urllib3 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from kaggle) (1.25.8)
    Requirement already satisfied: requests in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from kaggle) (2.25.1)
    Requirement already satisfied: tqdm in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from kaggle) (4.59.0)
    Requirement already satisfied: text-unidecode>=1.3 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from python-slugify->kaggle) (1.3)
    Requirement already satisfied: idna<3,>=2.5 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from requests->kaggle) (2.10)
    Requirement already satisfied: chardet<5,>=3.0.2 in d:\users\ebarrien\appdata\local\continuum\anaconda3\lib\site-packages (from requests->kaggle) (4.0.0)
    


```python
from kaggle.api.kaggle_api_extended import KaggleApi
api = KaggleApi()
api.authenticate()
```

Descargamos data set desde el api


```python
api.dataset_download_file('ashwinik/spotify-playlist','spotify_dataset.csv')
```




    True



Por medio de Boto3 subimos dataset al AWS a la zona Raw


```python
AWS_ACCESS_KEY_ID = "AKIAWH3A7HI4IQG4TV6I"
AWS_SECRET_ACCESS_KEY = "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"
```


```python
file_name='spotify_dataset.csv.zip'
object_name='spotify_playlist.csv.zip'
AWS_S3_BUCKET = 'prueba-nequi-raw'
```


```python
s3_client = boto3.client(
    's3',
    aws_access_key_id=AWS_ACCESS_KEY_ID,
    aws_secret_access_key=AWS_SECRET_ACCESS_KEY
)

def upload_file(file_name, bucket, object_name=None):
    if object_name is None:
        object_name = file_name
    try:        
        response = s3_client.upload_file('spotify_dataset.csv.zip',AWS_S3_BUCKET,object_name,ExtraArgs={'ServerSideEncryption': "AES256"})
    except ClientError as e:
        logging.error(e)
        return False
    return True

upload_file(file_name, AWS_S3_BUCKET, object_name)
```




    True



![image.png](attachment:image.png)

cargamos al notebook dataset para realizar transformaciones y limpieza


```python
session = boto3.session.Session(
    aws_access_key_id=AWS_ACCESS_KEY_ID,
    aws_secret_access_key=AWS_SECRET_ACCESS_KEY
)

raw_s3_bucket ='prueba-nequi-raw'
raw_path = f"s3://{raw_s3_bucket}/{object_name}"
raw_df_spotify = wr.s3.read_csv(path=raw_path,
                        boto3_session=session,
                        low_memory=False,
                        compression='zip',
                        escapechar= '.',
                        error_bad_lines = False,
                        warn_bad_lines=False
                       )
```


```python
raw_df_spotify.head(10)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user_id</th>
      <th>"artistname"</th>
      <th>"trackname"</th>
      <th>"playlistname"</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>9cc0cfd4d7d7885102480dd99e7a90d6</td>
      <td>Elvis Costello</td>
      <td>(The Angels Wanna Wear My) Red Shoes</td>
      <td>HARD ROCK 2010</td>
    </tr>
    <tr>
      <th>1</th>
      <td>9cc0cfd4d7d7885102480dd99e7a90d6</td>
      <td>Elvis Costello &amp; The Attractions</td>
      <td>(What's So Funny 'Bout) Peace, Love And Unders...</td>
      <td>HARD ROCK 2010</td>
    </tr>
    <tr>
      <th>2</th>
      <td>9cc0cfd4d7d7885102480dd99e7a90d6</td>
      <td>Tiffany Page</td>
      <td>7 Years Too Late</td>
      <td>HARD ROCK 2010</td>
    </tr>
    <tr>
      <th>3</th>
      <td>9cc0cfd4d7d7885102480dd99e7a90d6</td>
      <td>Elvis Costello &amp; The Attractions</td>
      <td>Accidents Will Happen</td>
      <td>HARD ROCK 2010</td>
    </tr>
    <tr>
      <th>4</th>
      <td>9cc0cfd4d7d7885102480dd99e7a90d6</td>
      <td>Elvis Costello</td>
      <td>Alison</td>
      <td>HARD ROCK 2010</td>
    </tr>
    <tr>
      <th>5</th>
      <td>9cc0cfd4d7d7885102480dd99e7a90d6</td>
      <td>Lissie</td>
      <td>All Be Okay</td>
      <td>HARD ROCK 2010</td>
    </tr>
    <tr>
      <th>6</th>
      <td>9cc0cfd4d7d7885102480dd99e7a90d6</td>
      <td>Paul McCartney</td>
      <td>Band On The Run</td>
      <td>HARD ROCK 2010</td>
    </tr>
    <tr>
      <th>7</th>
      <td>9cc0cfd4d7d7885102480dd99e7a90d6</td>
      <td>Joe Echo</td>
      <td>Beautiful</td>
      <td>HARD ROCK 2010</td>
    </tr>
    <tr>
      <th>8</th>
      <td>9cc0cfd4d7d7885102480dd99e7a90d6</td>
      <td>Paul McCartney</td>
      <td>Blackbird - Live at CitiField, NYC - Digital A...</td>
      <td>HARD ROCK 2010</td>
    </tr>
    <tr>
      <th>9</th>
      <td>9cc0cfd4d7d7885102480dd99e7a90d6</td>
      <td>Lissie</td>
      <td>Bright Side</td>
      <td>HARD ROCK 2010</td>
    </tr>
  </tbody>
</table>
</div>




```python
raw_df_spotify.shape
```




    (12774191, 4)




```python
raw_df_spotify.dtypes
```




    user_id            object
     "artistname"      object
     "trackname"       object
     "playlistname"    object
    dtype: object




```python
raw_df_spotify.duplicated().sum()
```




    224




```python
spotify_cleaned=raw_df_spotify.drop_duplicates()
```


```python
spotify_cleaned.columns = ['user_id','artistname','trackname','playlistname']
```


```python
spotify_cleaned.dtypes
```




    user_id         object
    artistname      object
    trackname       object
    playlistname    object
    dtype: object




```python
spotify_cleaned.isna().sum()
```




    user_id              0
    artistname       33089
    trackname          101
    playlistname    139593
    dtype: int64




```python
spotify_cleaned = raw_df_spotify.dropna()
```


```python
spotify_cleaned.isna().sum()
```




    user_id         0
    artistname      0
    trackname       0
    playlistname    0
    dtype: int64




```python
spotify_resumen = spotify_cleaned.groupby(['user_id'])["playlistname"].nunique().reset_index()
sns.histplot(spotify_resumen["playlistname"])
```




    <AxesSubplot:xlabel='playlistname', ylabel='Count'>




    
![png](output_76_1.png)
    



```python
spotify_cleaned["trackname"].value_counts()[0:51].plot(kind= 'bar',title="50 canciones mas importantes",figsize = (16,6))
```




    <AxesSubplot:title={'center':'50 canciones mas importantes'}>




    
![png](output_77_1.png)
    


Cargamos nuevamente dataset al AWS en la zona Stage


```python
stage_s3_bucket ='prueba-nequi-stage'
stage_path_dir = 'spotify_playlist/'
stage_path = f"s3://{stage_s3_bucket}/{stage_path_dir}"


database='spotify_playlist'
table='data_spotify_playlist'
dtype={
    'user_id':'string',
    'artistname':'string',
    'trackname':'string',
    'playlistname':'string'
}
description = 'contiene a los usuarios sus listas de reproducción y las pistas contenidas en estas listas de reproducción'
comments = {
    'user_id':'es un hash del nombre de usuario de Spotify del usuario',
    'artistname':'es el nombre del artista',
    'trackname':'es el título de la pista',
    'playlistname':'es el nombre de la lista de reproducción que contiene esta pista'
}

```


```python
session = boto3.session.Session(
    aws_access_key_id=AWS_ACCESS_KEY_ID,
    aws_secret_access_key=AWS_SECRET_ACCESS_KEY,
    region_name='sa-east-1'
)

wr.s3.to_parquet(
    df=spotify_cleaned,
    path=stage_path,
    boto3_session=session,
    dataset = True,
    mode ='overwrite',
    database = database,
    table = table,   
    dtype =dtype,
    description = description,
    columns_comments = comments  
)
```

    D:\Users\ebarrien\AppData\Local\Continuum\anaconda3\lib\site-packages\awswrangler\_data_types.py:653: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy
      df[col] = df[col].astype(desired_type)
    




    {'paths': ['s3://prueba-nequi-stage/spotify_playlist/0b7e417de7ae48eb9b7bfeace55ca0c7.snappy.parquet'],
     'partitions_values': {}}



![spoti.jpg](attachment:spoti.jpg)

consumimos data para visualizar dashboard

![spoti2.jpg](attachment:spoti2.jpg)

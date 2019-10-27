# Big Data Processing (Spark + Scala)
Práctica de Big Data Processing del Bootcamp Big Data &amp; Machine Learning de KeepCoding

## Desarrollo
Este repositorio describe cada una de las partes desarrolladas en la práctica del módulo Big Data Processing perteneciente al _Bootcamp Big Data & Machine Learning_ de [KeepCoding](https://keepcoding.io/es/)

Para el desarrollo de la práctica se ha creado un proyecto Spark con Intellij escrito en Scala y haciendo uso de sbt como gestor de dependencias.

A continuación, se describen los pasos desarrollados para la ejecución de la práctica:

### Datasets
En al carpeta datasets del proyecto tenemos todos las fuentes de datos utilizadas en este proyecto.

- blacklist_words.txt (lista negra de todas las palabras que la aplicación debe considerar para mandar la notificación al gobierno)
- iot_messages.dat (se trata de los mensajes ficticios capturados por los dispositivos de iot). En este caso he considerado que el procesamiento de los datos lo basaríamos sobre el tiempo del evento y por lo tanto en el propio mensaje he incluido el timestamp.
Date	Timestamp	MessageId	Message	IOT_Id	ScrUserId	DstUserId

|Atributo | Tipo|
|-----|------|
|EventTime|Timestamp|
|MessageId|Long|
|Message|String|
|IOT_Id|String|
|ScrUserId|Long|
|DstUserId|Long|

- stopwords.txt (son las palabras que no se considerarán como relevantes, en nuestro caso había que descartar determinantes, conjunciones y artículos)
- users.txt (contiene la información de los usuarios de la red como nombre, edad y sexo)

### Arquitectura y despliegue
Para el desarrollo y pruebas del sistema se ha considerado que los mensajes capturados por los IoTs se envían a un servidor de Kafka donde serán consumidos y procesados por una aplicación de Spark Streaming.

El topic creado para la recepción de los mensajes de los IoTs ha sido ‘iot’.  
```bash
bin\windows\kafka-topics.bat --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic iot
```
Una vez arrancada la aplicación de Spark, ésta se quedará a la espera de recibir las mensajes que se envíen a Kafka. Para la simulación se puede utilizar el fichero con todos los mensajes a procesar _iot_messages.dat_, o simular manualmente el envío de mensajes de forma que podamos jugar con los timestamps de los mensajes.  
```bash
bin\windows\kafka-console-producer.bat -broker-list localhost:9092 --topic iot < iot_messages.dat
```

### Código
Los pasos ejecutados dentro de la clase Main de Spark son:  
1. Levantar sesión de Spark
1. Cargar en dataframes las fuentes de datos estáticas (usuarios y stopwords). En este caso stopwords se creará como objecto broadcast para que cachearlo en cada uno de los nodos y que no se tenga que enviar cada vez que se tiene que consultar.
1. Crear el dataframe para la lectura en streaming de los datos de Kafka.
1. Limpieza y preparación de datos.  
    - Eliminar duplicados basándonos en el messageId y haciendo uso de la función _withWatermark_ para indicar la ventana de tiempo sobre la cual queremos evitar duplicidades,  esto último es necesario si no queremos mantener en memoria todos los mensajes recibidos a la hora de eliminar duplicidades.
    - Se ha creado una función encargada de processar cada una de los mensajes de la red
       -	Decodifica los mensajes (simulado)
       -	Elimina los acentos de las palabras y las convierte a minúsculas.
       -	Separa los mensajes en palabras.
       -	Filtra las palabras que tenemos en las lista de stopwords.  
1. Query con la agregación de las palabras en una ventana temporal de 1 hora. Para hacer las prueba se ha modificado a 1 minuto. Como estamos basándo el procesado en “event time” también se han utilizado las “watermarks” de Spark para gestionar los datos que pudieran llegar retrasados. La idea sería descartar aquellos mensajes que lleguen con un retraso superior a x minutos.
1. Crear 3 streamwriters de la query de agregación con un trigger de 1 hora para el procesado de mensajes. La idea es que cada hora salte el procesado para realizar la cuenta de palabras de esa hora.
    - A consola
    - A memoria (pruebas)
    - A fichero, por si nos interesara tener persistencia de los agregados calculados. Los ficheros se han particionado por hora.

### Estudio y dudas
**Stopwords** (palabras no relevantes). En spark ML disponemos de transformadores de texto como son _Tokenizer_ y _StopWordsRemover_ que podríamos haber utilizado para extraer las palabras y limpiar aquellas que se consideran con menos valor. Como en la práctica se nos enunciaba la eliminación concreta de árticulos, determinantes y conjunciones hemos optado por filtrar solamente estas palabras en concreto.  
Alguna de las conjunciones españolas están compuestas de varias palabras, pero para la realización de la práctica se han descompuesto, algunas de ellas, como palabras individuales a filtrar.

Una de las principales dudas que me ha surgido con esta práctica es cómo con la API structured streaming de Spark podemos obtener un **Top k** de un agregado de datos, ya que a la hora de hacer agregaciones en streaming ya se nos advierte que no podremos hacer ordenaciones ni limits.  
Por otro lado, podemos, con esta API, desarrollar procesos para que leer datos de diversas fuentes, hacer transformaciones y escribirlas en distintos soportes (fichero, consola, memoria,…), pero no veo claro donde podemos programar la lógica para realizar una acción en función de unos resultados obtenidos. En nuestro caso, teníamos que enviar una notificación de alerta en funcion de unas palabras detectadas.
Al parecer con los **Foreach Sink** podemos ejecutar acciones a partir del stream procesado, pero con este tipo de Sink solamente puedes evaluar row a row, así que nos podria servir para enviar una notificación evaluando row a row, pero no para calcular un _Top_ o _Rank_.  
Para ejecutar alguna acción también he valorado el uso de _Listeners asíncronos_. Mi idea era la de escuchar la ejecución del procesado de un micro-batch del streamWriter a memoria (haciendo uso de la API asincrona) y sobre ese evento ahcer una query y el envio de notificaciones (en el código se puede visualizar esta prueba).
El problema de esta solución es que la salida a memoria es una salida para debug, no la recomiendan para producción y además la query que ejecutas sobre la tabla en memoria siempre te devuelve todos los resultados por lo que tendríamos nosotros que filtrar por la hora que nos interesa interrrogar.

Continuará...


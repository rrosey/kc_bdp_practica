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


Spark

Apache Spark es un framework para el análisi procesamiento de grandes cantidades de datos 
Apache Spark es un motor para el procesamiento de grandes cantidades de datos
Escrito en Scala: Programacion funcional que corre en una JVM
Spark shell para python y scala
Spark applications en python y scala y java

Spark provee un stack de librerias que corren sobre el core Spark
	─ Core Spark provee los Resilient Distributed Datasets (RDDs)
	─ Spark SQL trabaja con datos estructurados
	─ MLlib supports scalable machine learning
	─ Spark Streaming applications process data in real time
	─ GraphX works with graphs and graph-parallel computation


Spark Shell
	Python > pyspark2
	Scala > spark2-shell

Spark Session

	Es el intermediario entre spark y el programa
	En el codigo al principio hay que llamar a SparkSession, en la shell se crea este objeto por defecto
	La clase SparkSession provee funciones y atributos para acceder a las funcionalidades de Spark


DataFrames - org.apache.spark.sql.DataFrame
	Objeto con datos estructurados como una tabla: coleccion inmutable de objetos de tipo Row (parecido a una tupla)
	Internamente genera un esquema 

Operaciones con DataFrames
─ Transformaciones generan un nuevo DataFrame transformado a partir de otros
─ Actions recolecta valores de un DataFrame y se lo retorna al ejecutor que tipicamente al programa Spark (driver) o a un archivo


//////////////trabajar con datos tipo parquet (HDFS)
parquet-tools schema hdfs://localhost/loudacre/basestations_import_parquet/
val baseDF = sqlContext.read.parquet("loudacre/basestations_import_parquet/")

subir ficheros de datos a HDFS: 
	$ hdfs dfs -put devices.json /loudacre/

eliminar un directorio HDFS
	$ hdfs dfs -rm -r /loudacre/weblogs/iplist/

leer un archivo
	scala> val devDF = spark.read.json("/loudacre/devices.json")

imprimir esquemade un dataFrame
	scala> devDF.printSchema

Acciones sobre un dataframe:	
	scala> devDF.show(10,false) //muestra 10 rows sin truncar

	scala> val rows = devDF.take(5) //toma 5 rows y se lo asigna a rows
	scala> rows.foreach(println)	

Transformaciones:
	scala> val sorrento10 = devDF.where("make='Sorrento'").where("model='F10L'")
	scala> devDF.where("make='Sorrento'").where("model like 'F3%'").show


DataFrames lee datos de distintas fuentes 
─ ficheros planos: CSV ─ JSON ─ Plain text
─ Binarios : Apache Parquet ─ Apache ORC
─ Tablas: Hive metastore ─ JDBC	

//carga CSV
val myDF = spark.read.format("csv").option("header","true").load("/loudacre/ejemplo.csv") //CSV

//CARGA Tabla - Hive metastore (BBDD relacional)
val accounts = spark.read.table("accounts") 

//
myDF.write.mode("append").option("path","/loudacre/mydata").saveAsTable("my_table")

myDF.write.mode("append").option("path","/loudacre/accounts_zip94913").csv("zip94913")

write.option("header","true").csv("/loudacre/accounts_zip94913")

//Schemas
scala> import org.apache.spark.sql.types._

scala> val columnsLoudacre = List(
StructField("Date Time", TimestampType),
StructField("Model name and number", StringType),
StructField("Unique device ID", StringType),
StructField("Device temperature", IntegerType),
StructField("Ambient temperature", IntegerType),
StructField("Battery available", IntegerType),
StructField("Signal strength", IntegerType),
StructField("CPU utilization", IntegerType),
StructField("RAM memory usage", IntegerType),
StructField("GPS", BooleanType),
StructField("Bluetooth status", StringType),
StructField("WiFi status", StringType),
StructField("Latitude", LongType),
StructField("Longitude", LongType))

val schemaLoudacre = StructType(columnsLoudacre)

scala> val deviceStatus = spark.read.schema(schemaLoudacre).textFile("/loudacre/devicestatus.txt")

 option("delimiter", "\t")

 val myDF = spark.option("inferSchema", "true").createDataFrame(mydata)

val txt1 = spark.read.format("txt").option("inferSchema", "true").option("delimiter", ",").option("delimiter", "|").load("/loudacre/devicestatus.txt") //CSV

val deviceStatus = spark.read.schema(schemaLoudacreStatus).option("delimiter",",").option("delimiter","|").textFile("/loudacre/devicestatus.txt")


____________________________ ejercicio pag 50____________________________________

//DataFrame basado en la tabla accounts de Hive metastore (BBDD relacional)
val accountsDF = spark.read.table("accounts")

//Hay dos sintaxis para la select de un campo:
accountsDF.select(accountsDF("first_name")).show
accountsDF.select($"first_name").show

//Generar un objeto org.apache.spark.sql.ColumnName
val fnCol = accountsDF("first_name")
val fnCol = accountsDF($"first_name")
scala> val lnCol = accountsDF.$"last_name"
lnCol: org.apache.spark.sql.ColumnName = last_name

//Se puede generar una columna con una expresion boolean
scala> val lucyCol = (fnCol === "Christopher")
lucyCol: org.apache.spark.sql.Column = (first_name = Christopher)

scala> accountsDF.select($"first_name",$"last_name",donaldCol).show
+----------+---------+---------------------+
|first_name|last_name|(first_name = Donald)|
+----------+---------+---------------------+
|    Donald|   Becton|                 true|
|     Donna|    Jones|                false|
|    Dorthy| Chalmers|                false|


//la operacion where requiere un expresion booleana: 
accountsDF.where(lucyCol).show(5)
accountsDF.where(fnCol == "Christopher").show(5)

accountsDF.select($"city", $"state",$"phone_number".substr(1,3)).show(5)

accountsDF.select($"city", $"state",$"phone_number".substr(1,3).alias("area_code")).show(5)
+-------------+-----+---------+
|         city|state|area_code|
+-------------+-----+---------+
|      Oakland|   CA|      510|
|San Francisco|   CA|      415|
|    San Mateo|   CA|      650|
|    San Mateo|   CA|      650|
|     Richmond|   CA|      510|
+-------------+-----+---------+

//metodos de agregacion: groupBy
accountsDF.groupBy("zipCode").count().show()

El groupBy tranforma a un RelationalGroupedDataset object que contiene metodos como el count (max, min, sum...):
scala> accountsDF.groupBy("zipCode")
res1: org.apache.spark.sql.RelationalGroupedDataset = org.apache.spark.sql.RelationalGroupedDataset@3847fe81

scala> accountsDF.groupBy("zipCode").count().count
res3: Long = 8613 


//join
val peopleDF = spark.read.option("header","true").csv("people-no-pcode.csv")
val pcodesDF = spark.read.option("header","true").csv("pcodes.csv")

peopleDF.join(pcodesDF, "pcode").show()//inner join 
peopleDF.join(pcodesDF, peopleDF("pcode") === pcodesDF("pcode"), "left_outer").show() //left outer join



____________________________ejercicio pag 53 Join Account Data with Cellular Towers by Zip Code


//DF de cuentas
val accountsDF = spark.read.table("accounts")

//DF de dispositivos
val devicesDF = spark.read.json("/loudacre/devices.json")					 

//DF de cuentas-dispositivos
val accountdeviceDF = spark.read.format("csv").option("inferSchema", "true").option("header", "true").load("/loudacre/accountdevice/part-00000-f3b62dad-1054-4b2e-81fd-26e54c2ae76a.csv")



val join = accountsDF.join(accountdeviceDF, accountsDF("acct_num") === accountdeviceDF("account_id"))
					 .join(devicesDF, accountdeviceDF("device_id") === devicesDF("devnum"))
					 .where(accountsDF("acct_close_dt").isNull )
					 .groupBy("device_id", "make", "model")
					 .count()
					 .withColumnRenamed("count","active_num")
					 .orderBy($"active_num".desc)

join.write.mode("overwrite").save("/loudacre/top_devices")

parquet-tools head hdfs://localhost/loudacre/top_devices


val activeAccountsDF = spark.read.table("accounts").where($"acct_close_dt".isNull )

val activeDevicesIDsDF = activeAccountsDF.join(accountdeviceDF, activeAccountsDF("acct_num") === accountdeviceDF("account_id")).select("device_id")

val activeDevices = activeDevicesIDsDF.join(devicesDF, activeDevicesIDsDF("device_id") === devicesDF("devnum")).groupBy("device_id", "make", "model")


____________________________________agregaciones ejercios

empresa.csv > agregar por dpto: dpto | total salarios | media salarios | varianza salarios | desviacion estandar salarios | media edad | max edad | min edad

stddev_samp  - desviacion estandar
variance - varianza

mejora: cifras redondeadas | columnas español

Pag 201

//import a HDFS empresa.csv
$ hdfs dfs -put empresa.csv /loudacre/empresa.csv

//df empresa.csv
scala> val empresaDF = spark.read.format("csv").option("inferSchema", "true").option("header", "true").load("/loudacre/empresa.csv")

scala> empresaDF.printSchema
root
 |-- department: string (nullable = true)
 |-- id: integer (nullable = true)
 |-- name: string (nullable = true)
 |-- age: integer (nullable = true)
 |-- salary: double (nullable = true)


val calculoDF =  empresaDF.groupBy($"department".as("departamento"))
						  .agg(round(sum($"salary"),2).as("salarioTotal"), 
						  	   round(avg($"salary"),2).as("media"), 
						  	   round(variance($"salary"),2).as("varianza"), 
						  	   round(stddev_samp($"salary"),2).as("desviacion"),  
						  	   max($"age").as("mayorEdad"), 
						  	   min($"age").as("menorEdad"))


val calculoDF =  empresaDF.groupBy($"department".as("departamento")).agg(round(sum($"salary"),2).as("salarioTotal"), round(avg($"salary"),2).as("media"), round(variance($"salary"),2).as("varianza"), round(stddev_samp($"salary"),2).as("desviacion"),  max($"age").as("mayorEdad"), min($"age").as("menorEdad"))


scala> calculoDF.show
+------------+------------+-------+---------+----------+---------+---------+
|departamento|salarioTotal|  media| varianza|desviacion|mayorEdad|menorEdad|
+------------+------------+-------+---------+----------+---------+---------+
|    finanzas|      5851.2| 1950.4|232125.21|    481.79|       41|       34|
|   marketing|     8250.45|2062.61|461987.97|     679.7|       50|       20|
+------------+------------+-------+---------+----------+---------+---------+

__________________________________________________________________
RDD Resilient Distributed Dataset


scala> val myRDD = sc.parallelize(myData)


$ hdfs dfs -put $DEVDATA/frostroad.txt /loudacre/

//carga de datos de un archivos en un RDD
scala> val myRDD = sc.textFile("/loudacre/frostroad.txt")

//collect devuelce un array con todos los elementos: 
scala> val lines = myRDD.collect


scala> val makes1RDD = sc.textFile("/loudacre/makes1.txt")
scala> val makes2RDD = sc.textFile("/loudacre/makes2.txt")

scala> val allMakesRDD = makes1RDD.union(makes2RDD)
scala> for (make <- makes1RDD.distinct.collect()) println(make)


scala> val logsRDD = sc.textFile("/loudacre/weblogs/")
scala> val jpglogsRDD = logsRDD.filter(line => line.contains(".jpg"))

scala> val ipsRDD = logsRDD.map(line => line.split(' ')(0))

scala> ipsRDD.saveAsTextFile("/loudacre/iplist")

//Use RDD transformations to create a dataset consisting of the IP address and corresponding user ID for each request for an HTML file. (Filter for files with the
//.html extension; disregard requests for other file types.) The user ID is the third field in each log file line. Save the data into a comma-separated text file in the
//directory /loudacre/userips_csv. Make sure the data is saved in the form of comma-separated strings:
scala > logsRDD.filter(line => line.contains(".html")).map(line => line.split(' ')(0).concat(",").concat(line.split(' ')(2))).saveAsTextFile("/loudacre/userips_csv")

//Load the new CSV files in /loudacre/userips_csv created above into a DataFrame, then view the data and schema.
scala> val useripsDF = spark.read.format("csv").option("inferSchema", "true").option("header", "true").load("/loudacre/userips_csv")

============BONUS1===========
Upload the devicestatus.txt file to HDFS.
val dsRDD = sc.textFile("/loudacre/devicestatus.txt")

18. Determine which delimiter to use (the 20th character—position 19—is the first use of the delimiter).
19. Filter out any records which do not parse correctly (hint: each record should have exactly 14 values).
val fDsDD = dsRDD.filter(line => line.split(line(19)).length == 14)

20. Extract the date (first field), model (second field), device ID (third field), and latitude and longitude (13th and 14th fields respectively).
21. The second field contains the device manufacturer and model name (such as Ronin
S2). Split this field by spaces to separate the manufacturer from the model (for
example, manufacturer Ronin, model S2). Keep just the manufacturer name.

val csvDsRDD = fDsDD.map(line => line.split(line(19))).map(v => v(0)+','+v(1).split(' ')(0)+','+v(2)+','+v(12)+','+v(13))

22. Save the extracted data to comma-delimited text files in the
scala> csvDsRDD.saveAsTextFile("/loudacre/devicestatus_etl")

== ALTERNATIVA convertirlo a dataFrame
import org.apache.spark.sql._
val rowsDsRDD = fDsDD.map(line => line.split(line(19))).map(v => Row(v(0),v(1).split(' ')(0),v(2),v(12),v(13)))

import org.apache.spark.sql.types._
val mySchema = StructType(Array(StructField("date", StringType),StructField("model", StringType),StructField("device ID", StringType),StructField("latitude", StringType), StructField("longitude", StringType)))

org.apache.spark.sql
val dsDF = spark.createDataFrame(rowsDsRDD,mySchema)


============BONUS2===========
Start with the ActivationModels stub script in the bonus exercise directory:

// Stub code to copy into Spark Shell

import scala.xml._

// Given a string containing XML, parse the string, and 
// return an iterator of activation XML records (Nodes) contained in the string
def getActivations(xmlstring: String): Iterator[Node] = {
    val nodes = XML.loadString(xmlstring) \\ "activation"
    nodes.toIterator
}

// Given an activation record (XML Node), return the model name
def getModel(activation: Node): String = {
   (activation \ "model").text
}

// Given an activation record (XML Node), return the account number
def getAccount(activation: Node): String = {
   (activation \ "account-number").text
}


27. Use wholeTextFiles to create an RDD from the activations dataset. The resulting RDD will consist of tuples, in which the first value is the name of the file,
and the second value is the contents of the file (XML) as a string.

val actRDD = sc.wholeTextFiles("/loudacre/activations")

28. Each XML file can contain many activation records; use flatMap to map the contents of each file to a collection of XML records by calling the provided
getActivations function. getActivations takes an XML string, parses it, and returns a collection of XML records; flatMap maps each record to a separate RDD
element.
29. Map each activation record to a string in the format account-number:model. Use the provided getAccount and getModel functions to find the values from the activation record.

scala> val accModelRDD = actRDD.map(t => getActivations(t._2)).flatMap(x => x).map(x => getAccount(x) + ":" + getModel(x))
scala> val accModelRDD = actRDD.map{case(url, xml) => getActivations(xml)}.flatMap(x => x).map(x => getAccount(x) + ":" + getModel(x))

30. Save the formatted strings to a text file in the directory /loudacre/accountmodels.
scala> accModelRDD.saveAsTextFile("/loudacre/accountmodels")

=============================Pair RDDs==============
Pares RDDs son un tipo especial de RDD
─ cada elemento es un par clave-valor (clave no unico)
─ la clave y el valor son de cualquier tipo

val usersRDD = sc.textFile("userlist.tsv").map(line => line.split('\t').map(fields => (fields(0),fields(1)))

scala> val myData = List("00001 sku010:sku933:sku022","00002 sku912:sku331","00003 sku888:sku022:sku010:sku594","00004 sku411")
scala> val myRDD = sc.parallelize(myData)

//flatMapValues
scala> val myRDD2 = myRDD.map(_.split(" ")).map(x => (x(0), x(1))).flatMapValues(skus => skus.split(':'))
//groupByKey + sortByKey . el groupBy agrupa en un objeto iterable CompactBuffer >>  (00001,CompactBuffer(sku010, sku933, sku022))
scala> val myRDD3 = myRDD2.groupByKey.sortByKey()
//mapValues
scala> val myRDD4 = myRDD3.mapValues(_.mkString(":"))
//por ultimos lo contatenos:
myRDD4.map{case(id, sku) => s"$id $sku"}.foreach(println)


=============================MAP REDUCE ==============
Map-reduce en spark usa pares RDD. 2 Fases:
> Fase Map 
─ Opera de registro  en registro y los transforma
─ Examples: map, flatMap, filter, keyBy

> Fase Recuce
─ Hace un reduce de varios records de la salida del Map
─ Examples: reduceByKey, sortByKey, mean

//Ejemplo contar palabras
val myData = List("the cat sat on the mat","the aardvark sat on the sofa")
scala> val myRDD = sc.parallelize(myData)
//Fase Map
scala> val myRDD2 = myRDD.flatMap(_.split(" ")).map(x=>(x,1))
//Fase Reduce
scala> val myRDD3 = myRDD2.reduceByKey(_+_)
//Ordenar por numero de ocurrencias:
scala> val myRDD4 = myRDD3.sortBy(_._2, ascending=false)
scala> myRDD4.take(10)
res25: Array[(String, Int)] = Array((the,4), (sat,2), (on,2), (mat,1), (aardvark,1), (cat,1), (sofa,1))


__________________________________ ejercicio 67


1) Explore Web Log Files
val wl2RDD = sc.textFile("/loudacre/weblogs/*2.log")

//Using map-reduce logic, count the number of requests from each user.
val wl2RDD1 = wl2RDD.map(_.split(" ")).map(x=>(x(2),1)).reduceByKey(_+_)

//Use countByKey to determine how many users visited the site for each frequency.
//That is, how many users visited once, twice, three times, and so on.
val freqHitMap = wl2RDD1.map{case (user, hit) => (hit, user)}.countByKey()
val freqHitMap = wl2RDD1.map(_.swap).countByKey()


//Create an RDD where the user ID is the key, and the value is the list of all the IP
//addresses that user has connected from. (IP address is the first field in each request
//line.)
val userIPs = wl2RDD.map(_.split(" ")).map(x=>(x(2),x(0))).groupByKey

//print
for (userIP <- userIPs.take(10)) {
   println(userIP._1 + ":")
   for (ip <- pair._2) println("\t"+ip)
}

2) Join Web Log Data with Account Data

val aRDD = sc.textFile("/user/hive/warehouse/accounts")

val accRDD = aRDD.map(_.split(",")).map(acc=>(acc(0),acc))
val accRDD = aRDD.map(_.split(",")).keyBy(_(0))

//Joining by Keys
val accwl2RDD = accRDD.join(wl2RDD1)

//Display the user ID, hit count, and first name (4th value) and last name (5th
//value) for the first five elements. The output should look similar to this:
accwl2RDD.map{case ( id, rowHit) => (id, rowHit._2, rowHit._1(3)+" " + rowHit._1(4))}

//BONUS///////////////////////////

//Use keyBy to create an RDD of account data with the postal code (9th field in theCSV file) as the key.
val pcRDD = aRDD.map(_.split(",")).keyBy(x=>x(8)).map{(cp,row) => (pc, row(3) + "," + row(4))}

//Create a pair RDD with postal code as the key and a list of names (Last Name,First
//Name) in that postal code as the value.
//val pcNames = pcRDD.map{case (pc,row) => (pc, row(3) + "," + row(4))}.groupByKey
val pcNames = pcRDD.mapValues(row => row(4) + "," + row(3)).groupByKey

//Sort the data by postal code
pcNames.sortByKey(false)

//////////////////// departamento_edad.csv/////////////////////////////
obtener la media de edad por deparatamento

//RDD origen de datos
val deRDD = sc.textFile("/loudacre/departamento_edad")

def isAllDigits(s: String) : Boolean = s forall Character.idDigit

val resultadoDD = deRDD.map(_.split(","))
						.filter{case(Array(dep, edad)) => isAllDigits(edad) } //filtra cabecera
						.map{case(Array(dep, edad)) => (dep, edad.toInt)}  //pair RDD dpto-edad


resultadoDD.mapValues(_-> 1) //par RDD
		   .reduceByKey{case ( (e1,c1), (e2,c2) ) => (e1+e2, c1+c2)} //se agrupa por departamento y se suma 
		   .mapValues(t => t._1/t._2.toDouble ) //media de la edad


______________________Spark SQL_______________________________________

//importar people_parquet
hdfs dfs -put $DEVSH/examples/example-data/people.parquet /loudacre

//se generar la tabla people
impala-shell -f /home/training/training_materials/devsh/examples/example-data/create_people_table.sql 


el script es:

DROP TABLE IF EXISTS people;
CREATE TABLE people (pcode STRING, first_name STRING, last_name STRING, age INT) STORED AS PARQUET;
INSERT INTO  people VALUES ('02134','Hopper','Grace',52);
INSERT INTO  people VALUES ('94020','Turing','Alan',32);
INSERT INTO  people VALUES ('94020','Lovelace','Ada',28);
INSERT INTO  people VALUES ('87501','Babbage','Charles',49);
INSERT INTO  people VALUES ('02134','Wirth','Niklaus',48);

este comando vuelva los datps en hive => /user/hive/warehouse/people

val myDF = spark.sql("SELECT * FROM people WHERE pcode = 94020")
myDF.printSchema()

spark.sql("SELECT * FROM parquet.'/loudacre/people.parquet' WHERE firstName LIKE 'A%' ").show()

//Catalog API 

//dataFrame con los tablas
spark.catalog.listTables.show

//columnas
spark.catalog.listColumns("accounts").show

Ejercicio creaar una tabla y una vista Create and query a view

//DataFrame a partir de un CSV
val accountDeviceDF = spark.read.format("csv").option("inferSchema", "true").option("header", "true").load("/loudacre/accountdevice/part-00000-f3b62dad-1054-4b2e-81fd-26e54c2ae76a.csv")

//Vista temporal TableType: TEMPORARY Su scope es la sesión
accountDeviceDF.createTempView("account_dev")

//se puede hacer consultas
spark.sql("select * from account_dev").show()

//join de dos tablas:
val nameDevDF = spark.sql("SELECT acct_num,first_name, last_name, account_device_id FROM accounts JOIN account_dev ON acct_num = account_id")

//guardalo como tabla name_dev /loudacre/name_dev => lo garda como TableType: EXTERNAL
//si no lo indico lo guarda en hive
nameDevDF.write.option("path", "/loudacre/name_dev").saveAsTable("name_dev")


//////////////////DATASET/////////////////////////

Dataset es una coleccion distribuida de objetos fuertemente tipados 
Normalmente son de tipo String o de Int o de Clases case (las clases POJO de java)
DataFrame es un subtipo de DataSet que contiene un Row 

scala> val strings = Seq("a string","another string")
scala> val stringDS = spark.createDataset(strings)

scala> case class Name(firstName: String, lastName: String)
scala> case class PcodeLatLon(pcode: String, latlon: Tuple2[Double,Double])

scala> val namesDF = spark.read.json("/loudacre/names.json")

//conversion a un DataSet de Names 
scala> val namesDS = namesDF.as[Name]

//DataSet a partir de de un RDD
val pLatLonRDD = sc.textFile("/loudacre/latlon.tsv").map(line => line.split('\t')).map(fields =>(PcodeLatLon(fields(0),(fields(1).toFloat,fields(2).toFloat))))
val pLatLonDS = spark.createDataset(pLatLonRDD)


---- Ejercicio Explore Datasets using web log data (pag 74)

//case class AccountIP
scala> case class AccountIP (id: Int, ip: String)
defined class AccountIP

//RDD con cuentas-IP
scala> val accountIPRDD = sc.textFile("/loudacre/weblogs").map(line => line.split(' ')).map(fields => new AccountIP(fields(2).toInt,fields(0)))

//Dataset a partir de un RDD
val accountIPDS = spark.createDataset(accountIPRDD)

//where que es un alias de filter
accountIPDS.distinct.where("id==59938").show
accountIPDS.distinct.filter("id==59938").show

//group y count
val accountIPCountDS = accountIPDS.groupBy("id","ip").count()

----------------BONUS
// case class user IDs y IP 
case class AccountIP (id: Int, ip: String)

//RDD
val accountIPRDD=sc.textFile("/loudacre/weblogs").map(line => line.split(' ')).map(fields => new AccountIP(fields(2).toInt,fields(0)))

val accountIPDS = spark.createDataset(accountIPRDD)
accountIPDS.createOrReplaceTempView("account_ip")
val queryDF = spark.sql("SELECT DISTINCT *  FROM account_ip WHERE id < 200")

queryDF.printSchema
queryDF.show



==========CAPITULO 13  Writing, Configuring, and Running Apache Spark Applications



spark2-submit --class NameList namelist-1.0.jar people.json namelist/

Application Deployment Mode
spark2-submit --master yarn --deploy-mode cluster NameList people.json namelist/


spark2-submit --class intro.df.AccountsByState target/accounts-by-state-1.0.jar CA



============================================== SPARK STREAMING

Flujo de datos: generación continua con RDD

DStream—RDDs 
- Un DStream (Discretized Stream) divide el flijp de datros a data stream into batches of n
seconds
▪ Processes each batch in Spark as an RDD
▪ Returns results of RDD operations in batches


============================================== DATA SOURCES

Advanced data sources
─ Apache Kafka
─ Apache Flume
─ Twitter
─ ZeroMQ
─ Amazon Kinesis

Data Sources estan basado en receivers
─ Los datos se reciben en un worker nodo
─ El Receiver distribuye los RDD al cluster en particiones


Apache Flume es un sistema de colecciones de datos en tiempo real escalable, extensible

Un evento Flume incluye metadata en un header (metadata) and a payload containing event
data
─ Example event: a line of web server log output
▪ Flume agents are configured with a source and a sink
─ A source receives events from an external system
─ A sink sends events to their destination
▪ Types of sources include server output, Kafka messages, HTTP requests, and files
▪ Types of sinks include Spark Streaming applications, HDFS directories, Apache
Avro, and Kafka

─ Now widely used for collection of any streaming event data
─ Supports aggregating data from many sources into HDFS


Un DStream puede recibir datos de Flume. Un receiver funciona 
─ Push-based
─ Pull-based
▪ Push-based
─ One Spark executor must run a network receiver on a specified host
─ Configure Flume with an Avro sink to send to that receiver on that host

▪ Pull-based
─ Uses a custom Flume sink in spark.streaming.flume package
─ Strong reliability and fault tolerance guarantees


============================================
KAFKA 

Apache Kafka es un servicio de mensajeria rapido, escalable, distribuido (tolerante a fallos)
─ Los mensajes se persiste en disco 

Concepto
- Productor programas que publican mensajes en el topic
- Mensaje
- Brokers(En cluster) reciben almacenan y distribuyen mensajes.
- Topic receptaculo de mensajes. Cola de mensajeria. Pueden estar particionados
- Consumidor reciben los mensajes del Tópic (Se suscriben al topic)

▪ A message is a single data record passed by Kafka
▪ One or more brokers in a cluster receive, store, and distribute messages

Crear un Kafka topic (cola de mensajeria)
$ kafka-topics --create --zookeeper master-1:2181 --replication-factor 1 --partitions 2 --topic weblogs

listar topics
$ kafka-topics --list --zookeeper master-1:2181

detalle del topic
$ kafka-topics --describe weblogs --zookeeper master-1:2181

|	Topic:weblogs	PartitionCount:2	ReplicationFactor:1	Configs:
|		Topic: weblogs	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
|		Topic: weblogs	Partition: 1	Leader: 0	Replicas: 0	Isr: 0

arrancar productor kakfa
kafka-console-producer --broker-list master-1:2181 --topic weblogs

arrancar consumidor kakfa
kafka-console-consumer --zookeeper master-1:2181 --topic weblogs --from-beginning

========================= FLUME ejercicio 111

hdfs dfs -mkdir -p /loudacre/weblogs_flume

//Start the Flume agent using the configuration you just reviewed
$ flume-ng agent --conf /etc/flume-ng/conf --conf-file $DEVSH/exercises/flume/spooldir.conf --name agent1 -Dflume.root.logger=INFO,console


copy-move-weblogs.sh /home/training/training_materials/devsh/data/flume/weblogs_spooldir


# spooldir.conf: A Spooling Directory Source

# Name the components on this agent
agent1.sources = shakespeare
agent1.sinks = hdfs-sink
agent1.channels = channel

# Describe/configure the source
agent1.sources.shakespeare.type = exec
agent1.sources.shakespeare.command = tail -f /tmp/flume
agent1.sources.shakespeare.channels = memory-channel

# Describe the sink
agent1.sinks.hdfs-sink.type = hdfs
agent1.sinks.hdfs-sink.hdfs.path = /home/training/flume/
agent1.sinks.hdfs-sink.channel = channel
agent1.sinks.hdfs-sink.hdfs.rollInterval = 0
agent1.sinks.hdfs-sink.hdfs.rollSize = 524288
agent1.sinks.hdfs-sink.hdfs.rollCount = 0
agent1.sinks.hdfs-sink.hdfs.fileType = DataStream

a1.sinks.hdfs-sink.type = hdfs
a1.sinks.hdfs-sink.channel = channel
a1.sinks.hdfs-sink.hdfs.path = /user/training/flume/%y-%m-%d
a1.sinks.hdfs-sink.hdfs.filePrefix = flume-%y-%m-%d
a1.sinks.hdfs-sink.hdfs.rollSize = 1048576
a1.sinks.hdfs-sink.hdfs.rollCount = 100
a1.sinks.hdfs-sink.hdfs.rollInterval = 120
a1.sinks.hdfs-sink.hdfs.fileType = DataStream
a1.sinks.hdfs-sink.hdfs.idleTimeout = 10
a1.sinks.hdfs-sink.hdfs.useLocalTimeStamp = true


# Use a channel which buffers events in memory
agent1.channels.channel.type = FILE
agent1.channels.channel.capacity = 100000
agent1.channels.channel.transactionCapacity = 1000



Hay que crear un agente que tenga como source exec con la salida del
comando tail -F /tmp/flume y como sink hdfs con las siguientes condiciones:
1. Los ficheros que se crean en hdfs deben tener un tamaño aproximado de
524.288 bytes. No queremos que haya ficheros con un tamaño inferior a
524.288 excepto cuando se produza la situación descrita en el punto 2.

2. Después de 30 segundos en los que no haya input el fichero con extensión
tmp se debe cerrar.

3. En el sink, aparte de otros, hay que especificar los siguientes parámetros:
a. hdfs.rollCount
b. hdfs.rollInterval
c. hdfs.rollSize
d. hdfs.idleTimeout

4. El directorio de destino debe ser /user/training/flume.
El script generar_datos.sh coge el fichero shakespeare.txt y cada milésima de
segundo añade una línea al fichero /tmp/flume.
generar_datos.sh
while read line
do
echo "$line"
sleep 0.001
done < shakespeare.txt >> /tmp/flume

Los pasos a seguir a la hora de ejecutar el ejercicio son:
1. Ejecutar > /tmp/flume para crear el fichero vacío /tmp/flume
2. Arrancar el agente
3. Ejecutar ./generar_datos.sh para mandar línea al fichero /tmp/flume


===================================Flafka


#consumidor
kafka-console-consumer   --zookeeper master-1:2181   --topic weblogs

#agente
flume-ng agent --conf /etc/flume-ng/conf --conf-file $DEVSH/exercises/flafka/spooldir_kafka.conf --name agent1 -Dflume.root.logger=INFO,console

[training@localhost ~]$ $DEVSH/scripts/copy-move-weblogs.sh /home/training/training_materials/devsh/data/flume/weblogs_spooldir



# spooldir_kafka.conf: A Spooling Directory Source with a Kafka Sink

# Name the components on this agent
agent1.sources = webserver-log-source
agent1.sinks = kafka-sink
agent1.channels = memory-channel

# Configure the source
agent1.sources.webserver-log-source.type = spooldir
agent1.sources.webserver-log-source.spoolDir = /flume/weblogs_spooldir
agent1.sources.webserver-log-source.channels = memory-channel

# Configure the sink
agent1.sinks.kafka-sink.type = org.apache.flume.sink.kafka.KafkaSink
agent1.sinks.kafka-sink.topic = weblogs
agent1.sinks.kafka-sink.brokerList = master-1:9092
agent1.sinks.kafka-sink.batchSize = 20
agent1.sinks.kafka-sink.channel = memory-channel


# Use a channel which buffers events in memory
agent1.channels.memory-channel.type = memory
agent1.channels.memory-channel.capacity = 100000
agent1.channels.memory-channel.transactionCapacity = 1000
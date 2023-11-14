# Debezium-2.3

Debezium 2.3 + Azure EventHub + Azure BlobStorage 

Problem: Connectors were continuously failing with the below error message
io.debezium.DebeziumException: The db history topic or its content is fully or partially missing. Please check database history topic configuration and re-execute the snapshot.

Root cause: Schema history must have infinite retention in order to make sure that upon restart the connector's table structure metadata can be properly rebuilt and that all tables you want to capture are known to the connector based on the number of columns and primary keys at the point in time that the connector is reading the transaction logs. If not, a re-snapshot will be required everytime the connector restarts. By Chris Cranford (Red Hat)

Resolution: As Azure event hub does not support creation of topics with infinite retention and no compaction, Debezium itself providing mechanism to store schema history in Azure Blob Storage by adding Azure Blob storage extension.

Steps for creating debezium/connect image with Azure Blob Storage:
1.	Download the following jar files from Maven repository (https://mvnrepository.com/ ) individually or from https://github.com/hrishikeshtt/Debezium/tree/main/2.3/debezium-storage/debezium-storage-azure-blob

      1. azure-core-1.36.0.jar
      2. azure-core-http-netty-1.13.0.jar
      3. azure-storage-blob-12.21.0.jar
      4. azure-storage-common-12.20.0.jar
      5. azure-storage-internal-avro-12.6.0.jar
      6. debezium-storage-azure-blob-2.3.0.Final.jar
      7. jackson-dataformat-xml-2.13.4.jar
      8. netty-buffer-4.1.86.Final.jar
      9. netty-codec-4.1.86.Final.jar
      10. netty-codec-dns-4.1.86.Final.jar
      11. netty-codec-http-4.1.86.Final.jar
      12. netty-codec-http2-4.1.86.Final.jar
      13. netty-codec-socks-4.1.86.Final.jar
      14. netty-common-4.1.86.Final.jar
      15. netty-handler-4.1.86.Final.jar
      16. netty-handler-proxy-4.1.86.Final.jar
      17. netty-resolver-4.1.86.Final.jar
      18. netty-resolver-dns-4.1.86.Final.jar
      19. netty-resolver-dns-classes-macos-4.1.86.Final.jar
      20. netty-resolver-dns-native-macos-4.1.86.Final-osx-x86_64.jar
      21. netty-tcnative-boringssl-static-2.0.56.Final-linux-aarch_64.jar
      22. netty-tcnative-boringssl-static-2.0.56.Final-linux-x86_64.jar
      23. netty-tcnative-boringssl-static-2.0.56.Final-osx-aarch_64.jar
      24. netty-tcnative-boringssl-static-2.0.56.Final-osx-x86_64.jar
      25. netty-tcnative-boringssl-static-2.0.56.Final-windows-x86_64.jar
      26. netty-tcnative-boringssl-static-2.0.56.Final.jar
      27. netty-tcnative-classes-2.0.56.Final.jar
      28. netty-transport-4.1.86.Final.jar
      29. netty-transport-classes-epoll-4.1.86.Final.jar
      30. netty-transport-classes-kqueue-4.1.86.Final.jar
      31. netty-transport-native-epoll-4.1.87.Final-linux-x86_64.jar
      32. netty-transport-native-kqueue-4.1.87.Final-osx-x86_64.jar
      33. netty-transport-native-unix-common-4.1.87.Final.jar
      34. reactive-streams-1.0.4.jar
      35. reactor-core-3.4.26.jar
      36. reactor-netty-core-1.0.27.jar
      37. reactor-netty-http-1.0.27.jar
      38. stax2-api-4.2.1.jar.
      39. woodstox-core-6.4.0.jar
      40. jackson-datatype-jsr310-2.13.4.jar

https://github.com/hrishikeshtt/Debezium/tree/main/2.3/debezium-storage/debezium-storage-azure-blob 

2.	Create a temporary container by uing debezium/connect image
docker create --name temp_container debezium/connect:2.3.0.Final

3.	Copy and paste each and every jar files downloaded to the kafka/connect/debezium-connector-sqlserver location of the temp_container like below

docker cp C:/Users/1234/Desktop/debezium-storage-azure-blob/azure-storage-blob-12.21.0.jar temp_container:kafka/connect/debezium-connector-sqlserver

Repeat the command for all remaining jar files by changing the jar file name. Please make sure to specify your downloaded path.

4.	Follow the below steps by committing the changes for temp_container and create a new image
a.	docker commit temp_container 
b.	docker images -a (This command list all the images with ids, select the latest image id)
c.	docker tag <latest image id> debeziumconnect-azureblobstorage:2.3
5.	Verify all the jars are added to the new image debeziumconnect-azureblobstorage:2.3
a.	Run the below commands.
docker run -t -i  debeziumconnect-azureblobstorage:2.3 /bin/bash
cd connect/debezium-connector-sqlserver
ls -la
which should list all the jars file which were added like below.
 

6.	Add the following properties with the connector configuration
schema.history.internal=io.debezium.storage.azure.blob.history.AzureBlobSchemaHistory
schema.history.internal.azure.storage.account.connectionstring=...
schema.history.internal.azure.storage.account.container.name=...
schema.history.internal.azure.storage.blob.name=...

7.	Please create the connector
```
$config = '{    \"name\": \"$(sqlconnectorname)\",     \"config\": {        \"connector.class\": \"io.debezium.connector.sqlserver.SqlServerConnector\",         \"database.hostname\": \"$(hostname)\",         \"database.port\": \"$(databaseport)\",         \"database.user\": \"$(user)\",         \"database.password\": \"$(password)\",         \"database.names\": \"$(dbname)\",         \"topic.prefix\": \"$(servername)\",         \"table.include.list\": \"$(tables)\",         \"schema.history.internal\":\"io.debezium.storage.azure.blob.history.AzureBlobSchemaHistory\",\"schema.history.internal.azure.storage.account.connectionstring\":\"$(schemahistoryazurestorageconnectionstring)\",\"schema.history.internal.azure.storage.account.container.name\":\"$(schemahistorycontainername)\",\"schema.history.internal.azure.storage.blob.name\":\"$(schemahistoryazurestorageblobname)\",\"transforms\": \"Reroute\",\"transforms.Reroute.type\": \"io.debezium.transforms.ByLogicalTableRouter\",\"transforms.Reroute.topic.regex\": \"(.*)([^0-9]).*\",\"transforms.Reroute.topic.replacement\": \"$(cdcmasterhub)\",\"retries\":\"2147483647\",\"errors.retry.timeout\":\"-1\",\"errors.retry.delay.max.ms\":\"300000\",\"errors.log.enable\":\"true\",\"errors.log.include.messages\":\"true\",\"errors.tolerance\":\"all\",\"database.sqlserver.agent.status.query\":\"SELECT    func_is_sql_server_agent_running()\",\"plugin.path\":\"/kafka/connect\",\"decimal.handling.mode\":\"double\",\"driver.encrypt\": \"true\",\"driver.trustServerCertificate\": \"true\",\"io.debezium.jdbc.JdbcConnection\":\"TRACE\",\"io.debezium.connector.sqlserver.SqlServerConnection\":\"TRACE\",\"snapshot.mode\": \"schema_only\",\"topic.creation.default.replication.factor\": \"$(TopicReplicationFactor)\",  \"topic.creation.default.partitions\": \"$(PartitionCount)\",\"topic.creation.default.cleanup.policy\": \"delete\"    }}'


kubectl exec -n $(namespace) -i $(podname) -- bash -c "curl http://$(podip):8083/connectors -X POST -H 'Accept:application/json' -H 'Content-Type:application/json' -d '$config'"
```


Connector should be created in worker as well as schema history will be saved as blob in the specified path in Azure Blob Storage.

Now connectors will run without the schema history error.

Problem: The driver could not establish a secure connection to SQL Server by using Se
cure Sockets Layer (SSL) encryption. Error: \"Certificates do not conform to algorithm constraints\".

How to get the root cause: Add the below logging properties with the connector configuration
"io.debezium.jdbc.JdbcConnection":"TRACE",
"io.debezium.connector.sqlserver.SqlServerConnection":"TRACE"
- name: CONNECT_LOG4J_LOGGERS_IO_DEBEZIUM_JDBC_JDBCCONNECTION
  value: "TRACE"
 - name: CONNECT_LOG4J_LOGGERS_IO_DEBEZIUM_CONNECTOR_SQLSERVER_SQLSERVERCONNECTION
value: "TRACE"

Error:  Caused by: java.security.cert.CertPathValidatorException: Algorithm constraints check failed on signature algorithm: SHA1withRSA

Root cause: The issue is because SHA1withRSA algorithm is deprecated with JDK11 because its possible to spoof and do other insecure things with such an algorithm. Using JDK11 will not allow this algorithm to be used. By Chris Cranford (Red Hat)

Solution: Re-enable the SHA1withRSA algorithm in your java.security file

Steps:
1.	Remove the temp container if it is present.
Docker rm temp_container
2.	Create temp container
docker create --name temp_container debeziumconnect-azureblobstorage:2.3.0
3.	Copy and paste the java.config and java.secuirty to your local from debezium/connect image or from debeziumconnect-azureblobstorage:2.3 image
Use the find –name command to find the location of java.security and java.config files from the root directory of container

Use the readlink -f ./crypto-policies/back-ends/java.config command to get the associated text file for java.config file

docker cp your_container_name:/etc/java/java-11-openjdk/java-11-openjdk-11.0.19.0.7-1.fc37.x86_64/conf/security/java.security /Users/12345/Documents/java.security

docker cp your_container_name /usr/share/crypto-policies/DEFAULT/java.txt /Users/12345/Documents/java.txt

4.	Remove the SHA1 and RSA from jdk.tls.disabledAlgorithms, jdk.certpath.disabledAlgorithms properties from java.securiyt and java.txt files
5.	Copy and paste back these modified files to the container.

docker cp C:/Users/12345/Documents/java.security temp_container:/etc/java/java-11-openjdk/java-11-openjdk-11.0.19.0.7-1.fc37.x86_64/conf/security/java.security

docker cp C:/Users/12345/Documents/java.txt temp_container:/usr/share/crypto-policies/DEFAULT/java.txt

6.	Commit the changes and create a new image.

i.	docker commit temp_container 
ii.	docker images -a (This command list all the images with ids, select the latest image id)
iii.	docker tag <latest image id> debeziumconnect-azureblobstorage-legacy:2.3
7.	Create the worker based on the new image, add the connector and see changes are captured.


Push the new image to your image repository
1.	docker login your_docker_hubname -u your_username -p your_password
2.	docker push debeziumconnect-azureblobstorage:2.3

References:
1.	https://debezium.zulipchat.com/#narrow/stream/302529-community-general/topic/.E2.9C.94.20error.20while.20creating.20connector 
2.	https://nielsberglund.com/post/2022-01-14-how-to-stream-data-to-event-hubs-from-databases-using-kafka-connect--debezium-in-docker---ii/ 
3.	https://groups.google.com/g/debezium/c/YkyXZxJGTKk 
4.	https://issues.redhat.com/browse/DBZ-6180 
5.	https://stackoverflow.com/questions/56670437/add-a-file-in-a-docker-image 
6.	https://github.com/debezium/debezium/tree/main/debezium-storage/debezium-storage-azure-blob 
7.	https://github.com/debezium/container-images 
8.	https://repo1.maven.org/maven2/ 
9.	https://sqlzealots.com/2019/04/07/cdc-jobs-in-sql-server-capture-and-cleanup/ 
10.	https://kafka.apache.org/documentation/ 
11.	https://mvnrepository.com/artifact/io.debezium/debezium-storage-azure-blob 













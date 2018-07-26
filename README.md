# Spring/Hibernate App on Azure AppService (Tomcat) using JNDI to access SQL Server on Azure  

## Spring Hibernate JNDI Example

This Spring example is based on tutorial [Hibernate, JPA & Spring MVC - Part 2] (https://www.alexecollins.com/tutorial-hibernate-jpa-spring-mvc-part-2/) by Alex Collins

Spring application uses hibernate to connect to SQL Server. Hibernate is using container managed JNDI Data Source.

There are few options to configure JNDI for the web application running undet Tomcat (Tomcat JNDI)[https://www.journaldev.com/2513/tomcat-datasource-jndi-example-java] :

- Application context.xml - located in app `META-INF/context.xml` - define Resource element in the context file and container will take care of loading and configuring it.
- Server server.xml - Global, shared by applications, defined in `tomcat/conf/server.xml`
- server.xml and context.xml - defining Datasource globally and including ResourceLink in application context.xml

When deploying to Java application on Azure AppService, you can customize  out of the box managed Tomcat server.xml, but is not recommended as it will create snowflake deployement. That's why we will define JNDI Datasource on the **Application level**

# Create Azure AppService and database

In Azure Portal create `Web App + SQL` and configure settings for the App to use Tomcat
![Azure App Service config](https://github.com/lenisha/spring-jndi-appservice/raw/master/img/AppService.PNG  "App Service Config")

Copy the Connection String from Azure SQL database
![Azure SQL Connection](https://github.com/lenisha/spring-jndi-appservice/raw/master/img/ConnString.PNG "Azure  SQL Server")

Add the Connection String from Azure SQL database to **App Service / Application Settings**   settings
** DO NOT INCLUDE USERNAME/PASSWORD **
![Azure SQL Connection](https://github.com/lenisha/spring-jndi-appservice/raw/master/img/ConnectionString.PNG "Azure App Service Settings")

DB connection url for Azure SQL is usually in thins format `jdbc:sqlserver://jnditestsrv.database.windows.net:1433;database=jnditestsql;encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30;`

Adding Setting `JAVA_OPTS` with value `-D<connection name>=<jdbc url>`  would create environment variable `connection name` available to the Java application.
 In our example above - `SQLDB_URL`


# Managed Service Identity
Add MSI to AppService and grant it permissions to SQL database

```
az webapp identity assign --resource-group <resource_group> --name <app_service_name>
az ad sp show --id <id_created_above>

az sql server ad-admin create --resource-group <resource_group> --server-name <sql_server_name>  --display-name <admin_account_name> --object-id <id_created_above>
```

Example:

```
az webapp identity assign --resource-group jnditest --name testjndi
az ad sp show --id 82cc5f96-226a-4721-902c-7451cc68fd80

az sql server ad-admin create --resource-group jnditest --server-name jnditestsrv  --display-name admin-msi --object-id 82cc5f96-226a-4721-902c-7451cc68fd80
```

# Define DataSource


To define JNDI Datasource for Tomact Application, add file `META-INF/context.xml` to the application.
In this example it's added to `main/webapp/META-INF/context.xml` anc contains the following datasource definition

```
<Context>
    <Resource auth="Container" 
	    driverClassName="com.microsoft.sqlserver.jdbc.SQLServerDriver"
	    maxActive="8" maxIdle="4"
	    name="jdbc/tutorialDS" type="javax.sql.DataSource"
		url="${SQLDB_URL}"
	    factory="com.microsoft.sqlserver.msi.MsiDataSourceFactory" />
</Context>
```
where
- `msiEnable` flag tells Factory to use MSI identity to establish connection to Datasource
- `factory` overrides default Tomcat `BasicDataSourceFactory` and is MSI aware (included in `msi-mssql-jdbc` library)
- `url` points to url, in the example above provided by environment variable set by `JAVA_OPTS`

## Use SQL Server Hibernate Dialect

in `resources\META-INF\persistence.xml`

```
<property name="hibernate.dialect" value="org.hibernate.dialect.SQLServerDialect" />
```

## Update Application dependencies
JDBC driver for SQL server `sqljdbc42.jar` installed in Tomcat in Azure App Service by default is older version that does not support token authentication,
Include newer version that supports token based Authentication along with the app.

```

        <dependency>
            <groupId>com.microsoft.sqlserver.msi</groupId>
            <artifactId>msi-mssql-jdbc</artifactId>
	        <version>1.0</version>
        </dependency>

        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>2.8.0</version>
        </dependency>

        <dependency>
            <groupId>com.microsoft.sqlserver</groupId>
            <artifactId>mssql-jdbc</artifactId>
            <version>6.4.0.jre7</version>
        </dependency>
```

The `msi-mssql-jdbc` library (is not on Maven central yet) could be downloaded from: [msi-mssql-jdbc release](https://github.com/lenisha/msi-mssql-jdbc/releases/tag/v1.0)
or built and uploaded to Maven based repo.

In this example we load it from `lib\msi-mssql-jdbc.jar` into local maven repository using Maven plaugin

```
 mvn clean initialize
```

see `pom.xml` on how to use `maven-install-plugin

## To test locally:
`mvn clean  package`
and copy resulting war file from target directory to Tomcat webapps directory

Navigate to `localhost:8080/spring-jndi-appservice/create-user.html`

## To debug locally
set env variables
```
set JAVA_OPTS=-DSQLDB_URL=jdbc:sqlserver://server.database.windows.net:1433;database=database;msiEnable=true;encrypt=true;trustServerCerti
ficate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30;

set JPDA_SUSPEND=y

catalina.bat jpda start
```


## Deploy on Azure AppService

To test on Azure AppService deploy using Azure deployment options, the simplest is to use FTP:

### FTP
  - Get the FTP hostname and credentials from the App Service blade
  - Upload war file to `d:\home\site\wwwroot\webapps`  
  - Upload web.config file to `d:\home\site\wwwroot`
  - Restart App Service
  - Navigate to `https://appname.azurewebsites.net/tutorial-hibernate-jpa/create-user.html`

### Maven plugin
- setup authentication in .m2/settings.xml as described in https://docs.microsoft.com/en-us/java/azure/spring-framework/deploy-spring-boot-java-app-with-maven-plugin

- Add plugin definition in pom.xml
```
      <plugin>
            <groupId>com.microsoft.azure</groupId>
            <artifactId>azure-webapp-maven-plugin</artifactId>
            <version>0.2.0</version>
            <configuration>
                <authentication>
                    <serverId>azure-auth</serverId>
                </authentication>
                 <!-- Web App information -->
               <resourceGroup>testjavajndi</resourceGroup>
               <appName>testjavajndi</appName>
               <!-- <region> and <pricingTier> are optional. They will be used to create new Web App if the specified Web App doesn't exist -->
               <region>canadacentral</region>
               <pricingTier>S1</pricingTier>
               
               <!-- Java Runtime Stack for Web App on Windows-->
               <javaVersion>1.7.0_51</javaVersion>
               <javaWebContainer>tomcat 7.0.50</javaWebContainer>
               
               <!-- WAR deployment -->
              <deploymentType>war</deploymentType>

               <!-- Specify the war file location, optional if the war file location is: ${project.build.directory}/${project.build.finalName}.war -->
               <warFile>${project.build.directory}/${project.build.finalName}.war </warFile>

               <!-- Specify context path, optional if you want to deploy to ROOT -->
               <path>/tutorial-hibernate-jpa</path>

            </configuration>
        </plugin>
	
```
- run `mvn azure-webapp:deploy`

Example output:
```
[INFO] --- azure-webapp-maven-plugin:0.2.0:deploy (default-cli) @ tutorial-hibernate-jpa ---
AI: INFO 25-03-2018 19:31, 1: Configuration file has been successfully found as resource
AI: INFO 25-03-2018 19:31, 1: Configuration file has been successfully found as resource
[INFO] Start deploying to Web App testjavajndi...
[INFO] Authenticate with ServerId: azure-auth
[INFO] [Correlation ID: 462a8a7f-c2ec-40c8-a43b-80be705225b8] Instance discovery was successful
[INFO] Updating target Web App...
[INFO] Successfully updated Web App.
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 1 resource to C:\projects\tutorial-hibernate-jpa\target\azure-webapps\testjavajndi\webapps
[INFO] Starting uploading files to FTP server: waws-prod-yt1-005.ftp.azurewebsites.windows.net
[INFO] Starting uploading directory: C:\projects\tutorial-hibernate-jpa\target\azure-webapps\testjavajndi --> /site/wwwroot
[INFO] [DIR] C:\projects\tutorial-hibernate-jpa\target\azure-webapps\testjavajndi --> /site/wwwroot
[INFO] ..[DIR] C:\projects\tutorial-hibernate-jpa\target\azure-webapps\testjavajndi\webapps --> /site/wwwroot/webapps
[INFO] ....[FILE] C:\projects\tutorial-hibernate-jpa\target\azure-webapps\testjavajndi\webapps\tutorial-hibernate-jpa.war --> /site/wwwroot/webapps
[INFO] ...........Reply Message : 226 Transfer complete.

[INFO] Finished uploading directory: C:\projects\tutorial-hibernate-jpa\target\azure-webapps\testjavajndi --> /site/wwwroot
[INFO] Successfully uploaded files to FTP server: waws-prod-yt1-005.ftp.azurewebsites.windows.net
[INFO] Successfully deployed Web App at https://testjavajndi.azurewebsites.net
``` 


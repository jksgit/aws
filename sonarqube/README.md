This article will briefly describe how to install SonarQube as Docker container on Amazon EC2 and integrate it with Jenkins.

### Create database instance and user
Go to RDS > Parameter Groups  
Create a new Parameter Group with the following parameter:  
```
client_encoding = UTF8 (default for PostgreSQL will be UTF8)
```
We need to create a new RDS database for SonarQube (you may use an existing instance)  
 1. Go to RDS Instances
 2. Launch a new DB instance
 3. Select PostgreSQL
 4. Define your master username and password

Then create the database and add a user:
```
CREATE DATABASE sonar CHARACTER SET utf8 COLLATE utf8_general_ci;
CREATE USER sonar WITH ENCRYPTED PASSWORD 'sonar';
GRANT ALL PRIVILEGES ON DATABASE sonar TO sonar;

```

### Run SonarQube as EC2 Container
Let's create a new container for SonarQube and add it to ELB load balancer to make it easily accessible  
 1. Go to EC2 Container Service
 2. Create a new cluster
 3. On cluster details page, switch to "Task Definitions"
 4. Create new Task Definition:  
 - Give it a name (eg. SonarQube)  
 - Set task tole to None  
 - Select Bridge Network mode  
 5. Add a container
 Container name: SonarQube  
 Image: sonar:latest  
 You may want to add memory limit. Minimum 1024MB is recommended.  
 Ports: 9000 -> 9000  

 ENVIRONMENT
    Add env varialbes:  
    SONARQUBE_JDBC_USERNAME = sonar  
    SONARQUBE_JDBC_PASSWORD = sonar  
    SONARQUBE_JDBC_URL = jdbc:mysql://YOUR_DB.rds.amazonaws.com:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true  

Once the task is defined, it's time to create the ELB:
 1. Got to EC2 Dashboard -> Load Balancers
 2. Create a new Load Balancer
 3. Select the "Classic Load Balancer"
 4. Give it a name (eg. SonarQubeELB)
 5. Select a proper VPC subnet (should be the same as your ECS Cluster)
 6. Then load balance the HTTP protocol port 80 to Instance Port: 9000
 7. Add the right availability zone
 8. Define Health Check HTTP:9000 (you may want to test HTTP:9000/images/favicon.ico)

Finally, we have to create a new Service based on out task defined earlier and add it to the ELB we just created.
 1. Go back to EC2 Container Service
 2. Select your cluster
 3. Create a new Service
 4. Select SonarQube as task definition, give your service a name
 5. Number of tasks will be set to 1
 6. Then click on Configure ELB
 7. Select the Classic Load Balancer
 8. Select the SonarQubeELB you created
 9. Save

Once the service will start the task and become ACTIVE , you will see your SolarQube up and running.  
It can be accessible by DNS name of the load balancer.  

### Integration with Jenkins
Connect using SSH to your Jenkins server.  
Edit the ```settings.xml file```, located in $MAVEN_HOME/conf or ~/.m2, to set the plugin prefix and the SonarQube server URL.  
Example:  
```
<settings>
    <pluginGroups>
        <pluginGroup>org.sonarsource.scanner.maven</pluginGroup>
    </pluginGroups>
    <profiles>
        <profile>
            <id>sonar</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <!-- Optional URL to server. Default value is http://localhost:9000 -->
                <sonar.host.url>
                  http://myserver:9000
                </sonar.host.url>
            </properties>
        </profile>
     </profiles>
</settings>
```

Login to your Jenkins with administrative user:
 1. Go to Manage Jenkins
 2. Configure System
 3. In "SonarQube Servers" section
 4. add the SolarQube Server URL and an auth token to access it.

How to generate a token: http://docs.sonarqube.org/display/SONAR/User+Token  

### Analyzing a Maven Project
Analyzing a Maven project consists of running a Maven goal: ```sonar:sonar``` in the directory where the pom.xml file sits.
```
mvn clean verify sonar:sonar
# In some situation you may want to run sonar:sonar goal as a dedicated step. Be sure to use install as first step for multi-module projects
mvn clean install
mvn sonar:sonar
```

### Analyzing a Project using SolarQube Scanner Plugin
Add a Post Build step: Execute SonarQube Scanner.  
You will be required to provide some parameters in Analysis Properties:
```
sonar.login=admin
sonar.password=admin
sonar.projectKey=project:key
sonar.projectName=Project Name goes here
sonar.projectVersion=1.0
sonar.sources=$WORKSPACE
```

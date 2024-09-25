# Process Instance Migration example

## Description

Given a project with a BPMN diagram, we'll generate two versions:

* v1: BPMN diagram with two User Tasks, "Validate preliminar data" and "Validate final data".
* v2: BPMN diagram with a single User Task, "Validate final data". "Validate preliminar data" is removed from the BPMN flow in this version.

## Deployment
Using the docker compose provided in ./docker-compose/docker-compose.yaml, we'll start the Kogito process service in both v1 and v2. This way, we'll simulate an scenario where two different versions of a process are deployed, and we'll plan a migration of the active process instances from v1 to v2.


### Prerequisites
* git
* JDK 17
* Maven 3.9.6+
* Docker
* Docker compose

### Steps to build the reproducer

1. ``git clone -b 1.0.x https://github.com/jaimeizuz/kogito-bpmn-process-instance-migration.git``
   
2. ``mvn clean package -Pcontainer``
   
3. ``git checkout main https://github.com/jaimeizuz/kogito-bpmn-process-instance-migration.git``
   
4. ``mvn clean package -Pcontainer``
   
5.  In docker-compose folder, ``docker-compose up -d``

6. Start a new process instance in version 1 and save the resulting processInstanceId: \
``curl --location 'http://localhost:8081/SampleProcess'`` \\\
``--header 'Accept: application/json'`` \\\
``--header 'Content-Type: application/json'`` \\\
``--data '{}'``

7. Migrate process instance from version 1 to version 2: \
``curl --location 'http://localhost:8082/SampleProcess'`` \\\
``--header 'Accept: application/json'`` \\\
``--header 'Content-Type: application/json'`` \\\
``--data '{}'``



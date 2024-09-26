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

6. List the active user tasks in the first deployment: \
``curl --location 'http://localhost:8081/graphql?query=%7BUserTaskInstances(where%3A%7BprocessInstanceId%3A%7Bequal%3A%22d00e3fb7-d072-44fb-b3d4-4b2611d0b687%22%7D%7D%2CorderBy%3A%7BlastUpdate%3ADESC%7D)%7Bid%2Cname%2Cstate%2CactualOwner%2Cdescription%2ClastUpdate%2Cpriority%2CprocessId%2CprocessInstanceId%2Cendpoint%2Cinputs%2Coutputs%7D%7D'`` \\\
``--header 'Accept: application/json'`` \\\
``--header 'Content-Type: application/json'`` \\\
``--data '{"waitTime":"PT120S"}'``

7. Migrate process instances from version 1 to version 2: \
``curl --location 'http://localhost:8081/management/processes/SampleProcess/migrate' `` \\\
``--header 'Accept: application/json'`` \\\
``--header 'Content-Type: application/json'`` \\\
``--data '{`` \\\
``  "targetProcessId": "SampleProcess",`` \\\
``  "targetProcessVersion": "2.0"`` \\\
``}'``

8. Finish the migration of one of the process instances by sending an empty PUT to the v2 deployment:
``curl --location --request PUT 'http://localhost:8082/SampleProcess/<processInstanceId>'`` \\\
``--header 'Accept: application/json'`` \\\
``--header 'Content-Type: application/json'`` \\\
``--data '{`` \\\
``}'``

* This log will show in the second deployment container:
``2024-09-26 12:59:18 10:59:18 INFO  traceId=, parentId=, spanId=, sampled= [or.jb.fl.mi.MigrationPlanService] (executor-thread-1) Process instance 54df1e19-0478-4a06-a9cc-272b832016a2 will be migrated from Process [processId=SampleProcess, processVersion=1.0] to Process [processId=SampleProcess, processVersion=2.0] with plan Sample Process Migration Plan``


* The database should be updated like this:\
``select id, endpoint, last_update_time, process_id, version, start_time, state`` \
``from processes`` \
``where id = <processInstanceId>;`` \
\
``"id"	"endpoint"	"last_update_time"	"process_id"	"version"	"start_time"	"state"`` \
``"54df1e19-0478-4a06-a9cc-272b832016a2"	"http://localhost:8082/SampleProcess"	"2024-09-26 10:59:18.633"	"SampleProcess"	"2.0"	"2024-09-26 10:58:39.049"	1`` 

* However, the tasks table is not updated, which might mean that the process instance migration wasn't correct: \
``select id, process_id, name, reference_name, started, state, completed, last_update`` \
``from tasks`` \
``where process_instance_id = '54df1e19-0478-4a06-a9cc-272b832016a2';`` \
\
``"id"	"process_id"	"name"	"reference_name"	"started"	"state"	"completed"	"last_update"`` \
``"32d27a96-9f48-4698-ae50-89679a842166"	"SampleProcess"	"validatePreliminarData"	"Validate preliminar data"	"2024-09-26 10:58:39.048"	"Ready"		"2024-09-26 10:58:39.048"``

## Issues observed
1. Seems that the migration plan (.mpf file) has to be in the v1 project. At that point, no plan exists yet, which might force developers to add it to the v1 deployment after it's in production. Having the node mappings as input parameters in the /migrate operation seems more logical.
2. The expected result is that the new active user task is "Validate final data", according to the process migration plan. However, "Validate preliminar data" still stays as the active task. It shouldn't, as the v2 doesn't define it in the BPMN process.
3. Migration is not finished until an operation is executed against specific process instances. It doesn't seem to be the most straightforward approach because in case there are migration issues in the plan, the admin won't notice it immediately in the same way it happens in jBPM v7. This approach doesn't fit massive process instance migration.
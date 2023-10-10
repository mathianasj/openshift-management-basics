# OpenShift Management Basics
This lab will walk through different ways to manage OpenShift resources.  We will walk through creating an application that is a SQL explorer for a deployed database first by CLI, then through files, then finally through some operator uses.  At the end we can tie the yaml files into helm and show how automation can work in prepration for the next series GitOps with ArgoCD.

## Prereq
Create a new project to work in, replace mylastname with your last name

    oc new-project lab-basics-mylastname

## Manual CLI Creation
The OpenShift CLI has a concept of creating new applications using input parameters to generate the resources.

### Deploy Database
The first time we will run through this exercise and not create a database server with persistent storage.  We will also kill the pod and run the query again to show that the tables are no longer present

    oc new-app --template=mariadb-ephemeral --param=MYSQL_USER=admin --param=MYSQL_PASSWORD=admin --param=MYSQL_ROOT_PASSWORD=admin

### Deploy Web SQL Explorer
This is a web base sql explorer we will use to create tables, add data, and query the tables.
1. Create the deployment for the sqlpad 
    
    `oc new-app -e SQLPAD_PORT=3000 -e SQLPAD_ADMIN=admin@example.com -e SQLPAD_ADMIN_PASSWORD=secret --image=quay.io/kahlai/sqlpad:6.11.3`
1. Expose the newly deployed application as a route so we can access the app.

    `oc create route edge --service=sqlpad`
1. Get the route to open in a new browser tab, use the url from the output of the following
    
    `oc get route --template='https://{{.spec.host}}{{"\n"}}' -o go-template sqlpad`

### Test ephemeral storage
1. Provide the email "admin@examaple.com" and password "secret"
1. Create a new connection
    1. click "Choose a connection" -> "New connection"
    1. Enter "mariadb" for connection name
    1. Select "mysql2" for driver
    1. Enter "mariadb" for Host/Server/IP Address
    1. Enter "sampledb" for database
    1. Enter "admin" for database username
    1. Enter "admin" fro database password
    1. Click on save
1. Run the following sql by pasting the below statement and clicking select all the text and click run.  At the bottom of the screen you will see 3. select * from names dobule click that row to see the table contents.

    ```
        create table names (first_name varchar(255));
        insert into names values ('Joe');
        select * from names;
    ```
1. Now lets delete the database pod and rerun the select statement to see what happens.

    ```oc delete pod -l deploymentconfig==mariadb```

1. Switch back to the sqlpad app and select highlight the `select * from names` line and click run.  You should get "Table 'sampledb.names' doesn't exist". This shows that by default all pods receive no storage.

### Rerun by attaching persistent storage
Since database rely on persistent storage to maintain table entries between restarts lets add storage to our database pod.

1. Create a persistent volume claim.  A persistent volume claim is a request to provision the underlying storage in an automated fashion.
    ```
    oc create -f pvc.yaml
    ```
1. Add storage to our existing deployment

    ```
    oc set volume dc/mariadb --remove --mount-path=/var/lib/mysql/data --confirm

    oc set volume dc/mariadb --add --type=persistentVolumeClaim --mount-path=/var/lib/mysql/data --claim-name mariadb
    ```
1. Run the following sql by pasting the below statement and clicking select all the text and click run.  At the bottom of the screen you will see 3. select * from names dobule click that row to see the table contents.

    ```
        create table names (first_name varchar(255));
        insert into names values ('Joe');
        select * from names;
    ```
1. Now lets delete the database pod and rerun the select statement to see what happens.

    ```oc delete pod -l deploymentconfig==mariadb```

1. Switch back to the sqlpad app and select highlight the `select * from names` line and click run.  Now this time you will see that the database was persisted between pod deletion.

## Create With Files
Creating resources with the CLI is nice for testing and learning, but to promote repeatablity and automation how do we utilize code that can be persistent and resuable.  This is where we use YAML to declare everything we have done.  To simplify the lab I have created the completed folder which contains all the files to create and configure the database we created above.  We will delete everything but the persistent storage to demonstrate the yamls are the same and the database contents are preserved.

### Destroy Existing Database Deployment
1. Run the following command

    ```
    oc delete dc/mariadb
    oc delete svc/mariadb
    ```
1. Look at the completed/mariadb.yaml and completed/mairadb-svc.yaml.  These two files create the deployment config and the service to access the database from the sqlpad.  Run the following commands to start a new db pod.

    ```
    oc create -f completed/mariadb.yaml
    oc create -f completed/mariadb-svc.yaml
    ```
1. Switch back to the sqlpad app and select highlight the `select * from names` line and click run.  You will see that we created a new database instance and utilized the existing persistent storage to get the existing table data.


# Learning Gen3

## Local Setup Quickstart

### Prerequisites
* **Important:** [Increase](https://docs.docker.com/docker-for-mac/#resources) the Docker LinuxVM memory to atleast 3 to 4 GB (or else loading the portal will take a very long time or crash)
* [Create](https://github.com/uc-cdis/compose-services#setting-up-google-oauth-client-id-for-fence) a Google OAUTH client id and password.
* Install [Kitematic](https://kitematic.com) or have [Docker Dashboard](https://docs.docker.com/desktop/dashboard/) to view Docker logs
* Install [PSequel](http://www.psequel.com) or another PostgreSQL querying application
* AWS Account (Use a free tier)

### Setup
1. Run `git clone https://github.com/uc-cdis/compose-services.git`
2. Navigate to `$ cd compose-services`
3. Run `$ bash ./creds_setup.sh`
4. Open in VS Code `$ code .`

5. Comment out the following in `./nginx.conf`
```
    location /guppy/ {
            proxy_pass http://guppy-service/;
    }
```
6. Comment out the YAML node at `services.kibana-service` in `./docker-compose.yml`
7. Uncomment the `services.postgres.ports` node in `./docker-compose.yml` to in to allow connecting to the local Postgres DB
```yaml
    ports:
     - 5432:5432
```
8. Replace `yourlogin1@gmail.com` with the email address you used to create the Google OAUTH client in `./Secrets/user.yaml`

### Run Docker Compose
1. Run `docker-compose up -d`
2. Open Kitematic and watch the logs. Wait until `portal-service` container is past the `./node_modules/.bin/webpack --bail` command in the logs. This means webpack is done loading. (Can take several minutes)
3. Run the smoke tests using `bash smoke_test.sh localhost` and confirm no errors
4. Visit `http://localhost` and login with your Google account associated with the OAUTH clientid

### Create Program
1. While local instance is running, visit `https://localhost/_root`
2. Enable _User Form Submission_ button and select _program_ from drop-down ![image](images/user_form.png)
3. Enter _123_ and _Program1_ ![image](images/user_form_1.png)
4. Click _Upload submission json from form_ and see json result ![image](images/user_form_2.png)

### Visit Your Program
1. Visit `https://localhost/Program1`
2. Verify it exists
3. **Optional but STRONGLY reccomended**: Check the database using `select * from node_program;`

### Create Project
1. While local instance is running, visit `https://localhost/Program1`
2. Enable _User Form Submission_ button and select _project_ from drop-down 
3. Enter _P1_, _phs1_, and _project1_ ![image](images/program.png)
4. Click _Upload submission json from form_ and see the green result. Note: You may have to hit the button **SEVERAL** times to see the green result. ![image](images/program_1.png)
5. Verify your project is Created by clicking the blue Details button and/or checking the database with `select * from node_project;`

### Update user.yaml
1. Under `authz.resources` path in the `Secrets/user.yaml` file, replace the _Old_ with the _New_, using the program name `Program1` value and project code `P1`
```yaml
# Old
  - name: programs
    subresources:
    - name: MyFirstProgram
      subresources:
      - name: projects
        subresources:
        - name: MyFirstProject
```

```yaml
# New - NOTE that Gen3 config demands P1 which is the project CODE not NAME
  - name: programs
    subresources:
    - name: Program1
      subresources:
      - name: projects
        subresources:
        - name: P1
```
2. Under `authz.policies` find the id `MyFirstProject_submitter` and update the `resource_paths` value of `/programs/MyFirstProgram/projects/MyFirstProject` to be `/programs/Program1/projects/P1`  
3. Refresh using `docker exec -it fence-service fence-create sync --arborist http://arborist-service --yaml user.yaml` to have the containers recognize the `user.yaml` changes 

### Generate Test Metadata
1. Generate test metadata for the `Program1` program and `P1` project
```console
$ export TEST_DATA_PATH="$(pwd)/testData"
$ mkdir -p "$TEST_DATA_PATH"

$ docker run -it -v "${TEST_DATA_PATH}:/mnt/data" --rm --name=dsim --entrypoint=data-simulator quay.io/cdis/data-simulator:master simulate --url https://s3.amazonaws.com/dictionary-artifacts/datadictionary/develop/schema.json --path /mnt/data --program Progam1 --project P1 --max_samples 10
```

### Upload Test Metadata to Project
1. Visit https://localhost/submission and click _Submit Data_
![image](images/submission.png)
2. Click _Upload file_ and choose the `core_metadata_collection.json` file from the `/testData` directory ![image](images/project_upload.png)
3. Click _Submit_ and make sure you see a Success 200 message ![image](images/project_upload_1.png)
4. **Optional:** Repeat steps 2 and 3 for other test metadata files. For instance after `experiment.json` and `experimental_metadata.json` have been uploaded you should see the following at https://localhost/Program1-P1  ![image](images/project_upload_2.png)
5. You are done with this setup guide.

### Recap
* We started the Gen3 environment, then created a program and project. 
* Then we uploaded _metadata_ that fit into the project data model. 
* We did NOT upload any data (e.g. images, documents, etc) files.

---
<!-- ### Download and configure up Gen3 Client
```console
$ ./gen3-client configure --profile=cse_profile --cred=~/Downloads/credentials.json --apiendpoint=http://localhost/
2020/12/12 14:47:41 Profile 'cse_profile' has been configured successfully.
```

A `.gen3` directory should existing in your current user directory (e.g. `/Users/<current-user>/.gen3/`. 
View your configuration using `cat /Users/<current-user>/.gen3/config `

### Verify Gen3 Client Access
Verify you have access:
```console
$ ./gen3-client auth --profile=cse_profile
2020/12/12 15:01:23 
You don't currently have access to data from any projects at http://localhost
```

If you get the above warning add the following to the end of your `Secrets/gitops.json` file. According to this [Slack post](https://cdis.slack.com/archives/CDDPLU1NU/p1607962367255400?thread_ts=1607822151.254000&cid=CDDPLU1NU) the message _You don't currently have access to data from any projects_ is misleading. 

```json
  "showArboristAuthzOnProfile": true, 
  "showFenceAuthzOnProfile": false
```

* You will need to restart the `portal-service` using `docker-compose restart portal-service` or shutdown the entire docker compose environment using `docker-compose down` and then `docker-compose up -d`

* After you log back in navigate to https://localhost/identity to verify you have access to resources
![image](images/profile.png) -->

<!-- ### Upload test data using Gen3 client

1. Use the test metadata you previously created at `/testData`
2. Run the command `gen3-client upload --profile=cse_profile --upload-path=testData/`
3. Wait for possible retries 

An example of the command and results are below. Notice that the uploads are not reliable and may take several retries. You can see a log of success and failure in `~/.gen3/logs/`

```console
$ ./gen3-client upload --profile=cse_profile --upload-path=testData/

<...snip...>

Submission Results
Finished with 0 retries | 11
Finished with 1 retry   | 0
Finished with 2 retries | 5
Finished with 3 retries | 6
Finished with 4 retries | 1
Finished with 5 retries | 2
Failed                  | 3
TOTAL                   | 28
``` -->

<!-- ### Mapping uploaded test metadata

From https://gen3.org/resources/user/gen3-client/#3-upload-data-files:
> Files that have been successfully uploaded now have a GUID associated with them, and there is also an associated record in the indexd database. However, in order for the files to show up in the data portal, the files have to be registered in the PostgreSQL database. In other words, indexd records exist for the files, but sheepdog records (that is, structured metadata in the graph model) don’t exist yet. Thus, the files aren’t yet associated with any particular program, project, or node. To create the structured data records for the files via the sheepdog service, Windmill offers a “Map My Files” UI

In the screenshot below you can see the 25 files that got uploaded. Click the _Map My Files button_
![image](images/submission.png) -->
## Known Issues
* In a local setup if you try to upload _data_ files (via Gen3 client) you will see a status of _generating..._ at https://localhost/submission/files. This is because the data file upload process needs AWS SNS & SQS to trigger a job (see Slack [here](https://cdis.slack.com/archives/CDDPLU1NU/p1580764187092100?thread_ts=1580763342.092000) and [here](https://cdis.slack.com/archives/CDDPLU1NU/p1590767430357200?thread_ts=1590733336.355100&cid=CDDPLU1NU)).

## Appendix

### Questions
* Mapping of files seems to be a very manual process. Is there a way to do bulk mapping of metadata to files (blobs) offline then just import once?

### Verify local Index Service is healthy
* Visit https://localhost/index/index/ or https://localhost/index/_status

### Debug Postgres

1. If you did not expose the port in Docker Compose you can connect to the local Postgres container with `docker exec -it compose-services_postgres_1 bash`.
2. Once inside the container run `psql -Atx postgresql://fence_user:fence_pass@localhost/metadata_db`

**Alternatively**, if you are exposing the ports you can connect using psql from your main console window using the following:
Verify _Program1_ exists in the DB
```console
$ psql -Atx postgresql://fence_user:fence_pass@localhost/metadata_db
psql (13.1, server 9.6.20)
Type "help" for help.

metadata_db=# select * from node_program;
created|2020-12-12 18:19:57.683685+00
acl|{}
_sysan|{}
_props|{"name": "Program1", "dbgap_accession_number": "123"}
node_id|38373fbb-01da-5050-ba61-a3579eda3a37
```
Verify _project1_ exists in the DB
```console
$ psql -Atx postgresql://fence_user:fence_pass@localhost/metadata_db
psql (13.1, server 9.6.20)
Type "help" for help.

metadata_db=# select * from node_project;
created|2020-12-16 21:58:36.833556+00
acl|{}
_sysan|{}
_props|{"code": "P1", "name": "project1", "state": "open", "dbgap_accession_number": "phs1"}
node_id|49a949ef-60e1-5dba-84b9-7e3b2d37cdaf
```
### Links

* [Gen3 Data Commons Setup - Part 1](https://www.youtube.com/watch?v=xM54O4aMpWY)
* [Gen3 Data Commons Setup - Part 2](https://www.youtube.com/watch?v=iMmCxnbHpGo)
* [Gen3 Data Commons - Data Upload Tutorial](https://www.youtube.com/watch?v=QxQKXlbFt00)




# Troubleshooting
Documentation for troubleshooting and debugging known issues in Shuffle.

## Table of contents
* [Load all apps locally](#load_all_apps_locally)
* [Force stop executions](#how_to_stop_executions_in_loop)
* [Opensearch permission errors](#opensearch_permissions_error)
* [Recover admin user](#recover_admin_user)
* [Orborus can't reach backend](#orborus_can_not_reach_backend)
* [Abort all specific workflow executions](#abort_all_running_executions_of_a_specific_workflow)
* [Useful Opensearch Query](##Useful Opensearch Query)
* [Extract all workflows](##Extract all workflows)
* [Recover Organizations](#recover_organizations)

## Load all apps locally
In certain cases, you may have an issue loading apps into Shuffle. If this is the case, it most likely means you have proxy issues, and can't reach github.com for our apps. Here's how to manually load them into Shuffle using git

```
#1. IF proxy necessary: Set up the proxy for Git (install if you don't have it).
git config --global http.proxy http://proxy.mycompany:80

#2. Go to the shuffle folder where you have Shuffle installed, then go to the shuffle-apps folder (./shuffle/shuffle-apps)
git clone https://github.com/shuffle/shuffle-apps

#3. Go to the UI and hotload the apps: https://shuffler.io/docs/app_creation#hotloading_your_app (click the hotload button in the top left in the /apps UI)
```

## Orborus can not reach backend
In certain cases there may be DNS issues, leading to hanging executions. This is in most cases due to apps not being able to find the backend in some way. That's why the best solution _if possible_ is to use the IP as hostname for Orborus -> Backend communication.

## How to stop executions in loop
1. Run **docker ps**
```
CONTAINER ID   IMAGE                                     COMMAND                  CREATED       STATUS      PORTS                                                  NAMES
869c99231ed0   opensearchproject/opensearch:latest       "./opensearch-docker…"   5 weeks ago   Up 2 days   9300/tcp, 9600/tcp, 0.0.0.0:9200->9200/tcp, 9650/tcp   shuffle-opensearch
```

2. Run **docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name_or_id**
```
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' 869c99231ed0
or
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' shuffle-opensearch
```
2.1. Output
```
172.21.0.4
```

3. Run **curl -XDELETE http://<container_ip>:9200/workflowqueue-shuffle**
```
curl -XDELETE http://172.21.0.4:9200/workflowqueue-shuffle
```
3.1. Output
```
{"acknowledged":true}
```

## Abort all running executions of a specific worflow
Follow Python scripts allows to massively stop all running executions of a worflow
```python
import sys
import requests

API_ENDPOINT = "http(s)://<shuffle_endpoint>/api/v1"
API_KEY = "<your_api_key>"
WORKFLOW_NAME = "<workflow_name>"

def main():
    headers = {
        "Authorization": "Bearer " + API_KEY
    }

    with requests.get(API_ENDPOINT + "/workflows", headers=headers) as response:
        response.raise_for_status()
        data = response.json()

    wflows = list(filter(lambda wf: wf.get("name") == WORKFLOW_NAME, data))
    if len(wflows) == 0:
        print("Workflow not found")
        return 2
    if len(wflows) != 1:
        print("Something goes wrong")
        return 1

    wflow = wflows[0]

    # Get executions
    # example: http(s)://<shuffle_endpoint>/api/v1/workflows/b519c8f7-e9b0-4b47-b93d-cac013e4522f/executions
    wflow_id = wflow.get("id")
    url = "{}/workflows/{}/executions".format(API_ENDPOINT, wflow_id)
    with requests.get(url, headers=headers) as response:
        response.raise_for_status()
        data = response.json()

    still_running = list(filter(lambda ex: ex.get("status") == "EXECUTING", data))

    for exec in still_running:
        exec_id = exec.get("execution_id")
        print("[INFO] We're going to abort execution with ID {}".format(exec_id))

        url = "{}/workflows/{}/executions/{}/abort".format(API_ENDPOINT, wflow_id, exec_id)

        with requests.get(url, headers=headers) as response:
            response.raise_for_status()
            data = response.json()

        if data.get("success"):
            print("[INFO] Excecution successfully aborted")
        else:
            print("[ERROR] Unable to abort execution")

    return 0

if __name__ == "__main__":
    sys.exit(main())
```
Copy the script into a file called `abort_running_executions.py` and run it with
```

**Use python or python3 depending of your environment**
python abort_running_executions.py
```
In order to work _requests_ Python library must be installed in your Python execution env.

## Opensearch permissons error
![image](https://user-images.githubusercontent.com/21691729/124939209-d5694580-e000-11eb-8025-e2d475432e1b.png)
![image](https://user-images.githubusercontent.com/21691729/124939333-ec0f9c80-e000-11eb-9e56-5fbd06c7bfe3.png)
Give permissions to shuffle-database folder
```
sudo chown 1000:1000 -R shuffle-database
```

## Recover admin user
1. docker exec to get bash session into opensearch container "docker exec -it "container id" bash

2. pump curl results of users index into user.log file curl -X GET "localhost:9200/users/_search?pretty" -H 'Content-Type: application/json' -d'
{
	"query": {
		"match_all": {}
	}
}
' > users.log

3. open the users.log file with 'vi' and search for the admin user. Once found scroll down to the "apikey" section. this value will be the api key of the admin user.

4. I jumped onto another server within the same vlan as my shuffle server but these could be ran on local host too. We will create a new user and update the user's role to admin with the shuffle api

5. curl https://*ip of shuffle server/api/v1/users/register -H "Authorization: Bearer APIKEY" -d '{"username": "username", "password": "P@ssw0rd"}'    - will create new user

6. curl https://ip of shuffle server/api/v1/users/getusers -H "Authorization: Bearer APIKEY" - get the userid of the newly created user

7. curl https://ip of shuffle server*/api/v1/users/updateuser -H "Authorization: Bearer APIKEY" -d '{"user_id": "USERID", "role": "admin"}' - will set new user to the admin role.

8. Log into webui with the new user and you should have admin rights


## Useful Opensearch Query

### Find an user and its Orgs

```
curl -X GET "localhost:9200/users/_search?pretty" -H 'Content-Type: application/json' -d' { "query": { "match": {"username": "myuser"} } }

```

### Find all org

```
curl -X GET "localhost:9200/organizations/_search?pretty" -H 'Content-Type: application/json' -d' { "size": 10000, "query": { "match_all": {}}}'
```

Find all org IDs

```
curl -X GET "localhost:9200/organizations/_search?pretty" -H 'Content-Type: application/json' -d' { "size": 10000, "query": { "match_all": {}}}' | grep "\"id\" : \"" | sed 's/ *$//g' | sed 's/^[ \t]*//;s/[ \t]*$//' | uniq -u
```

# Extract all workflows

This procedure can help you extract workflows directly form opensearch even if Backend and FrontEnd are in awkward situation.

1. Extract the index info from Opensearch. (you may need to create a bind mount for the backup extraction)
```
curl -X GET "localhost:9200/workflow/_search?pretty" -H 'Content-Type: application/json' -d' { "size": 10000, "query": { "match_all": {}}}' > /mnt/backup/workflows.json
```

2. Script to separate all workflows

```

import json
import os

data = {}

with open("workflows.json", "r") as tmp:
    data = json.loads(tmp.read())

foldername = "./workflows_loaded"
try:
    os.mkdir(foldername)
except:
    pass

## Will break  with keyerror lol
for item in data["hits"]["hits"]:
    #print(item)
    try:
        item = item["_source"]
    except:
        continue

    filename = f"""{foldername}/{item["name"]}.json"""
    print(f"Writing {filename}")
    with open(filename, "w+") as tmp:
        tmp.write(json.dumps(item))

```
This script need to be run on the folder with the file workflows.json, it will create an workflows_loaded directory with all the workflows in it.
This can also be very useful either to backup your work or to export from a lab to a prod instance.

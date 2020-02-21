# Install and setup DroneCI

DroneCI is OSS Continuous Integration and Continuous Delivery platform 
that uses Docker container for build and, test and release workflow like pipeline.

Reference: https://docs.drone.io/

In this step, use DroneCI in Docker container only use CI tools(no deploy) 
and push pull-request to target github repository.

## Install ngrok

ngrok is local server binds NATs and firewalls to the public internet over secure tunnels.
ngrok is useful to get a tempolary DNS and callback URL for test.

The reason of using ngrok is that Github OAuth needs callback URL
(DroneCI requires OAuth).

Reference: https://ngrok.com/product

### Download and start ngrok

refer https://ngrok.com/download .

* Open `https://ngrok.com/download` and click `Download for Linux`
* Unzip download file `ngrok.zip`
* Start specific port 8080(this is sample) `./ngrok http 8080`

## Install and setup DroneCI

In this step, use Github as CI target for DroenCI.
When use github, setup as below.

### Create OAuth client for DroneCI

refer https://docs.drone.io/server/provider/github/ .

* Open Github page
* Setup OAuth for DroneCI, refer `https://docs.drone.io/server/provider/github/`
  * Click avator and select `Settings > Developer settings > OAuth Apps`
  * Set the dummy callback URL(such as `example.com`) in `Authorization callback URL` when use ngrok
  * Create a shared-secret(This secret keeps in your local disk or private git repositroy)
  * When setup done, you can get `Client ID` and `Client Secret`.  
    (`Client ID` and `Client Secret` set in DroenCI config.)

### Create Docker-compose file

We use Docker-compose to exec DroneCI.

First, create `docker-compose.yaml` as below.

Should set these parameters.
* `DRONE_GITHUB_CLIENT_ID`
* `DRONE_GITHUB_CLIENT_SECRET`
* `DRONE_RPC_SECRET`

```
version: '3'

services:
  drone-server:
    # Use latest version,
    # Reference: https://hub.docker.com/r/drone/drone/tags
    image: drone/drone:latest
    ports:
      - 8080:80
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data/:/data
    restart: always
    environment:
      - DRONE_GITHUB_SERVER=https://github.com
      - DRONE_GITHUB_CLIENT_ID=<OAuth client ID>
      - DRONE_GITHUB_CLIENT_SECRET=<OAuth client secret>
      - DRONE_RPC_SECRET=<Created by `openssl rand -hex 16`>
      - DRONE_SERVER_HOST=12345abc.ngrok.io # set ngrok url such as `12345abc.ngrok.io`.
      - DRONE_SERVER_PROTO=http # set `http` when use ngrok.
      - DRONE_TLS_AUTOCERT=false # set `false` when use ngrok.

  drone-agent:
    image: drone/agent:latest
    command: agent
    restart: always
    depends_on:
      - drone-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - DRONE_RPC_SERVER=http://drone-server
      - DRONE_RPC_SECRET=<same drone-sever's one Created by `openssl rand -hex 16`>
      - DRONE_RUNNER_CAPACITY=2
```

When customize paramters, refer `https://docs.drone.io/server/provider/github/`.

#### Note

When use ngrok, should get ngrok's host name accessable from intenet.

Start ngrok(port: `8080`) and get DNS name.

```
ngrok http 8080

ngrok by @inconshreveable
Session Status                online
Session Expires               7 hours, 43 minutes
Version                       2.3.35
Region                        United States (us)
Web Interface                 http://127.0.0.1:4040
Forwarding                    http://12345abc.ngrok.io -> http://localhost:8080
Forwarding                    https://12345abc.ngrok.io -> http://localhost:8080
```

Check DNS name and set `12345abc` to `DRONE_SERVER_HOST` in `docker-compose.yaml`.

#### CAUTION

Don't upload `docker-compose.yaml` to public repository without masking secrets.
If you want manage this yaml-file, should use *private* ripository or,
mask secrets before upload public repository.


### Execute DroenCI

Start DroneCI.

```
docker-compose up -d
Creating network "droneci_default" with the default driver
Creating droneci_drone-server_1 ... done
Creating droneci_drone-agent_1  ... done
```

Check DroneCI.

```
docker-compose ps
         Name                   Command           State               Ports            
---------------------------------------------------------------------------------------
droneci_drone-agent_1    /bin/drone-agent agent   Up                                   
droneci_drone-server_1   /bin/drone-server        Up      443/tcp, 0.0.0.0:8080->80/tcp

curl localhost:8080/version
{"version":"1.6.5"}
```

## Access to DroneCI through ngrok

Make sure accessable to DroneCI through ngrok.
```
curl http://12345abc.ngrok.io/version
{"version":"1.6.5"}
```
## Fix OAuth config

We have set callback URL(such as `example.com`),
then fix it correct URL(`12345abc.ngrok.io` showned by ngrok).

* Open Github page
* Click avator and select `Settings > Developer settings > OAuth Apps`
* Set the currect callback URL(such as `12345abc.ngrok.io/login`) in `Authorization callback URL` when use ngrok 

*Note* Don't forget input `/login` into callback URL.

## Authenticate DroneCI

Access to `http://12345abc.ngrok.io` from web browser.

Click `Authorize <Your account name having OAuth token>`.

After OAuth login, show DroneCI's UI in web browser.

## Activate your application

After authenticated, show repository list.

Select an application to hook DroneCI and click `ACTIVATE` then
creates webhook in repository of setting.

# Trouble shooting

Encounter something wrong, check below topics.

## DroneCI doesn't work

### 1. Check docker-compose

First, check status of DroneCI built by docker-compose.

You can check status by this command.

```
# stop and start DroneCI
docker-compose down
docker-compose up -d
# show logs
docker-compose logs
```

When some error occurred, you should fix `docker-compose.yaml`.

#### Check points:
* services.volumes
  * docker.sock mounts `/var/run/docker.sock:/var/run/docker.sock`
  * docker is running? (When docker is stopping, `systemctl start docker`)
* environment
  * DRONE_GITHUB_SERVER
  * DRONE_GITHUB_CLIENT_ID
    * This ID is provied from github(This is `OAuth client ID`)
  * DRONE_GITHUB_CLIENT_SECRET
    * This ID is provied from github
  * DRONE_PRC_SECRET
    * Created by `openssl -hex 16`
    * Set same one in `drone-agent.environment.DRONE_RPC_SECRET`
  * DRONE_SERVER_HOST
    * Set host name(DNS name) provideed from ngrok

#### Test:
Access to `localhost:8080/version`(command is `curl localhost:8080/version`).

DroneCI works when returns version such as `{"version":"1.6.5"}`.

### 2. Check ngrok

If ngrok doesn't work, start ngrok `ngrok http 8080`.
`8080` means host port that set in docker-compose's at `service.drone-server.port`.

#### Test:
Access to `<ngrok DNS name>/version`.(command is `curl <ngrok DNS name>/version`)

DroneCI works when returns version such as `{"version":"1.6.5"}`.

### 3. Check OAuth apps setting

Check below parameters of github OAuth apps.

* Client ID
  * Set `DRONE_GITHUB_CLIENT_ID` to Client ID in `docker-compose.yaml`
* Client Secret
  * Set `DRONE_GITHUB_CLIENT_SECRET` to Client Secret in `docker-compose.yaml`
* Check `Authorization callback URL`
  * Check it is same ngrok DNS name + '/login'(`<ngrok DNS name>/login`)

Finaly, access to `<ngrok DNS name>` and click `Authorize <Your account name having OAuth token>`.

### 4. Avoid ngrok/Too many connection

The ngrok's connection limit is 20.
When encounterd this problem, wait over one minute and acess again.

# Finaly

Enjoy DroenCI!

Thank you.

# OpenRadiant Proxy
Proxy intercepts the communication between the clients (Docker or Kubernetes) and
the OpenRadiant cluster, using HTTP session hijacking. It validates the tenant
and provided TLS certificates. It also redirects to proper shard when cluster
sharing is used.

## Proxy Environment Setup and Development
Steps below explain the basic configuration and setup to run Proxy. This setup
assumes the target system already has Docker installed, the Proxy will be deployed
locally to manage Swarm and Kubernetes instances also installed locally, using
the ansible scripts described in [Quick Start](../README.md#quick-start)
For other deployments, one can still follow these steps, with adjustments provided below.

The steps below work best with native docker [for Mac, Unix or Windows](http://www.docker.com/products/overview)
It would also work with `docker-machine`, but it requires additional steps. Look
for /[DOCKER MACHINE] tag.

### Step 1: Get proxy code
If you have not done this already, clone the repository:

```
git clone git@github.ibm.com:alchemy-containers/openradiant.git
# or
git clone https://github.ibm.com/alchemy-containers/openradiant.git
```
Then build and deploy it:
```
cd openradiant/proxy
./builddocker.sh
./rundocker.sh
```

This will start the Proxy as a container named `hjproxy`, running in the current terminal.
If you want to run the Proxy as background container (daemon), use the `-d` flag:
```
./rundocker.sh -d
```
The Proxy container can be seen along with its logs:
```
docker ps
docker logs -f hjproxy
```

If you don't want to run the Proxy as a container, you can run directly as GoLang
application. This requires Go libraries installed, proxy code compiled and added
to go path.
Review [Dockerfile](dockerize/Dockerfile) for more details.
Setup the environment and start the application:
```
source ./set_local_env.sh
./start_proxy.sh
```


### Step 2: Setting up the tenant
From another terminal go back to the proxy directory and then execute the script
to create TLS certificates and API key for the given tenant. The example is
using tenant `test1`
```
docker ps
docker exec hjproxy /hijack/make_TLS_certs.sh test1
```

This command will display the details about the newly created TLS certs, including
their location. Certificates are created on the mounted volumes, therefore they
are available inside the proxy container and for the current terminal.

You can create as many tenants as you like.

At the bottom of the output the script displays the docker environment setup for
the newly created tenant, including the location of the certs. Here is the sample
output:
```
Execute the following:
export DOCKER_HOST=localhost:8087
export DOCKER_TLS_VERIFY=1
export DOCKER_CONFIG=dockerize/OpenRadiant/1z5AmY6uDzqBT65mkPtffEhOutcxs3sghn9S9LrXfAOztCpR
export DOCKER_CERT_PATH=dockerize/OpenRadiant/1z5AmY6uDzqBT65mkPtffEhOutcxs3sghn9S9LrXfAOztCpR
```
Copy the last four lines and paste in a new terminal. Make sure you are in
`openradiant/proxy` directory.

Now you should be able to execute commands against OpenRadiant account that you
just created:

```
docker ps
docker run -d --name test --net none -m 128m mrsabath/web-ms
docker inspect test
```
To run the proxy against a different OpenRadiant deployment or cluster, change
the IP of the TARGET_SERVER value to this new location in `make_TLS_certs.sh` script.
By default it is set to the local server running on `radiant2` in vagrant:
`TARGET_SERVER="192.168.10.2"`, then rebuild and redeploy the Proxy.
The newly created tenant will be pointing at the new location.
You can also manually change the values in "/hijack/creds.json" file that lists
all the valid tenants.  
To modify the value in the running container:
```
docker exec -it hjproxy /bin/bash
```

**NOTE**: All default config options are defined in the [Dockerfile](dockerize/Dockerfile),
and can be overridden using the docker -e option on [startup](rundocker.sh)


## Hints and Troubleshooting Errors

 * `Error response from daemon: client is newer than server (client API version: 1.24, server API version: 1.22)`
 run `export DOCKER_API_VERSION=1.22` before running any docker commands.

 * `An error occurred trying to connect: Get https://localhost:8087/v1.21/images/4a419cdeaf69/json: tls: oversized record received with length 20527[]`
 In order to fix this problem, use `DOCKER_TLS_VERIFY=""` prefix for running 'docker' command

 * `docker: Error response from daemon: Task launched with invalid offers: Offer ea1a4d71-cf69-4292-90e7-530c77a5458b-O1 is no longer valid.`
 There is a caching problem on Mesos. Issue [#100](https://github.ibm.com/alchemy-containers/openradiant/issues/100)
 is tracking it. Simply just repeat your last command. It should purge the cache
 and work again.

 * `docker: Error response from daemon: driver failed programming external connectivity on endpoint hjproxy (0910f89f1b27f3b05081a0bcec3ceadb6d335873d191b3f055ff82257cf77e5d): Error starting userland proxy: write /port/tcp:0.0.0.0:8087:tcp:172.17.0.2:8087/ctl: errno 526.` Please make sure no
 other process is running on port specified for proxy. Standalone proxy test on 8087?

  * `Could not read CA certificate "dockerize/OpenRadiant/fprVv76aAWfrmxboOxsO6dbzfZcITidkIwBslPgMAchFfwZI/ca.pem": open dockerize/OpenRadiant/fprVv76aAWfrmxboOxsO6dbzfZcITidkIwBslPgMAchFfwZI/ca.pem: no such file or directory`
  Are you sure you are running your docker commands from `openradiant/proxy/`
  directory? 


## Running Test Scripts

* Test scripts are located in `proxy-test` folder, and should be run from this directory.

* Execute `source setup_local.sh` or `source setup_CCSAPI.sh`. Former tests local file authentication, latter, CCSAPI authentication.

* Run with the command `./test_containers.sh da07` (can also be run with no argument).

* Results delineated in file `test_containers_results.txt`.
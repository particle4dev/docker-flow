Docker Flow
===========

* [Introduction](#introduction)
* [Examples](#examples)
* [Usage](#usage)

  * [Setting it up](#setting-it-up)
  * [Reconfiguring proxy after deployment](#reconfiguring-proxy-after-deployment)
  * [Deploying a new release without downtime](#deploying-a-new-release-without-downtime)
  * [Scaling the service](#scaling-the-service)
  * [Testing deployments to production](#testing-deployments-to-production)
  * [Using custom Consul templates](#using-custom-consul-templates)

* [Feedback and Contribution](#feedback-and-contribution)

Introduction
------------

*Docker Flow* is a project aimed towards creating an easy to use continuous deployment flow. It depends on [Docker Engine](https://www.docker.com/products/docker-engine), [Docker Compose](https://www.docker.com/products/docker-compose), [Consul](https://www.consul.io/), and [Registrator](https://github.com/gliderlabs/registrator). Each of those tools is proven to bring value and are recommended for any Docker deployment.

The goal of the project is to add features and processes that are currently missing inside the Docker ecosystem. The project, at the moment, solves the problems of blue-green deployments, relative scaling, and proxy service discovery and reconfiguration. Many additional features will be added soon.

The current list of features is as follows.

* Blue-green deployment
* Relative scaling
* Proxy reconfiguration

The latest release can be found [here](https://github.com/vfarcic/docker-flow/releases/latest).

> For a detailed explanation of features, walkthrough, and examples, please read the **[Docker Flow: Walkthrough](https://technologyconversations.com/2016/04/18/docker-flow/)** article.

Examples
--------

The examples that follow assume that you have Docker Machine and Docker Compose installed. The easiest way to get them is through [Docker Toolbox](https://www.docker.com/products/docker-toolbox).

> If you are a Windows user, please run all the examples from *Git Bash* (installed through *Docker Toolbox*).

We'll use [Docker Machine](https://www.docker.com/products/docker-machine) to simulate a [Docker Swarm](https://www.docker.com/products/docker-swarm) cluster. That does not mean that the usage of **Docker Flow** is limited to either of those two. You can use it with a single [Docker Engine](https://www.docker.com/products/docker-engine) or a Swarm cluster set up in any other way.


### Setting it up

Please clone the code from this repository.

```sh
git clone https://github.com/vfarcic/docker-flow.git

cd docker-flow
```

Before proceeding further, download the [latest release](https://github.com/vfarcic/docker-flow/releases) to the *docker-flow* directory and make it executable.

We'll start by creating a server that will host Consul and Proxy as well as a Swarm cluster consisting of three nodes. As you'll see soon, the proxy will be provisioned automatically so, for the first server, we only need to create the machine and run Consul. We'll skip the explanation all the commands required for the setup of those four servers, and focus on the *Docker Flow* features. If you are interested in setup details, please explore the [setup.sh](https://github.com/vfarcic/docker-flow/blob/master/setup.sh) script.

Please run the following commands.

```
chmod +x setup.sh

./setup.sh
```

Let's take a look at the state of our Swarm cluster.

```bash
eval "$(docker-machine env --swarm swarm-master)"

docker ps -a
```

The output of the `ps` command is as follows.

```
ONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS               NAMES
ed3bf310d9c5        gliderlabs/registrator   "/bin/registrator -ip"   14 minutes ago      Up 14 minutes                           swarm-node-2/registrator
23d40aad10c5        gliderlabs/registrator   "/bin/registrator -ip"   15 minutes ago      Up 15 minutes                           swarm-node-1/registrator
89a2aa6f5651        gliderlabs/registrator   "/bin/registrator -ip"   17 minutes ago      Up 17 minutes                           swarm-master/registrator
6be0773f0008        swarm:latest             "/swarm join --advert"   17 minutes ago      Up 17 minutes                           swarm-node-2/swarm-agent
1fb37d0463a3        swarm:latest             "/swarm join --advert"   18 minutes ago      Up 18 minutes                           swarm-node-1/swarm-agent
549df450cb91        swarm:latest             "/swarm join --advert"   20 minutes ago      Up 20 minutes                           swarm-master/swarm-agent
38ea9a4b449f        swarm:latest             "/swarm manage --tlsv"   20 minutes ago      Up 20 minutes                           swarm-master/swarm-agent-master
```

As you can see, all three Swarm nodes are running *Registrator* and *Swarm Node* container. On top of that we have *Swarm Master* running on the main server.

Aside from the Swarm cluster, we have a lone server called *proxy* that is, at the moment, running only Consul.

```bash
eval "$(docker-machine env proxy)"

docker ps -a
```

The output of the `ps` command is as follows.

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
ccb5e812b7e1        consul              "docker-entrypoint.sh"   23 minutes ago      Up 23 minutes                           consul
```

With the cluster and the proxy server set up, we are ready to give **Docker Flow** a spin and see it in action.

### Reconfiguring proxy after deployment

*Docker Flow* requires the Consul address as well as the information about the node the proxy is (or will be) running on. It allows three ways to provide the necessary information. We can define arguments inside the *docker-flow.yml* file, as environment variables, or as command line arguments. In this example, we'll use all three input methods so that you can get familiar with them and choose the combination that suits you needs.

Let's start by defining proxy and Consul data through environment variables.

```bash
export PROXY_IP=$(docker-machine ip proxy)

export CONSUL_IP=$(docker-machine ip proxy)

export FLOW_PROXY_HOST=$(docker-machine ip proxy)

export CONSUL_IP=$(docker-machine ip proxy)

export FLOW_CONSUL_ADDRESS=http://$CONSUL_IP:8500

eval "$(docker-machine env proxy)"

export FLOW_PROXY_DOCKER_HOST=$DOCKER_HOST

export FLOW_PROXY_DOCKER_CERT_PATH=$DOCKER_CERT_PATH
```

The *FLOW_PROXY_HOST* variable is the IP of the host where the proxy is running while the *FLOW_CONSUL_ADDRESS* represents the full address of the Consul API. The *FLOW_PROXY_DOCKER_HOST* and *FLOW_PROXY_DOCKER_CERT_PATH* are the host and the certification path of the Docker Engine running on the server with the proxy. *Docker Flow* runs operations on multiple servers at the same time so we need to provide it with all the information it needs to do its tasks. In the examples we are exploring, it will deploy containers on the Swarm cluster, use Consul instance to store and retrieve information, and reconfigure the proxy every time a new service is deployed.

Now we are ready to deploy the first release of our sample service.

```bash
eval "$(docker-machine env --swarm swarm-master)"

./docker-flow \
    --blue-green \
    --target=app \
    --service-path="/demo" \
    --side-target=db \
    --flow=deploy --flow=proxy
```

We instructed `docker-flow` to use the *blue-green deployment* process and that the target (defined in [docker-compose.yml](https://github.com/vfarcic/docker-flow/blob/master/docker-compose.yml)) is *app*. We also told it that the service exposes an API on the address */demo that it requires a side (or secondary) target *db*. Finally, through the `--flow` arguments we specified that the we want it to deploy the targets and reconfigure the proxy. A lot happened in that single command so we'll explore the result in more detail.

Let's take a look at our servers and see what happened. We'll start with the Swarm cluster.

```bash
docker ps --format "table {{.Names}}\t{{.Image}}"
```

The output of the `ps` command is as follows.

```
NAMES                                 IMAGE
swarm-node-2/dockerflow_app-green_1   vfarcic/go-demo
swarm-node-1/dockerflow_db_1          mongo
...
```

*Docker Flow* run our main target *app* together with the side target named *books-ms-db*. Both targets are defined in [docker-compose.yml](https://github.com/vfarcic/docker-flow/blob/master/docker-compose.yml). Container names depend on many different factors, some of which are the Docker Compose project (defaults to the current directory as in the case of the *app* target) or can be specified inside the *docker-compose.yml* through the `container_name` argument as in the case of the *db* target). The first difference you'll notice is that *Docker Flow* added *blue* to the container name. The reason behind that is in the `--blue-green` argument. If present, *Docker Flow* will use the *blue-green* process to run the primary target. If you are unfamiliar with the process, please read the [Blue-Green Deployment](http://technologyconversations.com/2016/02/08/blue-green-deployment/) article for general information and [Docker Flow: Blue-Green Deployment and Relative Scaling](http://technologyconversations.com/2016/03/07/docker-flow-blue-green-deployment-and-relative-scaling/) for a more detailed explanation within the *Docker Flow* context.

Let's take a look at the *proxy* node as well.

```bash
eval "$(docker-machine env proxy)"

docker ps --format "table {{.Names}}\t{{.Image}}"
```

The output of the `ps` command is as follows.

```
NAMES               IMAGE
docker-flow-proxy   vfarcic/docker-flow-proxy
consul              consul
```

*Docker Flow* detected that there was no *proxy* on that node and run it for us. The *docker-flow-proxy* container contains *HAProxy* together with custom code that reconfigures it every time a new service is run. For more information about the *Docker Flow: Proxy*, please read the [project README](https://github.com/vfarcic/docker-flow-proxy).

Since we instructed Swarm to run our service somewhere inside the cluster, we could not know in advance which server will be chosen. In this particular case, our service ended up running inside the *swarm-node-2*. Moreover, to avoid potential conflicts and allow easier scaling, we did not specify which port the service should expose. In other words, both the IP and the port of the service were not defined in advance. Among other things, *Docker Flow* solves this by running *Docker Flow: Proxy* and instructing it to reconfigure itself with the information gathered after the container is run. We can confirm that the proxy reconfiguration was indeed successful by sending an HTTP request to the newly deployed service.

```bash
curl -i $PROXY_IP/demo/hello
```

The output of the `curl` command is as follows.

```
HTTP/1.1 200 OK
Date: Sun, 22 May 2016 17:23:41 GMT
Content-Length: 14
Content-Type: text/plain; charset=utf-8

hello, world!
```

Even though our service is running in one of the servers chosen by Swarm and is exposing a random port, the proxy was reconfigured and we can access it through a fixed IP and without a port (to be more precise through standard HTTP port 80 or HTTPS port 443).

### Deploying a new release without downtime

Let's imagine that we want to deploy a new release of the service. We do not want to have any downtime so we'll continue using the *blue-green* process. Since the current release is *blue*, the new one will be named *green*. Downtime will be avoided by running the new release (*green*) in parallel with the old one (*blue*) and after it is fully up and running, reconfigure the proxy so that all requests are sent to the new release. Only after the proxy is reconfigured, we want the old release to stop running and release the resources it was using. We can accomplish all that by running a `docker-flow` command. However, this time, we'll leverage the [docker-flow.yml](https://github.com/vfarcic/docker-flow/blob/master/docker-flow.yml) file that already has some of the arguments we used before.

The content of the [docker-flow.yml](https://github.com/vfarcic/docker-flow/blob/master/docker-flow.yml) is as follows.

```yml
target: app
side_targets:
  - db
blue_green: true
service_path:
  - /demo
```

Let's run the new release.

```bash
eval "$(docker-machine env --swarm swarm-master)"

./docker-flow \
    --flow=deploy --flow=proxy --flow=stop-old
```

Just like before, we should explore the Docker processes and see the result.

```bash
docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

The output of the `ps` command is as follows.

```bash
NAMES                                 IMAGE                    STATUS
swarm-node-2/dockerflow_app-green_1   vfarcic/go-demo          Up 7 seconds
swarm-master/dockerflow_app-blue_1    vfarcic/go-demo          Exited (2) 6 seconds ago
swarm-node-1/dockerflow_db_1          mongo                    Up 12 minutes
...
```

From the output, we can observe that the new release (*green*) is running and that the old (*blue*) was stopped. The reason the old release was only stopped and not entirely removed lies in potential need to quickly roollback in case a problem is discovered at some later moment in time.

Let's confirm that the proxy was reconfigured as well.

```bash
curl -i $PROXY_IP/demo/hello
```

The output of the `curl` command is as follows.

```
HTTP/1.1 200 OK
Date: Sun, 22 May 2016 17:25:19 GMT
Content-Length: 14
Content-Type: text/plain; charset=utf-8

hello, world!
```

The new release was deployed without any downtime and the proxy has been reconfigured to redirect all requests to it.

### Scaling the service

One of the great benefits *Docker Compose* provides is scaling. We can use it to scale to any number of instances. However, it allows only absolute scaling. We cannot instruct *Docker Compose* to apply relative scaling. That makes the automation of some of the processes difficult. For example, we might have an increase in traffic that requires us to increase the number of instances by two. In such a scenario, the automation script would need to obtain the number of instances that are currently running, do some simple math to get to the desired number, and pass the result to Docker Compose. On top of all that, proxy still needs to be reconfigured as well. *Docker Flow* makes this process much easier.

Let's see it in action.

```bash
./docker-flow \
    --scale="+2" \
    --flow=scale --flow=proxy
```

The scaling result can be observed by listing the currently running Docker processes.

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

The output of the `ps` command is as follows.

```
NAMES                                 IMAGE                    STATUS
swarm-node-2/dockerflow_app-green_3   vfarcic/go-demo          Up 8 seconds
swarm-node-1/dockerflow_app-green_2   vfarcic/go-demo          Up 8 seconds
swarm-node-2/dockerflow_app-green_1   vfarcic/go-demo          Up About a minute
```

The number of instances was increased by two. While only one instance was running before, now we have three.

Similarly, the proxy was reconfigured as well and, from now on, it will load balance all requests between those three instances.

We can use the same method to de-scale the number of instances by prefixing the value of the `--scale` argument with the minus sign (*-*). Following the same example, when the traffic returns to normal, we can de-scale the number of instances to the original amount by running the following command.

```bash
./docker-flow \
    --scale="-1" \
    --flow=scale --flow=proxy
```

### Testing deployments to production

The major downside of the proxy examples we run by now is the inability to verify the release before reconfiguring the proxy. Ideally, we should use the blue-green process to deploy the new release in parallel with the old one, run a set of tests that validate that everything is working as expected, and, finally, reconfigure the proxy only if all tests were successful. We can accomplish that easily by running `docker-flow` twice.

> Many tools aim at providing zero-downtime deployments but only a few of them (if any), take into account that a set of tests should be run before the proxy is reconfigured.

First, we should deploy the new version.

```bash
./docker-flow \
    --flow=deploy
```

Let's list the Docker processes.

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

The output of the `ps` command is as follows.

```
NAMES                                 STATUS              PORTS
swarm-master/dockerflow_app-blue_2    Up 8 seconds        192.168.99.101:32769->8080/tcp
swarm-node-2/dockerflow_app-blue_1    Up 9 seconds        192.168.99.103:32771->8080/tcp
swarm-node-1/dockerflow_app-green_2   Up About a minute   192.168.99.102:32768->8080/tcp
swarm-node-2/dockerflow_app-green_1   Up 2 minutes        192.168.99.103:32769->8080/tcp
...
```

At this moment, the new release (*blue*) is running in parallel with the old release (*green*). Since we did not specify the *--flow=proxy* argument, the proxy is left unchanged and still redirects to all the instances of the old release. What this means is that the users of our service still see the old release while we have the opportunity to test it. We can run integration, functional, or any other type of tests and validate that the new release indeed meets the expectations we have. While testing in production does not exclude testing in other environments (e.g. staging), this approach gives us greater level of trust by being able to validate the software under exactly the same circumstances our users will use it while, at the same time, not affecting them during the process (they are still oblivious of the existence of the new release).

> Please note that even though we did not specify the number of instances that should be deployed, *Docker Flow* deployed the new release and scaled it to the same number of instances as we had before.

After the tests are run, we have two paths we can take. If one of the tests failed, we can just stop the new release and fix the problem. Since the proxy is still redirecting all requests to the old release, our users were not affected by this failure, and we can dedicate our time towards fixing the problem. On the other hand, if all tests were successful, we can run the rest of the *flow* that will reconfigure the proxy and stop the old release.

```bash
./docker-flow \
    --flow=proxy \
    --flow=stop-old
```

### Using custom Consul templates

While, in most cases, the automatic proxy configuration is all you need, you might have a particular use case that would benefit from custom Consul templates. In such a case, you'd prepare your own templates and let *Docker Flow* use them throughout the process.

For more information regarding templating format, please visit the [Consul Template](https://github.com/hashicorp/consul-template) project.

The following example illustrates the usage of custom Consul templates.

```bash
eval "$(docker-machine env --swarm swarm-master)"

./docker-flow \
    --consul-template-fe-path test_configs/tmpl/go-demo-app-fe.tmpl \
    --consul-template-be-path test_configs/tmpl/go-demo-app-be.tmpl \
    --flow=deploy --flow=proxy --flow=stop-old
```

*Docker Flow* processed the template located in the `test_configs/tmpl/go-demo-app.tmpl` file and sent the result to the proxy. The rest of the process was the same as explained earlier.

Let's confirm whether the proxy was indeed configured correctly.

```bash
curl -i $PROXY_IP/demo/hello
```

The content of the [test_configs/tmpl/go-demo-app-fe.tmpl](https://github.com/vfarcic/docker-flow/blob/master/test_configs/tmpl/go-demo-app-fe.tmpl) and [test_configs/tmpl/go-demo-app-be.tmpl](https://github.com/vfarcic/docker-flow/blob/master/test_configs/tmpl/go-demo-app-be.tmpl) files is as follows.

```
	acl url_test-service path_beg /demo
	use_backend go-demo-app-be if url_test-service
```

```
backend go-demo-app-be
	{{ range $i, $e := service "SERVICE_NAME" "any" }}
	server {{$e.Node}}_{{$i}}_{{$e.Port}} {{$e.Address}}:{{$e.Port}} check
	{{end}}
```

It is a standard Consul template file with one exception. *SERVICE_NAME* will be replaced with the name of the service. You are free to create the Consul template in any form that suits you.

That concludes the quick tour through some of the features *Docker Flow* provides. Please explore the [Usage](#usage) section for more details.

Usage
-----

Arguments can be specified through *docker-flow.yml* file, environment variables, and command line arguments. If the same argument is specified in several places, command line overwrites all others, and environment variables overwrite *docker-flow.yml*.

### Command line arguments

|Command argument                     |Description|
|-------------------------------------|-----------|
|-b, --blue-green                     |Perform blue-green deployment. (**bool**)|
|    --cert-path=                     |Docker certification path. If not specified, DOCKER_CERT_PATH environment variable will be used instead.|
|-f, --compose-path=docker-compose.yml|Path to the Docker Compose configuration file. (default: docker-compose.yml)|
|-c, --consul-address=                |The address of the Consul server.|
|    --consul-template-be-path=          |The path to the Consul Template representing snippet of the backend configuration. If specified, proxy template will be loaded from the specified file.|
|    --consul-template-fe-path=          |The path to the Consul Template representing snippet of the frontend configuration. If specified, proxy template will be loaded from the specified file.|
|-F, --flow=                          |The actions that should be performed as the flow. Multiple values are allowed.<br>**deploy**: Deploys a new release<br>**scale**: Scales currently running release<br>**stop-old**: Stops the old release<br>**proxy**: Reconfigures the proxy<br>**test:[TARGET]**: Runs a test target specified through the test-docker-compose argument<br>(default: [deploy]) (**multi**)|
|-h, --help                           |Show this help message|
|-H, --host=                          |Docker daemon socket to connect to. If not specified, DOCKER_HOST environment variable will be used instead.|
|-p, --project=                       |Docker Compose project. If not specified, the current directory will be used instead.|
|    --proxy-docker-cert-path=        |Docker certification path for the proxy host.|
|    --proxy-docker-host=             |Docker daemon socket of the proxy host. This argument is required only if the proxy flow step is used.|
|    --proxy-host=                    |The host of the proxy. Visitors should request services from this domain. Docker Flow uses it to request reconfiguration when a new service is deployed or an existing one is scaled. This argument is required only if the proxy flow step is used.|
|    --proxy-reconf-port=             |The port used by the proxy to reconfigure its configuration|
|-S, --pull-side-targets              |Pull side or auxiliary targets. (**bool**)|
|-s, --scale=                         |Number of instances to deploy. If the value starts with the plus sign (+), the number of instances will be increased by the given number. If the value begins with the minus sign (-), the number of instances will be decreased by the given number.|
|    --service-path=                  |Path that should be configured in the proxy (e.g. /demo). This argument is required only if the proxy flow step is used. (**multi**)|
|-T, --side-target=                   |Side or auxiliary Docker Compose targets. Multiple values are allowed. (default: [db]) (**multi**)|
|-t, --target=                        |Docker Compose target. (default: app)|
|--test-compose-path=                 |Path to the Docker Compose configuration file used for tests. (default: docker-compose.yml)|

### Mappings from command line arguments to YML and environment variables

|Command argument                     |YML                    |Environment variable        |
|-------------------------------------|-----------------------|----------------------------|
|-b, --blue-green                     |blue_green             |FLOW_BLUE_GREEN             |
|    --cert-path=                     |cert_path              |FLOW_CERT_PATH              |
|-f, --compose-path=docker-compose.yml|compose_path           |FLOW_COMPOSE_PATH           |
|-c, --consul-address=                |consul_address         |FLOW_CONSUL_ADDRESS         |
|    --consul-template-be-path=       |consul_template_be_path|FLOW_CONSUL_TEMPLATE_BE_PATH|
|    --consul-template-fe-path=       |consul_template_fe_path|FLOW_CONSUL_TEMPLATE_FE_PATH|
|-F, --flow=                          |flow                   |FLOW                        |
|-H, --host=                          |host                   |FLOW_HOST or DOCKER_HOST    |
|-p, --project=                       |project                |FLOW_PROJECT                |
|    --proxy-docker-cert-path=        |proxy_docker_cert_path |FLOW_PROXY_DOCKER_CERT_PATH |
|    --proxy-docker-host=             |proxy_docker_host      |FLOW_PROXY_DOCKER_HOST      |
|    --proxy-host=                    |proxy_host             |FLOW_PROXY_HOST             |
|    --proxy-reconf-port=             |proxy_reconf_port      |FLOW_PROXY_RECONF_PORT      |
|-S, --pull-side-targets              |pull_side_targets      |FLOW_PULL_SIDE_TARGETS      |
|-s, --scale=                         |scale                  |SCALE                       |
|    --service-path=                  |service_path           |FLOW_SERVICE_PATH           |
|-T, --side-target=                   |side_targets           |FLOW_SIDE_TARGETS           |
|-t, --target=                        |target                 |FLOW_TARGET                 |
|--test-compose-path                  |test_compose_path      |FLOW_TEST_COMPOSE_PATH      |


Arguments can be strings, boolean, or multiple values. Command line arguments of boolean type do not have any value (i.e. *--blue-green*). Environment variables and YML arguments of boolean type should use *true* as value (i.e. *FLOW_BLUE_GREEN=true* and *blue_green: true*). When allowed, multiple values can be specified by repeating the command line argument (e.g. *--flow=deploy --flow=stop-old*). When specified through environment variables, multiple values should be separated with comma (e.g. *FLOW=deploy,stop-old*). YML accepts multiple values through the standard format.

```yml
flow:
  - deploy
  - stop-old
```

Feedback and Contribution
-------------------------

I'd appreciate any feedback you might give (both positive and negative). Feel fee to [create a new issue](https://github.com/vfarcic/docker-flow/issues), send a pull request, or tell me about any feature you might be missing. You can find my contact information in the [About](http://technologyconversations.com/about/) section of my [blog](http://technologyconversations.com/).

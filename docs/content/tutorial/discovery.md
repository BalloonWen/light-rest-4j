---
date: 2017-01-27T20:57:14-05:00
title: Registry and Discovery
---

# Introduction

This is a tutorial to show you how to use service registry and discovery
for microservices. The example services are implemented in RESTful style but
they can be implmented in graphql or hybrid as well. We are going to use 
api_a, api_b, api_c and api_d as our examples. To simply the tutorial, I am 
going to disable the security all the time. There are some details that might
not be shown in this tutorial, for example, walking through light-codegen config
files etc. It is recommended to go through [ms-chain](ms-chain/) before this
tutorial. 

The specifications for above APIs can be found at 
https://github.com/networknt/model-config rest folder.

# Preparation

In order to follow the steps below, please make sure you have the same 
working environment.

* A computer with MacOS or Linux (Windows should work but I never tried)
* Install git
* Install Docker
* Install JDK 8 and Maven
* Install Java IDE (Intellij IDEA Community Edition is recommended)
* Create a working directory under your user directory called networknt.

```
cd ~
mkdir networknt
```

# Clone the specifications

In order to generate the initial projects, we need to call light-codegen
and we need the specifications for these services.

```
cd ~/networknt
git clone git@github.com:networknt/model-config.git
```

In this repo, you have a generate.sh in the root folder to use docker
container of light-codegen to generate the code and there are api_a,
api_b, api_c and api_d in rest folder for swagger.json files and config.json
files for each API.

# Code generation

We are going to generate the code into light-example-4j repo, so let's
clone this repo into our working directory.

```
cd ~/networknt
git clone git@github.com:networknt/light-example-4j.git
```

In the above repo, there is a folder discovery contains all the projects
for this tutorial. In order to start from scratch, let's change the existing
folder to discovery.bak as a backup so that you can compare if your code is
not working in each step.

```
cd ~/networknt/light-example-4j
mv discovery discovery.bak
```

In order to generate the four projects, let's clone and build the light-codegen 
into the workspace. 

```
cd ~/networknt
git clone git@github.com:networknt/light-codegen.git
mvn clean install
```

Now let's generate the four APIs.

```
cd ~/networknt/light-codegen
java -jar codegen-cli/target/codegen-cli.jar -f light-rest-4j -o ../light-example-4j/discovery/api_a/generated -m ../model-config/rest/api_a/1.0.0/swagger.json -c ../model-config/rest/api_a/1.0.0/config.json
java -jar codegen-cli/target/codegen-cli.jar -f light-rest-4j -o ../light-example-4j/discovery/api_b/generated -m ../model-config/rest/api_b/1.0.0/swagger.json -c ../model-config/rest/api_b/1.0.0/config.json
java -jar codegen-cli/target/codegen-cli.jar -f light-rest-4j -o ../light-example-4j/discovery/api_c/generated -m ../model-config/rest/api_c/1.0.0/swagger.json -c ../model-config/rest/api_c/1.0.0/config.json
java -jar codegen-cli/target/codegen-cli.jar -f light-rest-4j -o ../light-example-4j/discovery/api_d/generated -m ../model-config/rest/api_d/1.0.0/swagger.json -c ../model-config/rest/api_d/1.0.0/config.json

```

We have four projects generated and compiled under generated folder under each 
project folder. 

# Test generated code

Now you can test the generated projects to make sure they are working with mock
data. We will pick up one project to test it but you can test them all.

```
cd ~/networknt/light-example-4j/discovery/api_a/generated
mvn clean install exec:exec
```

From another terminal, access the server with curl command and check the result.

```
curl http://localhost:7001/v1/data
["Message 1","Message 2"]
```

Based on the config files for each project, the generated service will listen to
7001, 7002, 7003 and 7004 for http connections. You can start other services to test
them on their corresponding port.


# Static

Now we have four standalone services and the next step is to connect them together.

Here is the call tree for these services.

API A will call API B and API C to fulfill its request. API B will call API D
to fulfill its request.

```
API A -> API B -> API D
      -> API C
```

Before we change the code, let's copy the generated projects to new folders so
that we can compare the changes later on.

```
cd ~/networknt/light-example-4j/discovery/api_a
cp -r generated static
cd ~/networknt/light-example-4j/discovery/api_b
cp -r generated static
cd ~/networknt/light-example-4j/discovery/api_c
cp -r generated static
cd ~/networknt/light-example-4j/discovery/api_d
cp -r generated static
```

Let's start update the code in static folders for each project. If you are
using Intellij IDEA Community Edition, you need to open light-example-4j
repo and then import each project by right click pom.xml in each static folder.

As indicated from the title, here we are going to hard code urls in API to API
calls in configuration files. That means these services will be deployed on the 
known hosts with known ports. And we will have a config file for each project to 
define the calling service urls.

### API A

For API A, as it is calling API B and API C, its handler needs to be changed to
calling two other APIs and needs to load a configuration file that define the 
urls for API B and API C.

DataGetHandler.java

```
package com.networknt.apia.handler;

import com.fasterxml.jackson.core.type.TypeReference;
import com.networknt.client.Client;
import com.networknt.config.Config;
import com.networknt.exception.ClientException;
import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;
import io.undertow.util.HttpString;
import org.apache.http.HttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.concurrent.FutureCallback;
import org.apache.http.impl.nio.client.CloseableHttpAsyncClient;

import java.io.IOException;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Vector;
import java.util.concurrent.CountDownLatch;

public class DataGetHandler implements HttpHandler {
    static String CONFIG_NAME = "api_a";
    static String apibUrl = (String)Config.getInstance().getJsonMapConfig(CONFIG_NAME).get("api_b_endpoint");
    static String apicUrl = (String) Config.getInstance().getJsonMapConfig(CONFIG_NAME).get("api_c_endpoint");

    @Override
    public void handleRequest(HttpServerExchange exchange) throws Exception {
        List<String> list = new Vector<String>();
        final HttpGet[] requests = new HttpGet[] {
                new HttpGet(apibUrl),
                new HttpGet(apicUrl),
        };
        try {
            CloseableHttpAsyncClient client = Client.getInstance().getAsyncClient();
            final CountDownLatch latch = new CountDownLatch(requests.length);
            for (final HttpGet request: requests) {
                //Client.getInstance().propagateHeaders(request, exchange);
                client.execute(request, new FutureCallback<HttpResponse>() {
                    @Override
                    public void completed(final HttpResponse response) {
                        try {
                            List<String> apiList = Config.getInstance().getMapper().readValue(response.getEntity().getContent(),
                                    new TypeReference<List<String>>(){});
                            list.addAll(apiList);
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                        latch.countDown();
                    }

                    @Override
                    public void failed(final Exception ex) {
                        ex.printStackTrace();
                        latch.countDown();
                    }

                    @Override
                    public void cancelled() {
                        System.out.println("cancelled");
                        latch.countDown();
                    }
                });
            }
            latch.await();
        } catch (ClientException e) {
            e.printStackTrace();
            throw new Exception("ClientException:", e);
        }
        // now add API A specific messages
        list.add("API A: Message 1");
        list.add("API A: Message 2");
        exchange.getResponseSender().send(Config.getInstance().getMapper().writeValueAsString(list));
    }
}

```

The following is the config file that define the url for API B and API C. This is hard
coded and can only be changed in this config file and restart the server. For now, I
am just changing the file in src/main/resources/config folder, but it should be 
externalized on official environment.

api_a.yml

```
api_b_endpoint: http://localhost:7002/v1/data
api_c_endpoint: http://localhost:7003/v1/data

```

### API B

Change the handler to call API D and load configuration for API D url.

DataGetHandler.java

```
package com.networknt.apib.handler;

import com.fasterxml.jackson.core.type.TypeReference;
import com.networknt.client.Client;
import com.networknt.config.Config;
import com.networknt.exception.ClientException;
import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;
import io.undertow.util.HttpString;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;

import java.io.IOException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class DataGetHandler implements HttpHandler {
    static String CONFIG_NAME = "api_b";
    static String apidUrl = (String) Config.getInstance().getJsonMapConfig(CONFIG_NAME).get("api_d_endpoint");

    @Override
    public void handleRequest(HttpServerExchange exchange) throws Exception {
        List<String> list = new ArrayList<String>();
        try {
            CloseableHttpClient client = Client.getInstance().getSyncClient();
            HttpGet httpGet = new HttpGet(apidUrl);
            //Client.getInstance().propagateHeaders(httpGet, exchange);
            CloseableHttpResponse response = client.execute(httpGet);
            int responseCode = response.getStatusLine().getStatusCode();
            if(responseCode != 200){
                throw new Exception("Failed to call API D: " + responseCode);
            }
            List<String> apidList = Config.getInstance().getMapper().readValue(response.getEntity().getContent(),
                    new TypeReference<List<String>>(){});
            list.addAll(apidList);
        } catch (ClientException e) {
            throw new Exception("Client Exception: ", e);
        } catch (IOException e) {
            throw new Exception("IOException:", e);
        }
        // now add API B specific messages
        list.add("API B: Message 1");
        list.add("API B: Message 2");
        exchange.getResponseSender().send(Config.getInstance().getMapper().writeValueAsString(list));
    }
}

```

Configuration file for API D url.

api_b.yml

```
api_d_endpoint: http://localhost:7004/v1/data

```

### API C


Update API C handler to return information that associates with API C.

DataGetHandler.java

```
package com.networknt.apic.handler;

import com.networknt.config.Config;
import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;
import io.undertow.util.HttpString;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class DataGetHandler implements HttpHandler {
    @Override
    public void handleRequest(HttpServerExchange exchange) throws Exception {
        List<String> messages = new ArrayList<String>();
        messages.add("API C: Message 1");
        messages.add("API C: Message 2");
        exchange.getResponseSender().send(Config.getInstance().getMapper().writeValueAsString(messages));
    }
}

```


### API D

Update Handler for API D to return messages related to API D.

DataGetHandler.java

```
package com.networknt.apid.handler;

import com.networknt.config.Config;
import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;
import io.undertow.util.HttpString;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class DataGetHandler implements HttpHandler {
    @Override
    public void handleRequest(HttpServerExchange exchange) throws Exception {
        List<String> messages = new ArrayList<String>();
        messages.add("API D: Message 1");
        messages.add("API D: Message 2");
        exchange.getResponseSender().send(Config.getInstance().getMapper().writeValueAsString(messages));
    }
}

```


### Start Servers

Now let's start all four servers from four terminals.

API A

```
cd ~/networknt/light-example-4j/discovery/api_a/static
mvn clean install exec:exec
```

API B

```
cd ~/networknt/light-example-4j/discovery/api_b/static
mvn clean install exec:exec

```

API C

```
cd ~/networknt/light-example-4j/discovery/api_c/static
mvn clean install exec:exec

```

API D

```
cd ~/networknt/light-example-4j/discovery/api_d/static
mvn clean install exec:exec

```

### Test Servers

Let's access API A and see if we can get messages from all four servers.

```
curl http://localhost:7001/v1/data

```
The result is 

```
["API C: Message 1","API C: Message 2","API D: Message 1","API D: Message 2","API B: Message 1","API B: Message 2","API A: Message 1","API A: Message 2"]
```


# Dynamic

The above step uses static urls defined in configuration files. It won't work in a
dynamic clustered environment as there are more instances of each service. In this
step, we are going to use cluster component with direct registry so that we don't
need to start external consul or zookeeper instances. We still go through registry
for service discovery but the registry is defined in service.yml. Next step we
will use consul server for the discovery to mimic real production environment.


First let's create a folder from static to dynamic.

```
cd ~/networknt/light-example-4j/discovery/api_a
cp -r static dynamic
cd ~/networknt/light-example-4j/discovery/api_b
cp -r static dynamic
cd ~/networknt/light-example-4j/discovery/api_c
cp -r static dynamic
cd ~/networknt/light-example-4j/discovery/api_d
cp -r static dynamic
```


### API A

Let's update API A Handler to use Cluster instance instead of using static config
files. 

DataGetHandler.java

```
package com.networknt.apia.handler;

import com.fasterxml.jackson.core.type.TypeReference;
import com.networknt.client.Client;
import com.networknt.cluster.Cluster;
import com.networknt.config.Config;
import com.networknt.exception.ClientException;
import com.networknt.service.SingletonServiceFactory;
import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;
import org.apache.http.HttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.concurrent.FutureCallback;
import org.apache.http.impl.nio.client.CloseableHttpAsyncClient;

import java.io.IOException;
import java.util.List;
import java.util.Vector;
import java.util.concurrent.CountDownLatch;

public class DataGetHandler implements HttpHandler {
    private static Cluster cluster = (Cluster) SingletonServiceFactory.getBean(Cluster.class);

    @Override
    public void handleRequest(HttpServerExchange exchange) throws Exception {
        List<String> list = new Vector<String>();

        String apibUrl = cluster.serviceToUrl("http", "com.networknt.apib-1.0.0", null) + "/v1/data";
        String apicUrl = cluster.serviceToUrl("http", "com.networknt.apic-1.0.0", null) + "/v1/data";

        final HttpGet[] requests = new HttpGet[] {
                new HttpGet(apibUrl),
                new HttpGet(apicUrl),
        };
        try {
            CloseableHttpAsyncClient client = Client.getInstance().getAsyncClient();
            final CountDownLatch latch = new CountDownLatch(requests.length);
            for (final HttpGet request: requests) {
                //Client.getInstance().propagateHeaders(request, exchange);
                client.execute(request, new FutureCallback<HttpResponse>() {
                    @Override
                    public void completed(final HttpResponse response) {
                        try {
                            List<String> apiList = Config.getInstance().getMapper().readValue(response.getEntity().getContent(),
                                    new TypeReference<List<String>>(){});
                            list.addAll(apiList);
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                        latch.countDown();
                    }

                    @Override
                    public void failed(final Exception ex) {
                        ex.printStackTrace();
                        latch.countDown();
                    }

                    @Override
                    public void cancelled() {
                        System.out.println("cancelled");
                        latch.countDown();
                    }
                });
            }
            latch.await();
        } catch (ClientException e) {
            e.printStackTrace();
            throw new Exception("ClientException:", e);
        }
        // now add API A specific messages
        list.add("API A: Message 1");
        list.add("API A: Message 2");
        exchange.getResponseSender().send(Config.getInstance().getMapper().writeValueAsString(list));
    }
}

```

Also, we need service.yml to inject several singleton implementations of
Cluster, LoadBanlance, URL and Registry. Please note that the key in parameters
is serviceId of your calling APIs


```
singletons:
- com.networknt.registry.URL:
  - com.networknt.registry.URLImpl:
      protocol: https
      host: localhost
      port: 8080
      path: direct
      parameters:
        com.networknt.apib-1.0.0: http://localhost:7002
        com.networknt.apic-1.0.0: http://localhost:7003
- com.networknt.registry.Registry:
  - com.networknt.registry.support.DirectRegistry
- com.networknt.balance.LoadBalance:
  - com.networknt.balance.RoundRobinLoadBalance
- com.networknt.cluster.Cluster:
  - com.networknt.cluster.LightCluster

```


### API B

DataGetHandler.java

```
package com.networknt.apib.handler;

import com.fasterxml.jackson.core.type.TypeReference;
import com.networknt.client.Client;
import com.networknt.cluster.Cluster;
import com.networknt.config.Config;
import com.networknt.exception.ClientException;
import com.networknt.service.SingletonServiceFactory;
import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class DataGetHandler implements HttpHandler {
    private static Cluster cluster = (Cluster) SingletonServiceFactory.getBean(Cluster.class);

    @Override
    public void handleRequest(HttpServerExchange exchange) throws Exception {
        List<String> list = new ArrayList<String>();
        String apidUrl = cluster.serviceToUrl("http", "com.networknt.apid-1.0.0", null) + "/v1/data";

        try {
            CloseableHttpClient client = Client.getInstance().getSyncClient();
            HttpGet httpGet = new HttpGet(apidUrl);
            //Client.getInstance().propagateHeaders(httpGet, exchange);
            CloseableHttpResponse response = client.execute(httpGet);
            int responseCode = response.getStatusLine().getStatusCode();
            if(responseCode != 200){
                throw new Exception("Failed to call API D: " + responseCode);
            }
            List<String> apidList = Config.getInstance().getMapper().readValue(response.getEntity().getContent(),
                    new TypeReference<List<String>>(){});
            list.addAll(apidList);
        } catch (ClientException e) {
            throw new Exception("Client Exception: ", e);
        } catch (IOException e) {
            throw new Exception("IOException:", e);
        }
        // now add API B specific messages
        list.add("API B: Message 1");
        list.add("API B: Message 2");
        exchange.getResponseSender().send(Config.getInstance().getMapper().writeValueAsString(list));
    }
}

```

Inject interface implementations and define the API D url.

```
singletons:
- com.networknt.registry.URL:
  - com.networknt.registry.URLImpl:
      protocol: https
      host: localhost
      port: 8080
      path: direct
      parameters:
        com.networknt.apid-1.0.0: http://localhost:7004
- com.networknt.registry.Registry:
  - com.networknt.registry.support.DirectRegistry
- com.networknt.balance.LoadBalance:
  - com.networknt.balance.RoundRobinLoadBalance
- com.networknt.cluster.Cluster:
  - com.networknt.cluster.LightCluster

```


### API C

API C is not calling any other APIs, so there is no change to its handler.




### API D

API D is not calling any other APIs, so there is no change to its handler.


### Start Servers

Now let's start all four servers from four terminals.

API A

```
cd ~/networknt/light-example-4j/discovery/api_a/dynamic
mvn clean install exec:exec
```

API B

```
cd ~/networknt/light-example-4j/discovery/api_b/dynamic
mvn clean install exec:exec

```

API C

```
cd ~/networknt/light-example-4j/discovery/api_c/dynamic
mvn clean install exec:exec

```

API D

```
cd ~/networknt/light-example-4j/discovery/api_d/dynamic
mvn clean install exec:exec

```

### Test Servers

Let's access API A and see if we can get messages from all four servers.

```
curl http://localhost:7001/v1/data

```
The result is 

```
["API C: Message 1","API C: Message 2","API D: Message 1","API D: Message 2","API B: Message 1","API B: Message 2","API A: Message 1","API A: Message 2"]
```

# Multiple API D Instances

In this step, we are going to start two API D instances that listening to 7004 and 7005.

Now let's copy from dynamic to multiple for each API.
 

```
cd ~/networknt/light-example-4j/discovery/api_a
cp -r dynamic multiple
cd ~/networknt/light-example-4j/discovery/api_b
cp -r dynamic multiple
cd ~/networknt/light-example-4j/discovery/api_c
cp -r dynamic multiple
cd ~/networknt/light-example-4j/discovery/api_d
cp -r dynamic multiple
```

 
### API B 

Let's modify API B service.yml to have two API D instances that listen to 7004
and 7005. 

service.yml

```
singletons:
- com.networknt.registry.URL:
  - com.networknt.registry.URLImpl:
      protocol: https
      host: localhost
      port: 8080
      path: direct
      parameters:
        com.networknt.apid-1.0.0: http://localhost:7004,http://localhost:7005
- com.networknt.registry.Registry:
  - com.networknt.registry.support.DirectRegistry
- com.networknt.balance.LoadBalance:
  - com.networknt.balance.RoundRobinLoadBalance
- com.networknt.cluster.Cluster:
  - com.networknt.cluster.LightCluster

```

### API D

In order to start two instances with the same code base, we need to modify the
server.yml before starting the server. 

Also, let's update the handler so that we know which port serves the request.

DataGetHandler.java

```
package com.networknt.apid.handler;

import com.networknt.config.Config;
import com.networknt.server.Server;
import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;

import java.util.ArrayList;
import java.util.List;

public class DataGetHandler implements HttpHandler {
    @Override
    public void handleRequest(HttpServerExchange exchange) throws Exception {
        int port = Server.config.getHttpPort();
        List<String> messages = new ArrayList<String>();
        messages.add("API D: Message 1 from port " + port);
        messages.add("API D: Message 2 from port " + port);
        exchange.getResponseSender().send(Config.getInstance().getMapper().writeValueAsString(messages));
    }
}

```

### Start Servers
Now let's start all five servers from five terminals. API D has two instances.

API A

```
cd ~/networknt/light-example-4j/discovery/api_a/multiple
mvn clean install exec:exec
```

API B

```
cd ~/networknt/light-example-4j/discovery/api_b/multiple
mvn clean install exec:exec

```

API C

```
cd ~/networknt/light-example-4j/discovery/api_c/multiple
mvn clean install exec:exec

```

API D


And start the first instance that listen to 7004.

```
cd ~/networknt/light-example-4j/discovery/api_d/multiple
mvn clean install exec:exec

```
 
Now let's start the second instance. Before starting the serer, let's update
server.yml with port 7005.

```

# Server configuration
---
# This is the default binding address if the service is dockerized.
ip: 0.0.0.0

# Http port if enableHttp is true.
httpPort:  7005

# Enable HTTP should be false on official environment.
enableHttp: true

# Https port if enableHttps is true.
httpsPort:  7445

# Enable HTTPS should be true on official environment.
enableHttps: true

# Keystore file name in config folder. KeystorePass is in secret.yml to access it.
keystoreName: tls/server.keystore

# Flag that indicate if two way TLS is enabled. Not recommended in docker container.
enableTwoWayTls: false

# Truststore file name in config folder. TruststorePass is in secret.yml to access it.
truststoreName: tls/server.truststore

# Unique service identifier. Used in service registration and discovery etc.
serviceId: com.networknt.apid-1.0.0

# Flag to enable service registration. Only be true if running as standalone Java jar.
enableRegistry: false

```

And start the second instance

```
cd ~/networknt/light-example-4j/discovery/api_d/multiple
mvn clean install exec:exec

```

### Test Servers

```
curl http://localhost:7001/v1/data
```

And the result can be the following alternatively as two instances of API D are load balanced.

```
["API C: Message 1","API C: Message 2","API D: Message 1 from port 7004","API D: Message 2 from port 7004","API B: Message 1","API B: Message 2","API A: Message 1","API A: Message 2"]
```

or 

```
["API C: Message 1","API C: Message 2","API D: Message 1 from port 7005","API D: Message 2 from port 7005","API B: Message 1","API B: Message 2","API A: Message 1","API A: Message 2"]
```

# Consul

Above step multiple demonstrates how to use direct registry to enable load balance and
it works the same way as Consul and Zookeeper registry. In this step, we are going to
use Consul for registry to enable cluster.

Now let's copy from multiple to consul for each API.
 

```
cd ~/networknt/light-example-4j/discovery/api_a
cp -r multiple consul
cd ~/networknt/light-example-4j/discovery/api_b
cp -r multiple consul
cd ~/networknt/light-example-4j/discovery/api_c
cp -r multiple consul
cd ~/networknt/light-example-4j/discovery/api_d
cp -r multiple consul
```


### API A

In order to switch from direct registry to consul registry, we just need to update
service.yml configuration to inject the consul implementation to the registry interface.

service.yml

```
singletons:
- com.networknt.registry.URL:
  - com.networknt.registry.URLImpl:
      protocol: light
      host: localhost
      port: 8080
      path: consul
      parameters:
        registryRetryPeriod: '30000'
- com.networknt.consul.client.ConsulClient:
  - com.networknt.consul.client.ConsulEcwidClient:
    - java.lang.String: localhost
    - int: 8500
- com.networknt.registry.Registry:
  - com.networknt.consul.ConsulRegistry
- com.networknt.balance.LoadBalance:
  - com.networknt.balance.RoundRobinLoadBalance
- com.networknt.cluster.Cluster:
  - com.networknt.cluster.LightCluster
```

Although in our case, there is no caller service for API A, we still need to register
it to consul by enable it in server.yml. Note that enableRegsitry is turn on and enableHttps
is turned off. 

If you don't turn off the https port, both ports will be registered on Consul. 

```

# Server configuration
---
# This is the default binding address if the service is dockerized.
ip: 0.0.0.0

# Http port if enableHttp is true.
httpPort:  7001

# Enable HTTP should be false on official environment.
enableHttp: true

# Https port if enableHttps is true.
httpsPort:  7441

# Enable HTTPS should be true on official environment.
enableHttps: false

# Keystore file name in config folder. KeystorePass is in secret.yml to access it.
keystoreName: tls/server.keystore

# Flag that indicate if two way TLS is enabled. Not recommended in docker container.
enableTwoWayTls: false

# Truststore file name in config folder. TruststorePass is in secret.yml to access it.
truststoreName: tls/server.truststore

# Unique service identifier. Used in service registration and discovery etc.
serviceId: com.networknt.apia-1.0.0

# Flag to enable service registration. Only be true if running as standalone Java jar.
enableRegistry: true

```


### API B

Let's update service.yml to inject consul registry instead of direct registry used in
the previous step.

service.yml

```
singletons:
- com.networknt.registry.URL:
  - com.networknt.registry.URLImpl:
      protocol: light
      host: localhost
      port: 8080
      path: consul
      parameters:
        registryRetryPeriod: '30000'
- com.networknt.consul.client.ConsulClient:
  - com.networknt.consul.client.ConsulEcwidClient:
    - java.lang.String: localhost
    - int: 8500
- com.networknt.registry.Registry:
  - com.networknt.consul.ConsulRegistry
- com.networknt.balance.LoadBalance:
  - com.networknt.balance.RoundRobinLoadBalance
- com.networknt.cluster.Cluster:
  - com.networknt.cluster.LightCluster
```

As API B will be called by API A, it needs to register itself to consul registry so
that API A can discover it through the same consul registry. To do that you only need
to enable server registry in config file.

server.yml

```

# Server configuration
---
# This is the default binding address if the service is dockerized.
ip: 0.0.0.0

# Http port if enableHttp is true.
httpPort:  7002

# Enable HTTP should be false on official environment.
enableHttp: true

# Https port if enableHttps is true.
httpsPort:  7442

# Enable HTTPS should be true on official environment.
enableHttps: false

# Keystore file name in config folder. KeystorePass is in secret.yml to access it.
keystoreName: tls/server.keystore

# Flag that indicate if two way TLS is enabled. Not recommended in docker container.
enableTwoWayTls: false

# Truststore file name in config folder. TruststorePass is in secret.yml to access it.
truststoreName: tls/server.truststore

# Unique service identifier. Used in service registration and discovery etc.
serviceId: com.networknt.apib-1.0.0

# Flag to enable service registration. Only be true if running as standalone Java jar.
enableRegistry: true
```

### API C

Although API C is not calling any other APIs, it needs to register itself to consul
so that API A can discovery it from the same consul registry.

service.yml

```
singletons:
- com.networknt.registry.URL:
  - com.networknt.registry.URLImpl:
      protocol: light
      host: localhost
      port: 8080
      path: consul
      parameters:
        registryRetryPeriod: '30000'
- com.networknt.consul.client.ConsulClient:
  - com.networknt.consul.client.ConsulEcwidClient:
    - java.lang.String: localhost
    - int: 8500
- com.networknt.registry.Registry:
  - com.networknt.consul.ConsulRegistry
- com.networknt.balance.LoadBalance:
  - com.networknt.balance.RoundRobinLoadBalance
- com.networknt.cluster.Cluster:
  - com.networknt.cluster.LightCluster
```


server.yml

```
# Server configuration
---
# This is the default binding address if the service is dockerized.
ip: 0.0.0.0

# Http port if enableHttp is true.
httpPort:  7003

# Enable HTTP should be false on official environment.
enableHttp: true

# Https port if enableHttps is true.
httpsPort:  7443

# Enable HTTPS should be true on official environment.
enableHttps: false

# Keystore file name in config folder. KeystorePass is in secret.yml to access it.
keystoreName: tls/server.keystore

# Flag that indicate if two way TLS is enabled. Not recommended in docker container.
enableTwoWayTls: false

# Truststore file name in config folder. TruststorePass is in secret.yml to access it.
truststoreName: tls/server.truststore

# Unique service identifier. Used in service registration and discovery etc.
serviceId: com.networknt.apic-1.0.0

# Flag to enable service registration. Only be true if running as standalone Java jar.
enableRegistry: true
```

### API D

Although API D is not calling any other APIs, it needs to register itself to consul
so that API B can discovery it from the same consul registry.

service.yml

```
singletons:
- com.networknt.registry.URL:
  - com.networknt.registry.URLImpl:
      protocol: light
      host: localhost
      port: 8080
      path: consul
      parameters:
        registryRetryPeriod: '30000'
- com.networknt.consul.client.ConsulClient:
  - com.networknt.consul.client.ConsulEcwidClient:
    - java.lang.String: localhost
    - int: 8500
- com.networknt.registry.Registry:
  - com.networknt.consul.ConsulRegistry
- com.networknt.balance.LoadBalance:
  - com.networknt.balance.RoundRobinLoadBalance
- com.networknt.cluster.Cluster:
  - com.networknt.cluster.LightCluster
```

server.yml

```
# Server configuration
---
# This is the default binding address if the service is dockerized.
ip: 0.0.0.0

# Http port if enableHttp is true.
httpPort:  7004

# Enable HTTP should be false on official environment.
enableHttp: true

# Https port if enableHttps is true.
httpsPort:  7444

# Enable HTTPS should be true on official environment.
enableHttps: false

# Keystore file name in config folder. KeystorePass is in secret.yml to access it.
keystoreName: tls/server.keystore

# Flag that indicate if two way TLS is enabled. Not recommended in docker container.
enableTwoWayTls: false

# Truststore file name in config folder. TruststorePass is in secret.yml to access it.
truststoreName: tls/server.truststore

# Unique service identifier. Used in service registration and discovery etc.
serviceId: com.networknt.apid-1.0.0

# Flag to enable service registration. Only be true if running as standalone Java jar.
enableRegistry: true

```


### Start Consul

Here we are starting consul server in docker to serve as a centralized registry. To make it
simpler, the ACL in consul is disable by setting acl_default_policy=allow.

```
docker run -d -p 8400:8400 -p 8500:8500/tcp -p 8600:53/udp -e 'CONSUL_LOCAL_CONFIG={"acl_datacenter":"dc1","acl_default_policy":"allow","acl_down_policy":"extend-cache","acl_master_token":"the_one_ring","bootstrap_expect":1,"datacenter":"dc1","data_dir":"/usr/local/bin/consul.d/data","server":true}' consul agent -server -ui -bind=127.0.0.1 -client=0.0.0.0
```


### Start four servers

Now let's start four terminals to start servers.  

API A

```
cd ~/networknt/light-example-4j/discovery/api_a/consul
mvn clean install -DskipTests
mvn exec:exec
```

API B

```
cd ~/networknt/light-example-4j/discovery/api_b/consul
mvn clean install -DskipTests
mvn exec:exec

```

API C

```
cd ~/networknt/light-example-4j/discovery/api_c/consul
mvn clean install -DskipTests 
mvn exec:exec

```

API D


And start the first instance that listen to 7004 as default

```
cd ~/networknt/light-example-4j/discovery/api_d/consul
mvn clean install -DskipTests 
mvn exec:exec

```

Now you can see the registered service from Consul UI.

```
http://localhost:8500/ui
```


### Test four servers

```
curl http://localhost:7001/v1/data
```

And the result will be

```
["API C: Message 1","API C: Message 2","API D: Message 1 from port 7004","API D: Message 2 from port 7004","API B: Message 1","API B: Message 2","API A: Message 1","API A: Message 2"]
```
 
### Start another API D
 
Now let's start the second instance of API D. Before starting the serer, let's update
server.yml with port 7005.

```

# Server configuration
---
# This is the default binding address if the service is dockerized.
ip: 0.0.0.0

# Http port if enableHttp is true.
httpPort:  7005

# Enable HTTP should be false on official environment.
enableHttp: true

# Https port if enableHttps is true.
httpsPort:  7444

# Enable HTTPS should be true on official environment.
enableHttps: false

# Keystore file name in config folder. KeystorePass is in secret.yml to access it.
keystoreName: tls/server.keystore

# Flag that indicate if two way TLS is enabled. Not recommended in docker container.
enableTwoWayTls: false

# Truststore file name in config folder. TruststorePass is in secret.yml to access it.
truststoreName: tls/server.truststore

# Unique service identifier. Used in service registration and discovery etc.
serviceId: com.networknt.apid-1.0.0

# Flag to enable service registration. Only be true if running as standalone Java jar.
enableRegistry: true

```

And start the second instance

```
cd ~/networknt/light-example-4j/discovery/api_d/consul
mvn clean install -DskipTests 
mvn exec:exec

```

### Test Servers

Wait 10 seconds, your API B cached API D service urls will be updated automatically
with the new instance. Now you have to instance of API D to serve API B.

```
curl http://localhost:7001/v1/data
```

And the result can be the following alternatively.

```
["API C: Message 1","API C: Message 2","API D: Message 1 from port 7004","API D: Message 2 from port 7004","API B: Message 1","API B: Message 2","API A: Message 1","API A: Message 2"]
```

or 

```
["API C: Message 1","API C: Message 2","API D: Message 1 from port 7005","API D: Message 2 from port 7005","API B: Message 1","API B: Message 2","API A: Message 1","API A: Message 2"]
```


### Shutdown one API D

Let's shutdown one instance of API D and wait for 10 seconds. Now when you call the same
curl command, API D message will be always served by the same port which is the one still
running.


# Docker

In this step, we are going to dockerize all the APIs and then use registrator for service
registry. 

Now let's copy from consul to docker for each API.
 

```
cd ~/networknt/light-example-4j/discovery/api_a
cp -r consul consuldocker
cd ~/networknt/light-example-4j/discovery/api_b
cp -r consul consuldocker
cd ~/networknt/light-example-4j/discovery/api_c
cp -r consul consuldocker
cd ~/networknt/light-example-4j/discovery/api_d
cp -r consul consuldocker
```

Before starting the services, let's start consul and registrator.

Consul

```
docker run -d -p 8400:8400 -p 8500:8500/tcp -p 8600:53/udp -e 'CONSUL_LOCAL_CONFIG={"acl_datacenter":"dc1","acl_default_policy":"allow","acl_down_policy":"extend-cache","acl_master_token":"the_one_ring","bootstrap_expect":1,"datacenter":"dc1","data_dir":"/usr/local/bin/consul.d/data","server":true}' consul agent -server -ui -bind=127.0.0.1 -client=0.0.0.0
```

Regsitrator

We use -ip 127.0.0.1 in the command line to make sure that ServiceAddress in
consul is populated with ip and port. The latest version of regsitrator won't
set default ip anymore.

```
docker run -d --name=registrator --net=host --volume=/var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest -ip 127.0.0.1 consul://localhost:8500
```


### API A

Since we are using registrator to register the service, we need to disable the application service registration.

server.yml

```
# Server configuration
---
# This is the default binding address if the service is dockerized.
ip: 0.0.0.0

# Http port if enableHttp is true.
httpPort:  7001

# Enable HTTP should be false on official environment.
enableHttp: true

# Https port if enableHttps is true.
httpsPort:  7441

# Enable HTTPS should be true on official environment.
enableHttps: false

# Keystore file name in config folder. KeystorePass is in secret.yml to access it.
keystoreName: tls/server.keystore

# Flag that indicate if two way TLS is enabled. Not recommended in docker container.
enableTwoWayTls: false

# Truststore file name in config folder. TruststorePass is in secret.yml to access it.
truststoreName: tls/server.truststore

# Unique service identifier. Used in service registration and discovery etc.
serviceId: com.networknt.apia-1.0.0

# Flag to enable service registration. Only be true if running as standalone Java jar.
enableRegistry: false
```


```
cd ~/networknt/light-example-4j/discovery/api_a/consuldocker
mvn clean install
docker build -t networknt/com.networknt.apia-1.0.0 .
docker run -it -p 7001:7001 --net=host --name=com.networknt.apia-1.0.0 networknt/com.networknt.apia-1.0.0
```

### API B

server.yml

```
# Server configuration
---
# This is the default binding address if the service is dockerized.
ip: 0.0.0.0

# Http port if enableHttp is true.
httpPort:  7002

# Enable HTTP should be false on official environment.
enableHttp: true

# Https port if enableHttps is true.
httpsPort:  7442

# Enable HTTPS should be true on official environment.
enableHttps: false

# Keystore file name in config folder. KeystorePass is in secret.yml to access it.
keystoreName: tls/server.keystore

# Flag that indicate if two way TLS is enabled. Not recommended in docker container.
enableTwoWayTls: false

# Truststore file name in config folder. TruststorePass is in secret.yml to access it.
truststoreName: tls/server.truststore

# Unique service identifier. Used in service registration and discovery etc.
serviceId: com.networknt.apib-1.0.0

# Flag to enable service registration. Only be true if running as standalone Java jar.
enableRegistry: false

```

```
cd ~/networknt/light-example-4j/discovery/api_b/consuldocker
mvn clean install
docker build -t networknt/com.networknt.apib-1.0.0 .
docker run -it -p 7002:7002 --net=host --name=com.networknt.apib-1.0.0 networknt/com.networknt.apib-1.0.0

```

### API C

server.yml

```

# Server configuration
---
# This is the default binding address if the service is dockerized.
ip: 0.0.0.0

# Http port if enableHttp is true.
httpPort:  7003

# Enable HTTP should be false on official environment.
enableHttp: true

# Https port if enableHttps is true.
httpsPort:  7443

# Enable HTTPS should be true on official environment.
enableHttps: false

# Keystore file name in config folder. KeystorePass is in secret.yml to access it.
keystoreName: tls/server.keystore

# Flag that indicate if two way TLS is enabled. Not recommended in docker container.
enableTwoWayTls: false

# Truststore file name in config folder. TruststorePass is in secret.yml to access it.
truststoreName: tls/server.truststore

# Unique service identifier. Used in service registration and discovery etc.
serviceId: com.networknt.apic-1.0.0

# Flag to enable service registration. Only be true if running as standalone Java jar.
enableRegistry: false

```

```
cd ~/networknt/light-example-4j/discovery/api_c/consuldocker
mvn clean install
docker build -t networknt/com.networknt.apic-1.0.0 .
docker run -it -p 7003:7003 --net=host --name=com.networknt.apic-1.0.0 networknt/com.networknt.apic-1.0.0

```

### API D

server.yml

```
# Server configuration
---
# This is the default binding address if the service is dockerized.
ip: 0.0.0.0

# Http port if enableHttp is true.
httpPort:  7004

# Enable HTTP should be false on official environment.
enableHttp: true

# Https port if enableHttps is true.
httpsPort:  7444

# Enable HTTPS should be true on official environment.
enableHttps: false

# Keystore file name in config folder. KeystorePass is in secret.yml to access it.
keystoreName: tls/server.keystore

# Flag that indicate if two way TLS is enabled. Not recommended in docker container.
enableTwoWayTls: false

# Truststore file name in config folder. TruststorePass is in secret.yml to access it.
truststoreName: tls/server.truststore

# Unique service identifier. Used in service registration and discovery etc.
serviceId: com.networknt.apid-1.0.0

# Flag to enable service registration. Only be true if running as standalone Java jar.
enableRegistry: false

```

```
cd ~/networknt/light-example-4j/discovery/api_d/consuldocker
mvn clean install
docker build -t networknt/com.networknt.apid-1.0.0 .
docker run -it -p 7004:7004 --net=host --name=com.networknt.apid-1.0.0 networknt/com.networknt.apid-1.0.0

```


### Test Servers

```
curl http://localhost:7001/v1/data
```

And here is the result.

```
["API C: Message 1","API C: Message 2","API D: Message 1 from port 7004","API D: Message 2 from port 7004","API B: Message 1","API B: Message 2","API A: Message 1","API A: Message 2"]
```

# Kubernetes

Kubernetes should be used for production service scheduling with registrator like Docker 
compose.  
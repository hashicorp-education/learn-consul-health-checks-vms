
# Monitor your application health with distributed checks

This is a solution runbook for the scenario deployed.


## Prerequisites

Login to the Bastion Host

```
$ ssh -i certs/id_rsa.pem admin@`terraform output -raw ip_bastion`
#...
admin@bastion:~$
```


### Configure CLI to interact with Consul

Configure your bastion host to communicate with your Consul environment using the two dynamically generated environment variable files.

```shell-session
$ source "/home/admin/assets/scenario/env-scenario.env" && \
  source "/home/admin/assets/scenario/env-consul.env"
```

That will produce no output.

After loading the needed variables, verify you can connect to your Consul 
datacenter.

```shell-session
$ consul members
```

That will produce an output similar to the following.

```plaintext
Node                  Address          Status  Type    Build   Protocol  DC   Partition  Segment
consul-server-0       10.0.4.16:8301   alive   server  1.17.0  2         dc1  default    <all>
hashicups-api-0       10.0.4.119:8301  alive   client  1.17.0  2         dc1  default    <default>
hashicups-api-1       10.0.4.195:8301  alive   client  1.17.0  2         dc1  default    <default>
hashicups-db-0        10.0.4.169:8301  alive   client  1.17.0  2         dc1  default    <default>
hashicups-frontend-0  10.0.4.124:8301  alive   client  1.17.0  2         dc1  default    <default>
hashicups-nginx-0     10.0.4.217:8301  alive   client  1.17.0  2         dc1  default    <default>
```


## Create ACL token for service registration

To register services in a Consul datacenter, when ACLs are enabled, you need a valid token with the proper permissions.

```shell-session
$ consul acl token create \
  -description="SVC HashiCups API token" \
  --format json \
  -service-identity="hashicups-api" | tee /home/admin/assets/scenario/conf/secrets/acl-token-svc-hashicups-api-1.json
```

That will produce an output similar to the following.

```plaintext
{
    "CreateIndex": 91,
    "ModifyIndex": 91,
    "AccessorID": "28320c2d-02a7-470b-1e02-0e987047282c",
    "SecretID": "540d7fa7-bf32-c295-651e-51010f55720b",
    "Description": "SVC HashiCups API token",
    "ServiceIdentities": [
        {
            "ServiceName": "hashicups-api"
        }
    ],
    "Local": false,
    "CreateTime": "2023-11-16T14:12:35.630251214Z",
    "Hash": "RtpgRlE69uEOWjeuaBUG2ezUM37J4OxeUhFaaRJaEac="
}
```


Retrieve the token from the `acl-token-svc-hashicups-api-1.json` file

```shell-session
$ export CONSUL_AGENT_TOKEN=`cat /home/admin/assets/scenario/conf/secrets/acl-token-svc-hashicups-api-1.json | jq -r ".SecretID"`
```

That will produce no output.

The token will be used in the service definition file.

## Understand service configuration for Consul services

Service configuration is composed by two parts, service definitions and health checks definitions.

### Service definition

The service definition requires the following parameters to be defined:
- `name` - the service name to register. Multiple instances of the same service share the same service name.
- `id` - the service id. Multiple instances of the same service require a unambiguous id to be registered. 
- `tags` - [optional] - tags to assign to the service instance. Useful for blue-green deployment, canary deployment or to identify services inside Consul datacenter.
- `port` - the port your service is exposing.
- `token` - a token to be used during service registration. This is the token you created in the previous section.

Below an example configuration for the service definition.


```hcl

service {
  name = "hashicups-api"
  id = "hashicups-api-1"
  tags = [ "inst_1" ]
  port = 8081
  token = "540d7fa7-bf32-c295-651e-51010f55720b"
}
```


Read more on service definitions at [Services configuration reference](https://developer.hashicorp.com/consul/docs/services/configuration/services-configuration-reference)

### Checks definition.

Consul is able to provide distributed monitoring for your services with the use of health checks.

Health checks configurations are nested in the service block. They can be defined using the following parameters:
- `id` - unique string value that specifies an ID for the check.
- `name` - required string value that specifies the name of the check.
- `service_id` - specifies the ID of a service instance to associate with a service check.
- `interval` - specifies how frequently to run the check.
- `timeout` - specifies how long unsuccessful requests take to end with a timeout.

The other parameter required to define the check is the type.

Consul supports multiple check types, but for this tutorial you will use the *TCP* and *HTTP* check types.

A tcp check establishes connections to the specified IPs or hosts. If the check 
successfully establishes a connection, the service status is reported as `success`. 
If the IP or host does not accept the connection, the service status is reported as `critical`.

An example of tcp check for the hashicups-api service, listening on port `8081` is the following:

```hcl

{
  id =  "check-hashicups-api.public",
  name = "hashicups-api.public status check",
  service_id = "hashicups-api-1",
  tcp  = "localhost:8081",
  interval = "5s",
  timeout = "5s"
}
```


HTTP checks send an HTTP request to the specified URL and report the service health based on the HTTP response code.

An example of tcp check for the hashicups-api service, which exposes an `health` endpoint to test service status, is the following:

```hcl

{
  id =  "check-hashicups-api.public.http",
  name = "hashicups-api.public  HTTP status check",
  service_id = "hashicups-api-1",
  http  = "http://localhost:8081/health",
  interval = "5s",
  timeout = "5s"
}
```


## Create service configuration for HashiCups API service

Create service configuration file.

```shell-session
$ tee /home/admin/assets/scenario/conf/hashicups-api-1/svc-hashicups-api.hcl > /dev/null << EOF
## -----------------------------
## svc-hashicups-api.hcl
## -----------------------------
service {
  name = "hashicups-api"
  id = "hashicups-api-1"
  tags = [ "inst_1" ]
  port = 8081
  token = "${CONSUL_AGENT_TOKEN}"

  checks =[
  {
  id =  "check-hashicups-api.public.http",
  name = "hashicups-api.public  HTTP status check",
  service_id = "hashicups-api-1",
  http  = "http://localhost:8081/health",
  interval = "5s",
  timeout = "5s"
  },
  {
    id =  "check-hashicups-api.public",
    name = "hashicups-api.public status check",
    service_id = "hashicups-api-1",
    tcp  = "localhost:8081",
    interval = "5s",
    timeout = "5s"
  },
  {
    id =  "check-hashicups-api.product",
    name = "hashicups-api.product status check",
    service_id = "hashicups-api-1",
    tcp  = "localhost:9090",
    interval = "5s",
    timeout = "5s"
  },
  {
    id =  "check-hashicups-api.payments",
    name = "hashicups-api.payments status check",
    service_id = "hashicups-api-1",
    tcp  = "localhost:8080",
    interval = "5s",
    timeout = "5s"
  }]
}
EOF

```

That will produce no output.

> To distinguish the two instances you are adding the `inst_1` tag to the service definition.

Copy the configuration file on the `hashicups-api-1` node.

```shell-session
$ scp -r -i /home/admin/certs/id_rsa /home/admin/assets/scenario/conf/hashicups-api-1/svc-hashicups-api.hcl admin@hashicups-api-1:/etc/consul.d/svc-hashicups-api.hcl
```

That will produce no output.

Login to `hashicups-api-1` from the bastion host.

```
$ ssh -i certs/id_rsa hashicups-api-1
#..
admin@hashicups-api-1:~
```


Verify the file got copied correctly on the VM.

```shell-session
$ cat /etc/consul.d/svc-hashicups-api.hcl
```

That will produce an output similar to the following.

```hcl
## -----------------------------
## svc-hashicups-api.hcl
## -----------------------------
service {
  name = "hashicups-api"
  id = "hashicups-api-1"
  tags = [ "inst_1" ]
  port = 8081
  token = "540d7fa7-bf32-c295-651e-51010f55720b"

  checks =[
  {
  id =  "check-hashicups-api.public.http",
  name = "hashicups-api.public  HTTP status check",
  service_id = "hashicups-api-1",
  http  = "http://localhost:8081/health",
  interval = "5s",
  timeout = "5s"
  },
  {
    id =  "check-hashicups-api.public",
    name = "hashicups-api.public status check",
    service_id = "hashicups-api-1",
    tcp  = "localhost:8081",
    interval = "5s",
    timeout = "5s"
  },
  {
    id =  "check-hashicups-api.product",
    name = "hashicups-api.product status check",
    service_id = "hashicups-api-1",
    tcp  = "localhost:9090",
    interval = "5s",
    timeout = "5s"
  },
  {
    id =  "check-hashicups-api.payments",
    name = "hashicups-api.payments status check",
    service_id = "hashicups-api-1",
    tcp  = "localhost:8080",
    interval = "5s",
    timeout = "5s"
  }]
}
```


Reload Consul to apply the service configuration.

```shell-session
$ consul reload
```

That will produce an output similar to the following.

```plaintext
Configuration reload triggered
```


Start `hashicups-api` service.

```shell-session
$ ~/start_service.sh
```

That will produce an output similar to the following.

```plaintext
b775696862bf0bf32b5ff5b5a89883b7a8d112f97a2d2ec623f16a23cd9a9c53
12dfd567d5285ecc129d58c0b16293e7bea49f05b6e466e9d268a1db08cac955
d1c71fb3b6b3877ba70325bd9ceb291b8e273b8687df748cc5964c2175b8f6b6
```


To continue with the tutorial, exit the ssh session to return to the bastion host.

```
$ exit
logout
Connection to hashicups-api-0 closed.
admin@bastion:~$
```


## Verify service registration

Consul CLI

```shell-session
$ consul catalog services -tags
```

That will produce an output similar to the following.

```plaintext
consul                  
hashicups-api           inst_0,inst_1
hashicups-db            inst_0
hashicups-frontend      inst_0
hashicups-nginx         inst_0
```


Consul API

```shell-session
$ curl --silent \
   --header "X-Consul-Token: $CONSUL_HTTP_TOKEN" \
   --connect-to server.${CONSUL_DATACENTER}.${CONSUL_DOMAIN}:8443:consul-server-0:8443 \
   --cacert ${CONSUL_CACERT} \
   https://server.${CONSUL_DATACENTER}.${CONSUL_DOMAIN}:8443/v1/catalog/service/hashicups-api | jq -r .
```

That will produce an output similar to the following.

```json
[
  {
    "ID": "c4cb2441-e455-e2db-e706-14bbad234011",
    "Node": "hashicups-api-0",
    "Address": "10.0.4.119",
    "Datacenter": "dc1",
    "TaggedAddresses": {
      "lan": "10.0.4.119",
      "lan_ipv4": "10.0.4.119",
      "wan": "10.0.4.119",
      "wan_ipv4": "10.0.4.119"
    },
    "NodeMeta": {
      "consul-network-segment": "",
      "consul-version": "1.17.0"
    },
    "ServiceKind": "",
    "ServiceID": "hashicups-api-0",
    "ServiceName": "hashicups-api",
    "ServiceTags": [
      "inst_0"
    ],
    "ServiceAddress": "",
    "ServiceWeights": {
      "Passing": 1,
      "Warning": 1
    },
    "ServiceMeta": {},
    "ServicePort": 8081,
    "ServiceSocketPath": "",
    "ServiceEnableTagOverride": false,
    "ServiceProxy": {
      "Mode": "",
      "MeshGateway": {},
      "Expose": {}
    },
    "ServiceConnect": {},
    "ServiceLocality": null,
    "CreateIndex": 58,
    "ModifyIndex": 58
  },
  {
    "ID": "55acde08-475a-731a-acdd-41d9711be386",
    "Node": "hashicups-api-1",
    "Address": "10.0.4.195",
    "Datacenter": "dc1",
    "TaggedAddresses": {
      "lan": "10.0.4.195",
      "lan_ipv4": "10.0.4.195",
      "wan": "10.0.4.195",
      "wan_ipv4": "10.0.4.195"
    },
    "NodeMeta": {
      "consul-network-segment": "",
      "consul-version": "1.17.0"
    },
    "ServiceKind": "",
    "ServiceID": "hashicups-api-1",
    "ServiceName": "hashicups-api",
    "ServiceTags": [
      "inst_1"
    ],
    "ServiceAddress": "",
    "ServiceWeights": {
      "Passing": 1,
      "Warning": 1
    },
    "ServiceMeta": {},
    "ServicePort": 8081,
    "ServiceSocketPath": "",
    "ServiceEnableTagOverride": false,
    "ServiceProxy": {
      "Mode": "",
      "MeshGateway": {},
      "Expose": {}
    },
    "ServiceConnect": {},
    "ServiceLocality": null,
    "CreateIndex": 92,
    "ModifyIndex": 92
  }
]
```


Using `dig` command

```shell-session
$ dig @consul-server-0 -p 8600 hashicups-api.service.dc1.consul
```

That will produce an output similar to the following.

```plaintext

; <<>> DiG 9.16.44-Debian <<>> @consul-server-0 -p 8600 hashicups-api.service.dc1.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 221
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;hashicups-api.service.dc1.consul. IN	A

;; ANSWER SECTION:
hashicups-api.service.dc1.consul. 0 IN	A	10.0.4.119

;; Query time: 3 msec
;; SERVER: 10.0.4.16#8600(10.0.4.16)
;; WHEN: Thu Nov 16 14:13:04 UTC 2023
;; MSG SIZE  rcvd: 77
```


Notice the output reports two instances for the service.

## Verify Consul load balancing functionalities

When multiple instances of a service are defined, Consul DNS will automatically provide basic round-robin load balancing capabilities.

```shell-session
$ for i in `seq 1 100` ; do dig @consul-server-0 -p 8600 hashicups-api.service.dc1.consul +short | head -1; done | sort | uniq -c
```

That will produce an output similar to the following.

```plaintext
    100 10.0.4.119
```


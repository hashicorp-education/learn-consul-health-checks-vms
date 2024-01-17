# Manage permissions with access control lists (ACLs)

This repository contains companion code for the following tutorial:

- [Monitor your application health with distributed checks](https://developer.hashicorp.com/consul/tutorials/developer-discovery/monitor-applications-health-checks)

> **WARNING:** the script is currently under development. Some configurations might not work as expected. **Do not test on production environments.**

The code in this repository is derived from [hashicorp-education/learn-consul-get-started-vms](https://github.com/hashicorp-education/learn-consul-get-started-vms).


### Deploy

```
cd self-managed/infrastructure/aws
```

```
terraform apply --auto-approve -var-file=../../ops/conf/monitor_application_health_with_distributed_checks.tfvars
```
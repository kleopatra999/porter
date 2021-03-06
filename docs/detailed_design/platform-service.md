platform-service interface
==========================

This is the interface services need to implement in order to work with porter.

porter favors convention over configuration where possible. This document
describes conventions used in porter, followed by the minimal configuration
needed, and finally some optional configuration.

Convention
----------

### Required Files

The minimum required files are `.porter/config` and a `Dockerfile`.

#### .porter/config

`.porter/config` is a YAML file describing high-level constructs such as the
service name, container definitions, environment-region tuples, etc.

[See a sample](../../sample_project/.porter/config)

#### Dockerfile

`Dockerfile` is the default Docker container that's built

[See a sample](../../sample_project/Dockerfile)

### Logging

Porter follows [12factor](http://12factor.net/logs) when it comes to logging.

Services are expected to log on STDOUT.

Logs are sent to rsyslog, which are aggregated into `/var/log/porter.log`.

The very first script run by an EC2 instance logs to
`/var/log/cloud-init-output.log` which can be useful when debugging failures
that lead to CloudFormation stacks not completing due to WaitCondition failures.

### Health check

The default health check is `GET /health HTTP/1.1`. A newly provisioned stack
won't complete and a hotswap won't succeed if a service doesn't return a
`HTTP/1.1 200 OK` response.

The health check method and path are configurable.

### Additional environment variables

**Who and where am I?**

```
PORTER_ENVIRONMENT
AWS_REGION
```

**Secrets and other service-defined config**

See [Container config](container-config.md) for more.

Minimal Configuration
---------------------

For exact required fields see the [config reference](config-reference.md)

The minimum configuration includes

- the service's name
- a single environment and AWS region
- the pre-built ELB in the environment-region (for elb-based topologies)

Optional Configuration
----------------------

An optional CloudFormation template can be provided.

See [CloudFormation customizations](cfn-customizations.md) for details

+++
title="How to Validate That Your Docker Configuration is Secure Using the Go SDK"
date=2024-08-27

[taxonomies]
categories = ["Docker", "Compliance", "CIS Benchmark"]
tags = ["post"]
+++

# Table of Contents
[Overview](#overview) <br/>
[Get Docker's Configuration from a Docker Client](#get-dockers-configuration-from-a-docker-client) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Create Docker Client](#create-docker-client) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Create Middleware Function](#create-middleware-function) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Run it!](#run-it) <br/>
[CIS Benchmark](#cis-benchmark) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Understanding CIS Benchmark](#understanding-cis-benchmark) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Why is it important?](#why-is-it-important) <br/>
[Validate if Your Docker Configuration is Secure](#validate-if-your-docker-configuration-is-secure) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Ensure live restore is enabled](#ensure-live-restore-is-enabled) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Run it!](#run-it) <br/>
[Conclusion](#conclusion) <br/>

# Overview

Today, I'll discuss how to interact with Docker on your host using the Go SDK. The 'go-docker' project by the Docker group was archived on March 11, 2024, and integrated into the Moby project. This change has modified how it is used.

Next, I will explain how to verify if your Docker environment is properly configured for production. I will reference the CIS benchmark and discuss how to maintain a secure environment in compliance with these standards.


# Get Docker’s Configuration from a Docker Client
## Create Docker Client

To retrieve Docker’s configuration from the Docker daemon, you first need to create a Docker client. Creating a new Docker client with the following code is recommended for efficiency and reusability:

```go
package client

import (
    "github.com/docker/docker/client"
    "context"
    "log"
)

type DockerClient struct {
    Client *client.Client
}

func NewDockerClient() (*DockerClient, error) {
    cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
    if err != nil {
        return nil, err
    }
    return &DockerClient{Client: cli}, nil
}
```

## Create Middleware Function

This process can be skipped as it merely replicates the original functionality. I created the middleware function for the SDK for convenience:

```go
package collector

import (
    "context"
    "agentless-for-docker-dev/util/client"
    "github.com/docker/docker/api/types/system"
)

func GetDetailedSystemInfo(dockerClient *client.DockerClient) (system.Info, error) {
    info, err := dockerClient.Client.Info(context.Background())
    if err != nil {
        return system.Info{}, err
    }
    return info, nil
}

```

## Run it!

Now, you can run the code in your desired manner. In my case, I expose this code via the go-gin web framework and Swagger API docs, allowing you to simply click the “Execute” button to retrieve the data. The result provides comprehensive information about your Docker system.

![Swagger Docs](https://github.com/user-attachments/assets/18f13d19-7271-4dd9-932d-13eefe592846)


# CIS Benchmark

## Understanding CIS Benchmark

Before starting the evaluation of the security benchmark, let's explore what the CIS benchmark is and why it is important to adhere to it.

### CIS Benchmark

The CIS (Center for Internet Security) Benchmarks are consensus-based best practice standards for security configurations. They offer detailed guidelines for various platforms and systems to ensure secure configurations.

### Why is it important?

- **Enhanced Security:** CIS Benchmarks provide technical controls to mitigate vulnerabilities and protect assets from cyber attacks.
- **Compliance:** Adhering to CIS Benchmarks helps meet regulatory requirements aligned with standards such as HIPAA, PCI-DSS, and SOX.
- **Consistent Implementation:** Following CIS Benchmarks enables organizations to implement consistent security policies and procedures, standardizing security across systems and ensuring uniform security levels.


# Validate if Your Docker Configuration is Secure

## **Ensure live restore is enabled**

One critical aspect of the CIS Docker Benchmark is “**2.15 Ensure live restore is enabled**.” Here’s a quote from the CIS Benchmark explaining its importance:

> The `--live-restore` option enables full support of daemon-less containers within Docker. It ensures that Docker does not stop containers on shutdown or restore and that it properly reconnects to the container when restarted.
> 

Then, how can we get this one? According to the CIS Benchmark, we can check this configuration through docker command like `docker info --format '{{ .LiveRestoreEnabled }}'`  or simple use ps command like `ps -ef | grep dockerd` .

But we want this to be programmatic, right?

To do that, you can use the function we wrote above. The code looks like this:

```go
package evaluator

import (
    "fmt"

    "agentless-for-docker-dev/pkg/evaluator/types"
    "agentless-for-docker-dev/util/client"
    "agentless-for-docker-dev/pkg/collector"
)

const (
    TargetConfiguration            = "LiveRestoreEnabled"
    DesiredLiveRestoreEnabledState = "true"
    DefaultLiveRestoreEnabledState = "false"
)

func EvaluateLiveRestoreEnabled(dockerClient *client.DockerClient) (types.AuditResult, error) {
    systemInfo, err := collector.GetDetailedSystemInfo(dockerClient)
    if err != nil {
        return types.NewAuditResult("", "Failed to retrieve system info", nil), err
    }

    constMap := map[string]string{
        "DesiredLiveRestoreEnabledState": DesiredLiveRestoreEnabledState,
        "DefaultLiveRestoreEnabledState": DefaultLiveRestoreEnabledState,
        "TargetConfiguration":            TargetConfiguration,
    }

    actualState := fmt.Sprintf("%v", systemInfo.LiveRestoreEnabled)
    auditResult := "FAIL"
    if actualState == DesiredLiveRestoreEnabledState {
        auditResult = "PASS"
    }
    auditMessage := fmt.Sprintf("%s is set to %s", TargetConfiguration, actualState)

    return types.NewAuditResult(
        auditResult,
        auditMessage,
        constMap,
    ), nil
}
```

## Run it!

Execute the code using go-gin and Swagger for API documentation. You can then determine if your Docker configuration meets the CIS Benchmark standards.

![Swagger Docs](https://github.com/user-attachments/assets/792a4d53-f0b2-4f29-bd19-1c9538fe071c)


# Conclusion

This post demonstrated how to interact with Docker using the Go SDK, specifically focusing on secure configuration practices as outlined by the CIS benchmark. By programmatically assessing and validating Docker security settings, this approach serves as a foundational method for implementing **Cloud Security Posture Management** (CSPM). It ensures compliance with regulatory standards such as HIPAA, PCI-DSS, and SOX, while standardizing security practices across all cloud and container environments. Such proactive security measures are crucial for maintaining integrity and security in today’s complex cloud landscapes, ensuring that environments are not only compliant but resilient against potential threats.
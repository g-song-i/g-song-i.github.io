+++
title="W(Where)TF is Docker daemon.json File?"
date=2024-09-09

[taxonomies]
categories = ["Docker", "Docker Daemon", "daemon configuration"]
tags = ["post"]
+++

# Overview

This article looks into the details of Docker daemon configurations after I noticed that the `daemon.json` file was missing from my host system. It explains what the Docker daemon is, what it does, and why the `daemon.json` file is missing. The discussion includes how Docker is typically set up and how to access and understand the daemon's settings. 

# Backgrounds

## What is Docker Daemon?

Let's grab some background first. What do you think the Docker daemon is?

The Docker daemon is a part of the Docker engine, running on the host OS as a process. According to [Aqua Security](https://www.aquasec.com/cloud-native-academy/docker-container/docker-architecture/#:~:text=Docker%20Daemon%3A%20A%20persistent%20background,API%20requests%20and%20processes%20them), the Docker daemon is described as follows:

> A persistent background process that manages Docker images, containers, networks, and storage volumes. The Docker daemon constantly listens for Docker API requests and processes them.
> 

## What is Docker’s daemon.json?

The `daemon.json` file is used for configuring the Docker daemon. According to the official Docker [documentation](https://docs.docker.com/engine/daemon/start/), there are two ways to configure the Docker daemon:

> 
> 
> - Use a JSON configuration file. This is the preferred option, since it keeps all configurations in a single place.
> - Use flags when starting `dockerd`.

You can use both ways for configuring the daemon. According to [this post](https://www.devopsschool.com/blog/how-to-configure-docker-daemon-with-a-configuration-file/), setting it with a JSON file is preferred, but I'm not sure (it's not from the official documentation). 

# How to get Docker daemon configuration?

Now you know that the Docker daemon is a process responsible for managing Docker's components, and there is a file named `daemon.json` that contains configurations for the Docker daemon. So, how do you get the Docker daemon's configuration when Docker is running? There are two options:

- **Get the configuration from the `daemon.json` file if the daemon is started with it**
- **Get the configuration using the `docker info` command**

You can use the `ps` command to inspect the `dockerd` process, but it might not display all the configurations we need. Additionally, you can check the configuration file, `daemon.json`, but there is no direct way to confirm whether the current Docker daemon was started using that specific file.

But, we can assume that if the `ps` command returns a path to a file with the [`--config-file`](https://docs.docker.com/engine/daemon/#configuration-file) parameter, then `dockerd` was started with that file. **If not, `dockerd` was started with the default configuration, either from their default location or the configuration** embedded in the software upon installation. You can refer [here](https://github.com/moby/moby/tree/master/daemon/config) to see the code for default configuration.

## Daemon file’s default location

According to [this post](https://docs.docker.com/engine/daemon/#configuration-file) from the Docker docs, the Docker daemon expects to find the configuration file by default at this location:

| OS | File |
| --- | --- |
| Linux, regular setup | /etc/docker/daemon.json |
| Linux, rootless mode | ~/.config/docker/daemon.json |
| Windows | C:\ProgramData\docker\config\daemon.json |

## W(Where)TF is my daemon.json????

This part is the most important in this post. I couldn’t find my `daemon.json` in the default location. What’s happening? Like what I said, Dockerd can be started with the default configuration without a configuration file, as [it is not created automatically](https://stackoverflow.com/questions/43689271/wheres-dockers-daemon-json-missing). That’s the reason you can’t find it in your location.

You can create the file if you want to boot Dockerd with that in the default location, or you can use another location and specify the customized location manually. So, do not panic if there is no `daemon.json`; you can still get the daemon’s configuration in other ways

# Get the current Dockerd configuration

Here, I describe ways to get the Dockerd configuration. As I mentioned before, you can find it in the `daemon.json` file or use the `docker info` command. Let’s assume that you have not created a `daemon.json` file before. Then, there are two options:

- Get the configuration using the `docker info` command.
- Get the configuration with the code.

Hah? Another option pops up, right? Let’s start!

## Get the configuration with docker info

First, you can see your daemon's configuration using the `docker info` command. Here is what the result looks like:

```bash
$ docker info
Client: Docker Engine - Community
 Version:    27.0.3
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.15.1
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v2.28.1
    Path:     /usr/libexec/docker/cli-plugins/docker-compose

Server:
 Containers: 18
  Running: 17
  Paused: 0
  Stopped: 1
 Images: 22
 Server Version: 27.0.3
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Using metacopy: false
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local splunk syslog
 Swarm: inactive
 Runtimes: runc io.containerd.runc.v2
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: ae71819c4f5e67bb4d5ae76a6b735f29cc25774e
 runc version: v1.1.13-0-g58aa920
 init version: de40ad0
 Security Options:
  apparmor
  seccomp
   Profile: builtin
  cgroupns
 Kernel Version: 6.5.0-44-generic
 Operating System: Ubuntu 22.04.4 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 16
 Total Memory: 62.88GiB
 Name: <hostname>
 ID: b3f46722-9862-43fd-913d-c8a4bd6a349e
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
```

## Get the configuration with the code (Go SDK)

Moreover, you can write code to retrieve the configuration! I use the Go SDK for this purpose. Here is an example, and you will need to configure the way to handle the result appropriately.

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

The result contains a lot of data, as described [here](https://github.com/moby/moby/blob/master/api/types/system/info.go). It might differ from the result of the command in some ways, so you need to make an effort to identify the differences and choose the right option for you. Part of the result looks like this:

```bash
{
  "ID": "b3f46722-9862-43fd-913d-c8a4bd6a349e",
  "Containers": 18,
  "ContainersRunning": 17,
  "ContainersPaused": 0,
  "ContainersStopped": 1,
  "Images": 22,
  "Driver": "overlay2",
  "DriverStatus": [
    [
      "Backing Filesystem",
      "extfs"
    ],
    [
      "Supports d_type",
      "true"
    ],
    [
      "Using metacopy",
      "false"
    ],
    [
      "Native Overlay Diff",
      "true"
    ],
    [
      "userxattr",
      "false"
    ]
  ],
  "Plugins": {
    "Volume": [
      "local"
    ],
    "Network": [
      "bridge",
      "host",
      "ipvlan",
      "macvlan",
      "null",
      "overlay"
    ],
    "Authorization": null,
    "Log": [
      "awslogs",
      "fluentd",
      "gcplogs",
      "gelf",
      "journald",
      "json-file",
      "local",
      "splunk",
      "syslog"
    ]
  },
  "MemoryLimit": true,
  "SwapLimit": true,
  "CpuCfsPeriod": true,
  "CpuCfsQuota": true,
  "CPUShares": true,
  "CPUSet": true,
  "PidsLimit": true,
  "IPv4Forwarding": true,
  "BridgeNfIptables": true,
  "BridgeNfIp6tables": true,
  "Debug": false,
  ...
```

# Conclusion

To wrap up, I found out why the `daemon.json` file was missing from my host system. It was because the Docker daemon was started with the default configuration. I also learned how to get the Docker daemon's configuration using the `docker info` command and the Go SDK. I hope this post helps you understand the Docker daemon and its configuration better. Thank you for reading!
---
layout: "docs"
page_title: "Drivers: podman"
sidebar_current: "docs-drivers-community-podman"
description: |-
  The Podman task driver uses podman (https://podman.io/) for containerizing tasks.
---

# Podman Task Driver

Name: `podman`

Homepage: https://github.com/pascomnet/nomad-driver-podman

The podman task driver plugin for Nomad uses the [Pod Manager (podman)][podman]
daemonless container runtime for executing Nomad tasks. Podman supports OCI
containers and its command line tool is meant to be [a drop-in replacement for
Docker's][podman-cli].

See the project's [homepage][homepage] for details.

## Client Requirements

* Linux host with [`podman`][podman] installed.
* [`nomad-driver-podman`][releases] binary in Nomad's [`plugin_dir`][plugin_dir].

You need a varlink enabled podman binary and a system socket activation unit, see https://podman.io/blogs/2019/01/16/podman-varlink.html.

Since the Nomad agent, nomad-driver-podman plugin binary, and podman will
reside on the same host, skip the ssh aspects of the podman varlink
documentation above.

## Task Configuration

Due to Podman's similarity to Docker, the example job created by [`nomad init
-short`][nomad-init] is easily adapted to use Podman instead:

```hcl
job "example" {
  datacenters = ["dc1"]

  group "cache" {
    task "redis" {
      driver = "podman"

      config {
        image = "docker://redis:3.2"

        port_map {
          db = 6379
        }
      }

      resources {
        cpu    = 500
        memory = 256

        network {
          mbits = 10
          port  "db"  {}
        }
      }
    }
  }
}
```

* `image` - The image to run.

```hcl
config {
  image = "docker://redis"
}
```

* `command` - (Optional) The command to run when starting the container.

```hcl
config {
  command = "some-command"
}
```

* `args` - (Optional) A list of arguments to the optional command. If no
  *command* is specified, the arguments are passed directly to the container.

```hcl
config {
  args = [
    "arg1",
    "arg2",
  ]
}
```

* `volumes` - (Optional) A list of `host_path:container_path` strings to bind
  host paths to container paths. 

```hcl
config {
  volumes = [
    "/some/host/data:/container/data"
  ]
}
```

* `tmpfs` - (Optional) A list of `/container_path` strings for tmpfs mount
  points. See `podman run --tmpfs` options for details. 

```hcl
config {
  tmpfs = [
    "/var"
  ]
}
```

* `hostname` -  (Optional) The hostname to assign to the container. When
  launching more than one of a task (using count) with this option set, every
  container the task starts will have the same hostname.

* `init` - Run an init inside the container that forwards signals and reaps processes.

```hcl
config {
  init = true
}
```

* `init_path` - Path to the container-init binary.

```hcl
config {
  init = true
  init_path = "/usr/libexec/podman/catatonit"
}
```

* `user` - Run the command as a specific user/uid within the container. See
  [task configuration][task].

* `memory_reservation` - Memory soft limit (unit = b (bytes), k (kilobytes), m
  (megabytes), or g (gigabytes))

After setting memory reservation, when the system detects memory contention or
low memory, containers are forced to restrict their consumption to their
reservation. So you should always set the value below --memory, otherwise the
hard limit will take precedence. By default, memory reservation will be the
same as memory limit.

```hcl
config {
  memory_reservation = "100m"
}
```

* `memory_swap` - A limit value equal to memory plus swap. The swap limit
  should always be larger than the [memory value][memory-value]. 

Unit can be b (bytes), k (kilobytes), m (megabytes), or g (gigabytes). If you
don't specify a unit, b is used. Set LIMIT to -1 to enable unlimited swap.

```hcl
config {
  memory_swap = "180m"
}
```

* `memory_swappiness` - Tune a container's memory swappiness behavior.  Accepts
  an integer between 0 and 100.

```hcl
config {
  memory_swappiness = 60
}
```

## Networking

Podman supports forwarding and exposing ports like Docker. See [Docker Driver
configuration][docker-ports] for details.

## Plugin Options

The podman plugin has options which may be customized in the agent's
configuration file.

* `volumes` stanza:

  * `enabled` - Defaults to `true`. Allows tasks to bind host paths (volumes)
    inside their container. 
  * `selinuxlabel` - Allows the operator to set a SELinux label to the
    allocation and task local bind-mounts to containers. If used with
    `volumes.enabled` set to false, the labels will still be applied to the
    standard binds in the container.

```hcl
plugin "nomad-driver-podman" {
  config {
    volumes {
      enabled      = true
      selinuxlabel = "z"
    }
  }
}
```

* `gc` stanza:

    * `container` - Defaults to `true`. This option can be used to disable
      Nomad from removing a container when the task exits.

```hcl
plugin "nomad-driver-podman" {
  config {
    gc {
      container = false
    }
  }
}
```

* `recover_stopped` - Defaults to `true`. Allows the driver to start and reuse
  a previously stopped container after a Nomad client restart. 
  Consider a simple single node system and a complete reboot. All previously managed containers
  will be reused instead of disposed and recreated.

```hcl
plugin "nomad-driver-podman" {
  config {
    recover_stopped = false
  }
}
```

[docker-ports]: /docs/drivers/docker.html#forwarding-and-exposing-ports
[homepage]: https://github.com/pascomnet/nomad-driver-podman
[memory-value]: /docs/job-specification/resources.html#memory
[nomad-init]: /docs/commands/job/init.html
[plugin_dir]: /docs/configuration/index.html#plugin_dir
[podman]: https://podman.io/
[podman-cli]: https://podman.io/whatis.html
[releases]: https://github.com/pascomnet/nomad-driver-podman/releases
[task]: /docs/job-specification/task.html#user

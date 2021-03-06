---
title: "Troubleshoot"
linkTitle: "Troubleshoot"
description: "Here’s how to troublshoot Sensu, including how to look into errors, service logging, log levels. Sensu service logs produced by sensu-backend and sensu-agent are often the best source of truth when troubleshooting issues, so start there."
weight: 210
version: "5.18"
product: "Sensu Go"
platformContent: true
platforms: ["Linux", "Windows"]
menu:
  sensu-go-5.18:
    parent: guides
---

- [Service logging](#service-logging)
	- [Log levels](#log-levels)
	- [Log file locations](#log-file-locations)
- [Sensu backend startup errors](#sensu-backend-startup-errors)
- [Permission issues](#permission-issues)
- [Handlers and event filters](#handlers-and-event-filters)
- [Assets](#assets)

## Service logging

Logs produced by Sensu services (sensu-backend and sensu-agent) are often the best place to start when troubleshooting a variety of issues.

### Log levels

Each log message is associated with a log level that indicates the relative severity of the event being logged:

| Log level          | Description |
|--------------------|--------------------------------------------------------------------------|
| panic              | Severe errors that cause the service to shut down in an unexpected state |
| fatal              | Fatal errors that cause the service to shut down (status 0)              |
| error              | Non-fatal service error messages                                         |
| warn               | Warning messages that indicate potential issues                          |
| info               | Information messages that represent service actions                      |
| debug              | Detailed service operation messages to help troubleshoot issues          |

You can configure these log levels by specifying the desired log level as the value of `log-level` in the service configuration file (`agent.yml` or `backend.yml`) or as an argument to the `--log-level` command line flag:

{{< highlight shell >}}
sensu-agent start --log-level debug
{{< /highlight >}}

You must restart the service if you change log levels via configuration files or command line arguments.
For help with restarting a service, see the [agent reference][5] or [backend reference][9].

### Log file locations

{{< platformBlock "Linux" >}}

#### Linux

Sensu services print [structured log messages][7] to standard output.
To capture these log messages to disk or another logging facility, Sensu services use capabilities provided by the underlying operating system's service management.
For example, logs are sent to the journald when systemd is the service manager, whereas log messages are redirected to `/var/log/sensu` when running under sysv init schemes.
If you are running systemd as your service manager and would rather have logs written to `/var/log/sensu/`, see [forwarding logs from journald to syslog][11].

The following table lists the common targets for logging and example commands for following those logs.
You may substitute the name of the desired service (e.g. `backend` or `agent`) for the `${service}` variable.

| Platform     | Version           | Target | Command to follow log |
|--------------|-------------------|--------------|-----------------------------------------------|
| RHEL/Centos  | >= 7       | journald     | {{< highlight shell >}}journalctl --follow --unit sensu-${service}{{< /highlight >}}   |
| RHEL/Centos  | <= 6       | log file     | {{< highlight shell >}}tail --follow /var/log/sensu/sensu-${service}{{< /highlight >}} |
| Ubuntu       | >= 15.04   | journald     | {{< highlight shell >}}journalctl --follow --unit sensu-${service}{{< /highlight >}}   |
| Ubuntu       | <= 14.10   | log file     | {{< highlight shell >}}tail --follow /var/log/sensu/sensu-${service}{{< /highlight >}} |
| Debian       | >= 8       | journald     | {{< highlight shell >}}journalctl --follow --unit sensu-${service}{{< /highlight >}}   |
| Debian       | <= 7       | log file     | {{< highlight shell >}}tail --follow /var/log/sensu/sensu-${service}{{< /highlight >}} |

{{% notice note %}}
**NOTE**: Platform versions are listed for reference only and do not supersede the documented [supported platforms](../../installation/platforms).
{{% /notice %}}

##### Narrow your search to a specific timeframe

Use the `journald` keyword `since` to refine the basic `journalctl` commands and narrow your search by timeframe.

Retrieve all the logs for Sensu since yesterday:

{{< highlight shell >}}
journalctl _COMM=sensu-backend.service --since yesterday | tee sensu-backend-$(date +%Y-%m-%d).log
{{< /highlight >}}

Retrieve all the logs for Sensu since a specific time:

{{< highlight shell >}}
journalctl _COMM=sensu-backend.service --since 09:00 --until "1 hour ago" | tee sensu-backend-$(date +%Y-%m-%d).log
{{< /highlight >}}

Retrieve all the logs for Sensu for a specific date range:

{{< highlight shell >}}
journalctl _COMM=sensu-backend.service --since "2015-01-10" --until "2015-01-11 03:00" | tee sensu-backend-$(date +%Y-%m-%d).log
{{< /highlight >}}

{{< platformBlockClose >}}

{{< platformBlock "Windows" >}}

#### Windows

The Sensu agent stores service logs to the location specified by the `log-file` configuration flag (default `%ALLUSERSPROFILE%\sensu\log\sensu-agent.log`, `C:\ProgramData\sensu\log\sensu-agent.log` on standard Windows installations).
For more information about managing the Sensu agent for Windows, see the [agent reference][1].
You can also view agent events using the Windows Event Viewer, under Windows Logs, as events with source SensuAgent.

If you're running a [binary-only distribution of the Sensu agent for Windows][2], you can follow the service log printed to standard output using this command:

{{< highlight text >}}
Get-Content -  Path "C:\scripts\test.txt" -Wait
{{< /highlight >}}

{{< platformBlockClose >}}

## Sensu backend startup errors

The following errors are expected when starting up a Sensu backend with the default configuration:

{{< highlight shell >}}
{"component":"etcd","level":"warning","msg":"simple token is not cryptographically signed","pkg":"auth","time":"2019-11-04T10:26:31-05:00"}
{"component":"etcd","level":"warning","msg":"set the initial cluster version to 3.3","pkg":"etcdserver/membership","time":"2019-11-04T10:26:31-05:00"}
{"component":"etcd","level":"warning","msg":"serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!","pkg":"embed","time":"2019-11-04T10:26:33-05:00"}
{{< /highlight >}}

The `serving insecure client requests` warning is an expected warning from the embedded etcd database.
[TLS configuration][3] is recommended but not required.
For more information, see [etcd security documentation][4].

## Permission issues

The Sensu user and group must own files and folders within `/var/cache/sensu/` and `/var/lib/sensu/`.
You will see a logged error like those listed here if there is a permission issue with either the sensu-backend or the sensu-agent:

{{< highlight shell >}}
{"component":"agent","error":"open /var/cache/sensu/sensu-agent/assets.db: permission denied","level":"fatal","msg":"error executing sensu-agent","time":"2019-02-21T22:01:04Z"}
{"component":"backend","level":"fatal","msg":"error starting etcd: mkdir /var/lib/sensu: permission denied","time":"2019-03-05T20:24:01Z"}
{{< /highlight >}}

Use a recursive `chown` to resolve permission issues with the sensu-backend:

{{< highlight shell >}}
sudo chown -R sensu:sensu /var/cache/sensu/sensu-backend
{{< /highlight >}}

or the sensu-agent:

{{< highlight shell >}}
sudo chown -R sensu:sensu /var/cache/sensu/sensu-agent
{{< /highlight >}}

## Handlers and event filters

Whether implementing new workflows or modifying existing workflows, you may need to troubleshoot various stages of the event pipeline.
In many cases, generating events using the [agent API][6] will save you time and effort over modifying existing check configurations.

Here's an example that uses cURL with the API of a local sensu-agent process to generate test-event check results:

{{< highlight shell >}}
curl -X POST \
-H 'Content-Type: application/json' \
-d '{
  "check": {
    "metadata": {
      "name": "test-event"
    },
    "status": 2,
    "output": "this is a test event targeting the email_ops handler",
    "handlers": [ "email_ops" ]
  }
}' \
http://127.0.0.1:3031/events
{{< /highlight >}}

It may also be helpful to see the complete event object being passed to your workflows.
We recommend using a debug handler like this one to write an event to disk as JSON data:

{{< language-toggle >}}

{{< highlight yml >}}
type: Handler
api_version: core/v2
metadata:
  name: debug
spec:
  type: pipe
  command: cat > /var/log/sensu/debug-event.json
  timeout: 2
{{< /highlight >}}

{{< highlight json >}}
{
  "type": "Handler",
  "api_version": "core/v2",
  "metadata": {
    "name": "debug"
  },
  "spec": {
    "type": "pipe",
    "command": "cat > /var/log/sensu/debug-event.json",
    "timeout": 2
  }
}
{{< /highlight >}}

{{< /language-toggle >}}

With this handler definition installed in your Sensu backend, you can add the `debug` to the list of handlers in your test event:

{{< highlight shell >}}
curl -X POST \
-H 'Content-Type: application/json' \
-d '{
  "check": {
    "metadata": {
      "name": "test-event"
    },
    "status": 2,
    "output": "this is a test event targeting the email_ops handler",
    "handlers": [ "email_ops", "debug" ]
  }
}' \
http://127.0.0.1:3031/events
{{< /highlight >}}

The event data should be written to `/var/log/sensu/debug-event.json` for inspection.
The contents of this file will be overwritten by every event sent to the `debug` handler.

{{% notice note %}}
**NOTE**: When multiple Sensu backends are configured in a cluster, event processing is distributed across all members.
You may need to check the filesystem of each Sensu backend to locate the debug output for your test event.
{{% /notice %}}

## Assets

Asset filters allow you to scope an asset to a particular operating system or architecture.
You can see an example in the [asset reference][10].
An improperly applied asset filter can prevent the asset from being downloaded by the desired entity and result in error messages both on the agent and the backend illustrating that the command was not found:

**Agent log entry**

{{< highlight json >}}
{
    "asset": "check-disk-space",
    "component": "asset-manager",
    "entity": "sensu-centos",
    "filters": [
        "true == false"
    ],
    "level": "debug",
    "msg": "entity not filtered, not installing asset",
    "time": "2019-09-12T18:28:05Z"
}
{{< /highlight >}}

**Backend event**

{{< highlight json >}}

 {
  "timestamp": 1568148292,
  "check": {
    "command": "check-disk-space",
    "handlers": [],
    "high_flap_threshold": 0,
    "interval": 10,
    "low_flap_threshold": 0,
    "publish": true,
    "runtime_assets": [
      "sensu-plugins-disk-checks"
    ],
    "subscriptions": [
      "caching_servers"
    ],
    "proxy_entity_name": "",
    "check_hooks": null,
    "stdin": false,
    "subdue": null,
    "ttl": 0,
    "timeout": 0,
    "round_robin": false,
    "duration": 0.001795508,
    "executed": 1568148292,
    "history": [
      {
        "status": 127,
        "executed": 1568148092
      }
    ],
    "issued": 1568148292,
    "output": "sh: check-disk-space: command not found\n",
    "state": "failing",
    "status": 127,
    "total_state_change": 0,
    "last_ok": 0,
    "occurrences": 645,
    "occurrences_watermark": 645,
    "output_metric_format": "",
    "output_metric_handlers": null,
    "env_vars": null,
    "metadata": {
      "name": "failing-disk-check",
      "namespace": "default"
    }
  },
  "metadata": {
    "namespace": "default"
  }
}
{{< /highlight >}}

If you see a message like this, review your asset definition &mdash; it means that the entity wasn't able to download the required asset due to asset filter restrictions.
You can review the filters for an asset by using the sensuctl `asset info` command with a `--format` flag:

{{< highlight shell >}}
sensuctl asset info sensu-plugins-disk-checks --format yaml
{{< /highlight >}}

or 

{{< highlight shell >}}
sensuctl asset info sensu-plugins-disk-checks --format json
{{< /highlight >}}

A common asset filter issue is conflating operating systems with the family they're a part of.
For example, although Ubuntu is part of the Debian family of Linux distributions, Ubuntu is not the same as Debian.
A practical example might be:

{{< highlight shell >}}
...
    - entity.system.platform == 'debian'
    - entity.system.arch == 'amd64'
{{< /highlight >}}

This would not allow an Ubuntu system to run the asset.

Instead, the asset filter should look like this:

{{< highlight shell >}}
...
    - entity.system.platform_family == 'debian'
    - entity.system.arch == 'amd64'
{{< /highlight >}}

or 

{{< highlight shell >}}
    - entity.system.platform == 'ubuntu'
    - entity.system.arch == 'amd64'
{{< /highlight >}}

This would allow the asset to be downloaded onto the target entity.

[1]: ../../reference/agent#operation
[2]: ../../installation/verify
[3]: ../securing-sensu/#sensu-agent-tls-authentication
[4]: https://etcd.io/docs/v3.4.0/op-guide/security/
[5]: ../../reference/agent/#restart-the-service
[6]: ../../reference/agent#events-post
[7]: https://dzone.com/articles/what-is-structured-logging
[9]: ../../reference/backend/#restart-the-service
[10]: ../../reference/assets/#asset-definition-multiple-builds
[11]: ../systemd-logs

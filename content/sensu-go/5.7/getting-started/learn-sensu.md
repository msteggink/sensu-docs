---
title: "Learn Sensu Go"
description: "Here’s everything you need to start learning Sensu Go, including how to set up our sandbox and your first three lesson plans. You'll learn how to create a monitoring event and event pipeline, as well as automate event production with the Sensu agent."
version: "5.7"
product: "Sensu Go"
---

In this tutorial, we'll download the Sensu sandbox and create a monitoring workflow with Sensu.

- [Set up the sandbox](#set-up-the-sandbox)
- [Lesson \#1: Create a monitoring event](#lesson-1-create-a-sensu-monitoring-event)
- [Lesson \#2: Create an event pipeline](#lesson-2-pipe-keepalive-events-into-slack)
- [Lesson \#3: Automate event production with the Sensu agent](#lesson-3-automate-event-production-with-the-sensu-agent)

---

## Set up the sandbox

**1. Install Vagrant and VirtualBox**

- [Download Vagrant](https://www.vagrantup.com/downloads.html)
- [Download VirtualBox](https://www.virtualbox.org/wiki/Downloads)

**2. Download the sandbox**

[Download from GitHub](https://github.com/sensu/sandbox/archive/master.zip) or clone the repository:

{{< highlight shell >}}
git clone https://github.com/sensu/sandbox && cd sandbox/sensu-go
{{< /highlight >}}

**3. Start Vagrant**

{{< highlight shell >}}
ENABLE_SENSU_SANDBOX_PORT_FORWARDING=1 vagrant up
{{< /highlight >}}

The Learn Sensu sandbox is a CentOS 7 virtual machine pre-installed with Sensu, InfluxDB, and Grafana.
It is intended for use as a learning tool; we do not recommend this tool as part of a production installation.
To install Sensu in production, please see the [installation guide](../../installation/install-sensu).
The sandbox startup process takes about five minutes.

_NOTE: The sandbox configures VirtualBox to forward TCP ports 3002 and 4002 from the sandbox virtual machine to the localhost to make it easier for you to interact with the sandbox dashboards. Dashboard links provided in this tutorial assume port forwarding from the VM to the host is active._

**4. SSH into the sandbox**

Thanks for waiting! To start using the sandbox:

{{< highlight shell >}}
vagrant ssh
{{< /highlight >}}

You should now have shell access to the sandbox and should be greeted with this prompt:  

{{< highlight shell >}}
[sensu_go_sandbox]$
{{< /highlight >}}

To exit out of the sandbox, use `CTRL`+`D`.
To erase and restart the sandbox, use `vagrant destroy` then `vagrant up`.
To reset the sandbox's Sensu configuration to the beginning of this tutorial, use `vagrant provision`.

_NOTE: The sandbox pre-configures sensuctl with the Sensu Go admin user, so you won't have to configure sensuctl each time you spin up the sandbox to try out a new feature. Before installing sensuctl outside of the sandbox, read the [first time setup reference](https://docs.sensu.io/sensu-go/5.0/sensuctl/reference/#first-time-setup) to learn how to configure sensuctl._  

---

## Lesson \#1: Create a Sensu monitoring event

First off, we'll make sure everything is working correctly by using the sensuctl command line tool.
We can use sensuctl to see that our Sensu backend instance has a single namespace, `default`, and two users: the default admin user and the user created for use by a Sensu agent.

{{< highlight shell >}}
sensuctl namespace list
  Name    
─────────
 default  

sensuctl user list
 Username       Groups       Enabled  
────────── ──────────────── ───────── 
admin      cluster-admins   true     
agent      system:agents    true    
{{< /highlight >}}

Sensu keeps track of monitored components as entities.
Let's start by using sensuctl to make sure Sensu hasn't connected to any entities yet:

{{< highlight shell >}}
sensuctl entity list
 ID   Class   OS   Subscriptions   Last Seen  
──── ─────── ──── ─────────────── ─────────── 
{{< /highlight >}}

Now we can go ahead and start the Sensu agent to start monitoring the sandbox:

{{< highlight shell >}}
sudo systemctl start sensu-agent
{{< /highlight >}}

We can use sensuctl to see that Sensu is now monitoring the sandbox entity:

{{< highlight shell >}}
sensuctl entity list
        ID          Class    OS          Subscriptions                  Last Seen            
────────────────── ─────── ─────── ───────────────────────── ─────────────────────────────── 
sensu-go-sandbox   agent   linux   entity:sensu-go-sandbox   2019-01-24 21:29:06 +0000 UTC  
{{< /highlight >}}

Sensu agents send keepalive events to help you monitor their status.
We can use sensuctl to see the keepalive events generated by the sandbox entity:

{{< highlight shell >}}
sensuctl event list
      Entity          Check                                       Output                                     Status   Silenced             Timestamp            
────────────────── ─────────── ──────────────────────────────────────────────────────────────────────────── ──────── ────────── ─────────────────────────────── 
sensu-go-sandbox   keepalive   Keepalive last sent from sensu-go-sandbox at 2019-01-24 21:29:06 +0000 UTC        0   false      2019-01-24 21:29:06 +0000 UTC 
{{< /highlight >}}

The sensu-go-sandbox keepalive event has status 0, meaning the agent is in an OK state and able to communicate with the Sensu backend.

We can also see the event and the entity in the [Sensu dashboard](http://localhost:3002).
Log in to the dashboard as the default admin user: username `admin` and password `P@ssw0rd!`.

## Lesson \#2: Pipe keepalive events into Slack

Now that we know the sandbox is working properly, let's get to the fun stuff: creating a workflow.
In this lesson, we'll create a workflow that sends keepalive alerts to Slack.
(If you'd rather not create a Slack account, you can skip ahead to [lesson 3](#lesson-3-automate-event-production-with-the-sensu-agent).)

**1. Get your Slack webhook URL**

If you're already an admin of a Slack, visit `https://YOUR WORKSPACE NAME HERE.slack.com/services/new/incoming-webhook` and follow the steps to add the Incoming WebHooks integration, choose a channel, and save the settings.
(If you're not yet a Slack admin, start [here](https://slack.com/get-started#create) to create a new workspace.)
After saving, you'll see your webhook URL under Integration Settings.

**2. Register the Sensu Slack handler asset**

[Assets][2] are shareable, reusable packages that make it easy to deploy Sensu plugins.
In this lesson, we'll use the [Sensu Slack handler asset][1] to power a `slack` handler.

Use sensuctl to register the [Sensu Slack handler asset][1].

{{< highlight shell >}}
sensuctl asset create sensu-slack-handler --url "https://github.com/sensu/sensu-slack-handler/releases/download/1.0.3/sensu-slack-handler_1.0.3_linux_amd64.tar.gz" --sha512 "68720865127fbc7c2fe16ca4d7bbf2a187a2df703f4b4acae1c93e8a66556e9079e1270521999b5871473e6c851f51b34097c54fdb8d18eedb7064df9019adc8"
{{< /highlight >}}

You should see a confirmation message from sensuctl.

{{< highlight shell >}}
Created
{{< /highlight >}}

The `sensu-slack-handler` asset is now ready to use with Sensu.
You can use sensuctl to see the complete asset definition.

{{< highlight shell >}}
sensuctl asset info sensu-slack-handler --format yaml
{{< /highlight >}}

_PRO TIP: You can use resources definition to create and update resources (like assets) using `sensuctl create --file filename.yaml`. See the [sensuctl docs][3] for more information._

**3. Create a Sensu Slack handler**

Open the `sensu-slack-handler.json` handler definition provided with the sandbox, and edit the definition to include your Slack channel, webhook URL, and the `sensu-slack-handler` asset.

{{< highlight shell >}}
"env_vars": [
  "KEEPALIVE_SLACK_WEBHOOK=https://hooks.slack.com/services/AAA/BBB/CCC",
  "KEEPALIVE_SLACK_CHANNEL=#monitoring"
],
"runtime_assets": ["sensu-slack-handler"]
{{< /highlight >}}

Now we can create a Slack handler named `keepalive` to process keepalive events.

{{< highlight shell >}}
sensuctl create --file sensu-slack-handler.json
{{< /highlight >}}

You can use sensuctl to see available event handlers.

{{< highlight shell >}}
sensuctl handler list
{{< /highlight >}}

You should see the `keepalive` handler.

{{< highlight shell >}}
  Name      Type   Timeout   Filters   Mutator                                                   Execute                                                                                                              Environment Variables                            Assets         
─────────── ────── ───────── ───────── ───────── ────────────────────────────────────────────────────────────────────────────────────────────────────────── ────────────────────────────────────────────────────────────────────────────────────────────────── ───────────────────── 
 keepalive   pipe         0                       RUN:  /usr/local/bin/sensu-slack-handler -c "${KEEPALIVE_SLACK_CHANNEL}" -w "${KEEPALIVE_SLACK_WEBHOOK}"   KEEPALIVE_SLACK_WEBHOOK=https://hooks.slack.com/services/XXX,KEEPALIVE_SLACK_CHANNEL=#monitoring   sensu-slack-handler  
{{< /highlight >}}

You should now see monitoring events in Slack indicating that the sandbox entity is in an OK state.

**4. Filter keepalive events**

Now that we're generating Slack alerts, let's reduce the potential for alert fatigue by adding a filter that only sends only warning, critical, and resolution alerts to Slack.

To accomplish this, we'll interactively add the built-in is_incident filter to the keepalive handler so we'll only receive alerts when the sandbox entity fails to send a keepalive event.

{{< highlight shell >}}
sensuctl handler update keepalive
{{< /highlight >}}

When prompted for the filters selection, enter `is_incident` to apply the incidents filter.

{{< highlight shell >}}
? Filters: [? for help] is_incident
{{< /highlight >}}

We can confirm that the keepalive handler now includes the incidents filter using sensuctl:

{{< highlight shell >}}
sensuctl handler info keepalive
=== keepalive
Name:                  keepalive
Type:                  pipe
Timeout:               0
Filters:               is_incident
{{< /highlight >}}

With the filter in place we should no longer be receiving messages in the Slack channel every time the sandbox entity sends a keepalive event.

Let's stop the agent and confirm that we receive the expected warning message.

{{< highlight shell >}}
sudo systemctl stop sensu-agent
{{< /highlight >}}

You should see the warning message in Slack after a couple of minutes, informing you that the sandbox entity is no longer sending keepalive events.

Before we go, start the agent to resolve the warning.

{{< highlight shell >}}
sudo systemctl start sensu-agent
{{< /highlight >}}

## Lesson \#3: Automate event production with the Sensu agent
So far we've used the Sensu agent's built-in keepalive feature, but in this lesson, we'll create a check that automatically produces workload-related events.
Instead of sending alerts to Slack, we'll store event data with [InfluxDB](https://www.influxdata.com/) and visualize it with [Grafana](https://grafana.com/).

**1. Make sure the Sensu agent is running**

{{< highlight shell >}}
sudo systemctl restart sensu-agent
{{< /highlight >}}

**2. Install Nginx and the Sensu HTTP Plugin**

We'll use the [Sensu HTTP Plugin](https://github.com/sensu-plugins/sensu-plugins-http) to monitor an Nginx server running on the sandbox.

First, install and start Nginx:

{{< highlight shell >}}
sudo yum install -y nginx && sudo systemctl start nginx
{{< /highlight >}}

And make sure it's working with:

{{< highlight shell >}}
curl -I http://localhost:80

HTTP/1.1 200 OK
{{< /highlight >}}

Then install the Sensu HTTP Plugin:

{{< highlight shell >}}
sudo sensu-install -p sensu-plugins-http
{{< /highlight >}}

We'll be using the `metrics-curl.rb` plugin.
We can test its output using:

{{< highlight shell >}}
/opt/sensu-plugins-ruby/embedded/bin/metrics-curl.rb -u "http://localhost"

...
sensu-go-sandbox.curl_timings.http_code 200 1535670975
{{< /highlight >}}


**3. Create an InfluxDB pipeline**
Now let's create the InfluxDB pipeline to store these metrics and visualize them with Grafana.
To create a pipeline to send metric events to InfluxDB, start by registering the [Sensu InfluxDB handler asset][4].

{{< highlight shell >}}
sensuctl asset create sensu-influxdb-handler --url "https://github.com/sensu/sensu-influxdb-handler/releases/download/3.1.2/sensu-influxdb-handler_3.1.2_linux_amd64.tar.gz" --sha512 "612c6ff9928841090c4d23bf20aaf7558e4eed8977a848cf9e2899bb13a13e7540bac2b63e324f39d9b1257bb479676bc155b24e21bf93c722b812b0f15cb3bd"
{{< /highlight >}}

You should see a confirmation message from sensuctl.

{{< highlight shell >}}
Created
{{< /highlight >}}

The `sensu-influxdb-handler` asset is now ready to use with Sensu.
You can use sensuctl to see the complete asset definition.

{{< highlight shell >}}
sensuctl asset info sensu-influxdb-handler --format yaml
{{< /highlight >}}

Open the `influx-handler.json` handler definition provided with the sandbox, and edit the `runtime_assets` attribute to include the `sensu-influxdb-handler` asset.

{{< highlight shell >}}
"runtime_assets": ["sensu-influxdb-handler"]
{{< /highlight >}}

Now you can use sensuctl to create the `influx-db` handler.

{{< highlight shell >}}
sensuctl create --file influx-handler.json
{{< /highlight >}}

We can use sensuctl to confirm that the handler has been created successfully.

{{< highlight shell >}}
sensuctl handler list
{{< /highlight >}}

You should see the `influx-db` handler.
(If you've completed [lesson \#2](#lesson-2-pipe-keepalive-events-into-slack), you'll also see the `keepalive` handler.)

**4. Create a check to monitor Nginx**

Use the `curl_timings-check.json` file provided with the sandbox to create a service check that runs `metrics-curl.rb` every 10 seconds on all entities with the `entity:sensu-go-sandbox` subscription and sends events to the InfluxDB pipeline:

{{< highlight shell >}}
sensuctl create --file curl_timings-check.json

sensuctl check list
     Name                                        Command                                     Interval   Cron   Timeout   TTL        Subscriptions        Handlers   Assets   Hooks   Publish?   Stdin?     Metric Format      Metric Handlers  
────────────── ──────────────────────────────────────────────────────────────────────────── ────────── ────── ───────── ───── ───────────────────────── ────────── ──────── ─────── ────────── ──────── ──────────────────── ───────────────── 
curl_timings   /opt/sensu-plugins-ruby/embedded/bin/metrics-curl.rb -u "http://localhost"         10                0     0   entity:sensu-go-sandbox                               true       false    graphite_plaintext   influx-db        
{{< /highlight >}}

This check defines a metrics handler and metric format.
In Sensu Go metrics are a core element of the data model, so we can build pipelines to handle metrics separately from alerts.
This allows us to customize our monitoring workflows to get better visibility and reduce alert fatigue.

After about 10 seconds, we can see the event produced by the entity:

{{< highlight shell >}}
sensuctl event info sensu-go-sandbox curl_timings --format json | jq .
...
  "history": [
    {
      "status": 0,
      "executed": 1556472457
    },
  ],
  "output": "sensu-go-sandbox.curl_timings.time_total 0.005 1556472657\n...",
  ...
  "output_metric_format": "graphite_plaintext",
  "output_metric_handlers": [
    "influx-db"
  ],
...
{{< /highlight >}}

Because we configured a metric format, the Sensu agent is able to convert the Graphite-formatted metrics provided by the check command into a set of Sensu-formatted metrics (not shown in the output), which are then sent to the InfluxDB handler that reads Sensu-formatted metrics and converts them to a format InfluxDB accepts.
Metric support isn't limited to just Graphite; the Sensu agent can extract metrics in multiple line protocol formats, including Nagios performance data.

**5. See the HTTP response code events for Nginx in [Grafana](http://localhost:4002/d/go01/sensu-go-sandbox).**

Log in to Grafana as username: `admin` and password: `admin`.
We should see a graph of live HTTP response codes for Nginx.

Now if we turn Nginx off, we should see the impact in Grafana:

{{< highlight shell >}}
sudo systemctl stop nginx
{{< /highlight >}}

Start Nginx:

{{< highlight shell >}}
sudo systemctl start nginx
{{< /highlight >}}

**6. Automate disk usage monitoring for the sandbox**

Now that we have an entity set up, we can easily add more checks.
For example, let's say we want to monitor disk usage on the sandbox.

First, install the plugin:

{{< highlight shell >}}
sudo sensu-install -p sensu-plugins-disk-checks
{{< /highlight >}}

And test it:

{{< highlight shell >}}
/opt/sensu-plugins-ruby/embedded/bin/metrics-disk-usage.rb

sensu-core-sandbox.disk_usage.root.used 2235 1534191189
sensu-core-sandbox.disk_usage.root.avail 39714 1534191189
...
{{< /highlight >}}

Then create the check using sensuctl and the `disk_usage-check.json` file included with the sandbox, assigning it to the `entity:sensu-go-sandbox` subscription and the InfluxDB pipeline:

{{< highlight shell >}}
sensuctl create --file disk_usage-check.json
{{< /highlight >}}

We should see it working in the [dashboard entity view](http://localhost:3002/#/entities) and via sensuctl:

{{< highlight shell >}}
sensuctl event list
{{< /highlight >}}

Now we should be able to see disk usage metrics for the sandbox in [Grafana](http://localhost:4002/d/go02/sensu-go-sandbox-combined).

You made it! You're ready for the next level of Sensu-ing.
Here are some resources to help continue your journey:

- [Install Sensu Go](../../installation/install-sensu)
- [Collect StatsD metrics](../../guides/aggregate-metrics-statsd)
- [Create a ready-only user](../../guides/create-read-only-user/)

[1]: https://bonsai.sensu.io/assets/sensu/sensu-slack-handler
[2]: ../../reference/assets
[3]: ../../sensuctl/reference#creating-resources
[4]: https://bonsai.sensu.io/assets/sensu/sensu-influxdb-handler
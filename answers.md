# Introduction
DataDog is a modern monitoring and analytics service that can be used with any app, containing any stack, at any scale. Below I will demonstrate how to use the DataDog Agent to monitor data, customize your dashboard and send email notifications to alert the user of changes.

# Setting up the DataDog Agent
I ran the DataDog Agent on OSX and Ubuntu. Installing the DataDog agent is very simple for both operating systems.

**MacOS**
The Agent for Mac can be installed as easily as running `DD_API_KEY={API_KEY} bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_mac_os.sh)"` in your terminal. The DataDog project can then be found at `/opt/datadog-agent/`.

**Ubuntu**
The Agent is just as easy to install on an Ubunut environment by running `DD_API_KEY={API_KEY} bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_script.sh)"` in this Ubuntu terminal. To access the project here, you first have to `cd /vagrant` and then the DataDog project can be found again at `/opt/datadog-agent/`.

# Getting Started
Before getting started using DataDog, I recommend taking a look through the [DataDog docs](https://docs.datadoghq.com/getting_started/). There are useful [Videos](https://docs.datadoghq.com/videos/) and [API Documentation](https://docs.datadoghq.com/api/). There is also a great video tour on the homepage of the [DataDog Website](https://www.datadoghq.com/).

# Collecting Metrics

**Tags**
The backbone of DataDog's service is the metrics that it collects from your hosts. It is important to add tags to organize your metrics across a large spectrum of hosts. Tags can be added in the yaml file located at `/opt/datadog-agent/etc/datadog.yaml` simply by adding a line like `tags: region:eastus, purpose:hiring_exercise`. The recommended format is `key:value` like `region:eastus`. You can see here that the keys, region and purpose, are listed at the top of the screen and their values, eastus and hiring_exercise, are surrounding your host.

![tags](tags.png?raw=true "Tags")

![tags](tags2.png?raw=true "Tags")

**Hostmap Link**
https://app.datadoghq.com/infrastructure/map?fillby=avg%3Acpuutilization&sizeby=avg%3Anometric&groupby=none&nameby=name&nometrichosts=false&tvMode=false&nogrouphosts=false&palette=green_to_orange&paletteflip=false&node_type=host


**PostgreSQL Integration**
Now that we have applied our tags to our host we can start setting up integrations for any applications we use. I use the [PostgreSQL DB](https://www.postgresql.org/about/) primarily so I set up an integration for Postgres. First, create a `postgres.yaml` file in the Agent's `conf.d` directory. The steps for finishing this setup are located in the [DataDog Integrations Docs](https://docs.datadoghq.com/integrations/postgres/). After this is done, your configuration should look something like:
```yaml
init_config:

instances:
  - host: localhost
    port: 5432
    username: datadog
    password: 5E5d2JYsANZZGDrlvy7N3drz
    tags:
    	-optional_tag
    	-optional_tag_2
```
Once successfully configured, you will see and message like the following in your Events Stream in the UI and `postgresql` will appear in your host map.

![PSQL](postgres.png?raw=true "PSQL")
![PSQL Host Map](psql_hostmap.png?raw=true "PSQL Host Map")

**Custom Agent Check**
To start an agent check, you need to have an empty config file and a python script to run the check. I added the `min_collection_interval: 45` to the config to update it from the default, 20 seconds, to 45 seconds. The metric created in this case is called `my_metric` which returns a random integer from 1 to 1000.

```python
# my_check.py
import random

from checks import AgentCheck

class HelloCheck(AgentCheck):
    def check(self, instance):
    	n = random.randint(1, 1000)
        self.gauge('my_metric', n)
```

```yaml
# my_check.yaml
init_config:
  min_collection_interval: 45

instances:
    [{}]
```

![Agent Check](check.png?raw=true "Agent Check")

**Bonus - Can I change collection interval without updating Python?**
Yes! You can change the collection interval in the .yaml file by adding a min_collection
interval to the init_config. This, however, doesn't mean that data is sent every 45 seconds. The DataDog Agent collects data every 20 seconds by default. This changes that collection interval for my_metric only so data is only collected after 45 seconds has passed. Since the DataDog Agent collects data every 20 seconds, it will skip my_metric after 20 seconds, skip it again after 40 seconds, and then collect it upon reaching 60 seconds.

# Vizualizing Data
**Timeboards**
We have already seen a few timeboards in the Dashboard of the DataDog UI, but how do we create a custom one? To start, everything you need to know about creating a Timeboard in DataDog can be found the [API Docs](https://docs.datadoghq.com/api/?lang=python#timeboards). 
To start, you will need to add your API Key and your App Key to the `options`. An App Key can be generated by visiting 'APIs' under the Integrations tab in the UI. You just assign the key a name and click `Create Application Key`.
One of the key parts of this script is to create proper requests. The proper grammar needed to create these requests can be found [here](https://docs.datadoghq.com/graphing/miscellaneous/graphingjson/#grammar).

**Create Timeboard Script**

```python
# create_timeboard.py
from datadog import initialize, api

options = {
    'api_key': 'API_KEY',
    'app_key': 'APP_KEY'
}

initialize(**options)

title = "Hiring Exercise Timeboard"
description = "Hiring Exercise Timeboard"
graphs = [{
	"definition": {
		"events": [],
		"requests": [
			{
				"q": "avg:my_metric{*}"
			},
			{
				"q": "anomalies(avg:postgresql.buffer_hit{*}*100, 'basic', 2)"
			},
			{
				"q": "sum:my_metric{*}.rollup(3600)"
			}
		],
		"viz": "timeseries"
	},
	"title": "Visualizing Data"
}]

template_variables = [{
    "name": "host1",
    "prefix": "host",
    "default": "host:my-host"
}]

read_only = True
api.Timeboard.create(title=title,
                     description=description,
                     graphs=graphs,
                     template_variables=template_variables,
                     read_only=read_only)
```

**Viewing my_metric**
`"q": "avg:my_metric{*}"` graphs the average value of my_metric over all hosts and tags (or my single host).
`"q": "sum:my_metric{*}.rollup(3600)"`takes the sum of my metric over the last hour (3600 seconds) and rolls it up into a single point.
`"q": "anomalies(avg:postgresql.buffer_hit{*}*100, 'basic', 2)"` shows the anomalies for buffers hit. Basic uses a simple lagging rolling quantile computation to determine the range of expected values, but it uses very little data and adjusts quickly to changing conditions but has no knowledge of seasonal behavior or longer trends. I multiplied buffers hit by 100 to have big enough data to be seen in the grap relative to my_metric. The anomalies function will highlight any sudden drops or spikes in red.

**Timeboard Link**
https://app.datadoghq.com/dash/639317/hiring-exercise-timeboard?live=true&page=0&is_auto=false&from_ts=1520460638441&to_ts=1520464238441&tile_size=xl

**Snapshot sent with @ notation**
![snapshot](snapshot.png?raw=true "snapshot")

**Bonus - What is the Anomaly graph displaying**
There is a grey box around it showing where it has dropped or increased. It would show a highlighted red if there was a big drop or spike.

# Monitoring Data
Now that our data is being properly represented, we will want to know if anything drastic happens without having to watch the monitor at all times. This is where alerts come in. Creating the initial monitor isn't too different from creating the timeboard. You will have to create a new App Key following the same steps you took to do so for your timeboard. After `initialize(**options)` we want to add a few options to notify us of a few things. First, you can add `thresholds` in dictionary format to declare at what point you want to send a critical alert and a warning alert. Next, set the boolean value of `notify_no_data` to `True` and set `no_data_timeframe` to `20`.
There are two ways to create an alert: using the [API](https://docs.datadoghq.com/api/?lang=python#monitors) and in the UI. To do so using the API, first set your alert type to `type="metric alert"`. Then create a query inside of `api.Monitor.create` of format `time_aggr(time_window):space_aggr:metric{tags}`. `time_aggr` can be set equal to avg, sum, max, min, change or pct_change. `time_window` can be set in minutes, hours or days so we set it to `last_5m`. `space_aggr` takes the same arguments as `time_aggr` and we want our metric to be `my_metric` over all hosts and tags. Then name your monitor. To set your message, choose a `handle` (or email address) using @ notation and set the warning message equal to `message`.
Updating your monitor in the UI is extremely simple. After running the script, you should see a monitor 'my_metric Monitor' under 'Manage Monitors' in the Monitors tab. If you go to the 'Edit' tab you will see your Alert and Warning Thresholds are already declared. To add a custom message for each threshold you can use Jinja logic in the fourth field to do so and set your recipients in the fifth.

![alert](alert_message.png?raw=true "alert")

**Create Monitor Script**
```python
# create_metric_monitor.py
from datadog import initialize, api

options = {
    'api_key': 'API_KEY',
    'app_key': 'APP_KEY'
}

# Create Monitor with thresholds
initialize(**options)

options = {
	"thresholds": {
		"critical": 800,
		"warning": 500
	},
    "notify_no_data": True,
    "no_data_timeframe": 20
}

api.Monitor.create(
    type="metric alert",
    query="avg(last_5m):avg:my_metric{*} > 500",
    name="my_metric Monitor",
    handle="tylerpwilmot@gmail.com",
    message="WARNING: my_metric is over 500! Alert is at 800.",
    options=options
)
```

**Warning Notification**

![warning](warning.png?raw=true "warning")

**No Data Notification**

![nodata](nodata.png?raw=true "nodata")

**Bonus**
**Silenced Notifications**
This monitor will send an alert pretty often, so I set downtimes for weekdays after work and weekends. This can also be done easily using the UI by going to the 'Monitors' tab and selecting 'Manage Downtime'. From here, you can choose what hours you would like to snooze your alerts and whether or not you want it to be a one time occurance or recurring. Also, you can include a message and tag yourself so you will get an email letting you know when downtimes begin and end.

![silenced](silenced.png?raw=true "silenced")
![silenced2](silenced2.png?raw=true "silenced2")

**Downtime Link**
https://app.datadoghq.com/monitors#downtime?id=296173286

# Collecting APM Data
Tracing is a more in depth version of monitoring. It provides a much more in depth performance analysis and automatically generates dashboards monitoring key metrics. 
I had some trouble getting this question to work. Initially I tried running ddtrace locally using OSX like I did with the previous problems but had issues with the DataDog Agent configuration.
`./trace-agent-osx-X.Y.Z -config /opt/datadog-agent/etc/datadog.conf`
To troubleshoot, I tried taking ownership of the file and running the source code. Next, I installed the Vagrant Ubuntu VM and installed the DataDog Agent there using my API Key. I copied the Flask app into vim so I had it in the VM. At first, it wouldn't allow me to install ddtrace with pip. In this environment, it would only install pip 1.0. pip 1.0 only points to the insecure pypi.python.org location so I had to work around that to hit its secure location.
`sudo pip install -v ddtrace -i https://pypi.python.org/simple/`
This points pip to the https address for pypiy ddtrace. Next, I had to install Flask. The virtualenv installed easily but pip 1.0 wouldn't install Flask for the same issues. I tried the workaround I worked out before but I was getting similar errors. Without Flask installed I couldn't ddtrace the Flask application. If I had a little more time I think I could have gotten this up and running. I noticed that the VM I was using was running Ubuntu 12.4 which was very out of date - probably why it would only support pip 1.0. In the future, I would try to run it on a more up to date version of Ubuntu.

**Bonus - What is the difference between a Service and a Resource**
A Service is a set of processes that work together to provide a feature set. For example, an application may have two services: **webapp** and **database**. A Resource is a particular query to a Service. For a SQL database, the resource would be the SQL query of itself.

# Final Question
I would use DataDog to monitor activity on applications like OpenTable or Resy to see what reservation trends are like seasonally and how reservations are affected by the weather. It would be cool to have concrete data on how dining trends are changing during different seasons in different parts of the country.

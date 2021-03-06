..
.. Copyright 2015 Comcast Cable Communications Management, LLC
..
.. Licensed under the Apache License, Version 2.0 (the "License");
.. you may not use this file except in compliance with the License.
.. You may obtain a copy of the License at
..
..     http://www.apache.org/licenses/LICENSE-2.0
..
.. Unless required by applicable law or agreed to in writing, software
.. distributed under the License is distributed on an "AS IS" BASIS,
.. WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
.. See the License for the specific language governing permissions and
.. limitations under the License.
..

****************************
Traffic Stats Administration
****************************

Traffic Stats actually consists of three seperate components:  Traffic Stats, InfluxDB, and Grafana.  See below for information on installing and configuring each component as well as configuring the integration between the three and Traffic Ops.

Installation
========================

**Installing Traffic Stats:**

	- Download the Traffic Stats RPM from the traffic control `downloads <http://traffic-control-cdn.net/downloads/index.html>`_ page.
	- Copy the Traffic Stats RPM to your server
	- sudo rpm -ivh <traffic_stats rpm>

**Installing InfluxDB:**

	**As of Traffic Stats 1.4.0, InfluxDb 0.9.6 or higher is required.  For InfluxDb versions less than 0.9.6 use Traffic Stats 1.3.0**

	In order to store traffic stats data you will need to install InfluxDB.  It is recommended InfluxDB be installed in a 3 server cluster; VMs are acceptable. The documentation for installing InfluxDB can be found on the InfluxDB `website <https://docs.influxdata.com/influxdb/latest/introduction/installation/>`_.


**Installing Grafana:**

	Grafana is used to display Traffic Stats/InfluxDB data in Traffic Ops.  Grafana is typically run on the same server as Traffic Stats but this is not a requirement.  Grafana can be installed on any server that can access InfluxDB and be accessed by Traffic Ops.  Documentation on installing Grafana can be found `here <http://docs.grafana.org/installation/>`_.

Configuration
=========================

**Configuring Traffic Stats:**

	Traffic Stats' configuration file can be found in /opt/traffic_stats/conf/traffic_stats.cfg.
	The following values need to be configured:

	     - *toUser:* The user used to connect to Traffic Ops
	     - *toPasswd:*  The password to use when connecting to Traffic Ops
	     - *toUrl:*  The URL of the Traffic Ops server used by Traffic Stats
	     - *influxUser:*  The user to use when connecting to InfluxDB (if configured on InfluxDB, else leave default)
	     - *influxPassword:*  That password to use when connecting to InfluxDB (if configured, else leave blank)
	     - *polling interval:*  The interval at which Traffic Monitor is polled and stats are stored in InfluxDB
	     - *statusToMon:*  The status of Traffic Monitor to poll (poll ONLINE or OFFLINE traffic monitors)
	     - *seelogConfig:*  The absolute path of the seelong config file
	     - *dailySummaryPollingInterval:* The interval, in seconds, at which Traffic Stats checks to see if daily stats need to be computed and stored.
	     - *cacheRetentionPolicy:* The default retention policy for cache stats
	     - *dsRetentionPolicy:* The default retention policy for deliveryservice stats
	     - *dailySummaryRetentionPolicy:* The retention policy to be used for the daily stats

**Configuring InfluxDB:**

	It is HIGHLY recommended that InfluxDB be configured for clustering.  Documentation on clustering configuration can be found on the clustering page of the `InfluxDB Website <https://docs.influxdata.com/influxdb/latest/clustering/>`_.

	Once InfluxDB is installed and clustering is configured, Databases and Retention Policies need to be created.  Traffic Stats writes to three different databases: cache_stats, deliveryservice_stats, and daily_stats.  More information about the databases and what data is stored in each can be found on the `overview <../overview/traffic_stats.html>`_ page.

	To easily create databases, retention policies, and continuous queries, run create_ts_databases.go from the influxdb_tools directory.  See the `InfluxDb Tools <traffic_stats.html#influxdb-tools>`_ section below for more information.

**Configuring Grafana:**

		In Traffic Ops the Health -> Graph View tab can be configured to display grafana graphs using influxDb data.  In order for this to work correctly, you will need two things 1) a parameter added to traffic ops with the graph URL (we will discuss later) and 2) the graphs created in grafana.  See below for how to create some simple graphs in grafana.  These instructions assume that InfluxDB has been installed and conifugred and that data has been written to it.  If this is not true, you will not see any graphs.

		- Login to grafana as an admin user http://grafana_url:3000/login
		- Choose Data Sources and then Add New
		- Enter the necessary information to configure your data source
		- Click on the 'Home' dropdown at the top of the screen and choose New at the bottom
		- Click on the green menu bar (with 3 lines) at the top and choose Add Panel -> Graph
		- Where it says 'No Title (click here)' click and choose edit
		- Choose your data source at the bottom
		- You can have grafana help you create a query, or you can create your own.  Here is a sample query:

			``SELECT sum(value)*1000 FROM "monthly"."bandwidth.cdn.1min" WHERE $timeFilter GROUP BY time(60s), cdn``
		- Once you have the graph the way you want it, click the 'Save Dashboard' button at the top
		- You should now have a new saved graph

	In order for Traffic Ops users to see Grafana graphs, Grafana will need to allow anonymous access.  Information on how to configure anonymous access can be found on the configuration page of the `Grafana Website  <http://docs.grafana.org/installation/configuration/#authanonymous>`_.

	Traffic Ops uses custom dashboards to display information about individual delivery services or cache groups.  In order for the custom graphs to display correctly, the `traffic_ops_*.js <https://github.com/Comcast/traffic_control/blob/master/traffic_stats/grafana/>`_ files need to be in the ``/usr/share/grafana/public/dashboards/`` directory on the grafana server.  If your Grafana server is the same as your Traffic Stats server the RPM install process will take care of putting the files in place.  If your grafana server is different from your Traffic Stats server, you will need to manually copy the files to the correct directory.

	More information on custom scripted graphs can be found in the `scripted dashboards <http://docs.grafana.org/reference/scripting/>`_ section of the Grafana documentation.

**Configuring Traffic Ops for Traffic Stats:**

	- The influxDb servers need to be added to Traffic Ops with profile = InfluxDB.  Make sure to use port 8086 in the configuration.
	- The traffic stats server should be added to Traffic Ops with profile = Traffic Stats.
	- Parameters for which stats will be collected are added with the release, but any changes can be made via parameters that are assigned to the Traffic Stats profile.

**Configuring Traffic Ops to use Grafana Dashboards**

	To configure Traffic Ops to use Grafana Dashboards, you need to enter the following parameters and assign them to the GLOBAL profile.  This assumes you followed the above instructions to install and configure InfluxDB and Grafana.  You will need to place 'cdn-stats','deliveryservice-stats', and 'daily-summary' with the name of your dashboards.

	+---------------------------+------------------------------------------------------------------------------------------------+
	|       parameter name      |                                        parameter value                                         |
	+===========================+================================================================================================+
	| all_graph_url             | https://<grafana_url>/dashboard/db/deliveryservice-stats                                       |
	+---------------------------+------------------------------------------------------------------------------------------------+
	| cachegroup_graph_url      | https://<grafanaHost>/dashboard/script/traffic_ops_cachegroup.js?which=                        |
	+---------------------------+------------------------------------------------------------------------------------------------+
	| deliveryservice_graph_url | https://<grafanaHost>/dashboard/script/traffic_ops_devliveryservice.js?which=                  |
	+---------------------------+------------------------------------------------------------------------------------------------+
	| server_graph_url          | https://<grafanaHost>/dashboard/script/traffic_ops_server.js?which=                            |
	+---------------------------+------------------------------------------------------------------------------------------------+
	| visual_status_panel_1     | https://<grafanaHost>/dashboard-solo/db/cdn-stats?panelId=2&fullscreen&from=now-24h&to=now-60s |
	+---------------------------+------------------------------------------------------------------------------------------------+
	| visual_status_panel_2     | https://<grafanaHost>/dashboard-solo/db/cdn-stats?panelId=1&fullscreen&from=now-24h&to=now-60s |
	+---------------------------+------------------------------------------------------------------------------------------------+
	| daily_bw_url              | https://<grafanaHost>/dashboard-solo/db/daily-summary?panelId=1&fullscreen&from=now-3y&to=now  |
	+---------------------------+------------------------------------------------------------------------------------------------+
	| daily_served_url          | https://<grafanaHost>/dashboard-solo/db/daily-summary?panelId=2&fullscreen&from=now-3y&to=now  |
	+---------------------------+------------------------------------------------------------------------------------------------+

InfluxDb Tools
=========================

Under the Traffic Stats source directory there is a directory called influxdb_tools.  These tools are meant to be used as one-off scripts to help a user quickly get new databases and continuous queries setup in influxdb.
They are specific for traffic stats and are not meant to be generic to influxdb.  Below is an brief description of each script along with how to use it.

**create_ts_databases**
	This script creates all `databases <https://docs.influxdata.com/influxdb/latest/concepts/key_concepts/#database>`_, `retention policies <https://docs.influxdata.com/influxdb/latest/concepts/key_concepts/#retention-policy>`_, and `continuous queries <https://docs.influxdata.com/influxdb/v0.11/query_language/continuous_queries/>`_ required by traffic stats.

	**How to use create_ts_databases:**

	Pre-Requisites:

		1. Go 1.4 or later
		2. configured $GOPATH (e.g. export GOPATH=~/go)

	Using create_ts_databases.go

		1. go get github.com/influxdata/influxdb

		2. go build create_ts_databases.go

		3. Run it:
			- ./create_ts_databases -help
			- optional flags:
				- influxUrl -  The influxdb url and port
				- replication -  The number of nodes in the cluster
				- user - The user to use
				- password - The password to use
			- example: ./create_ts_databases -influxUrl=localhost:8086 -replication=3 -user=joe -password=mysecret

**sync_ts_databases**
	This script is used to sync one influxdb environment to another.  Only data from continuous queries is synced as it is downsampled data and much smaller in size than syncing raw data.  Possible use cases are syncing from Production to Development or Syncing a new cluster once brought online.

	**How to use sync_ts_databases:**

	Pre-Requisites:

		1. Go 1.4 or later
		2. configured $GOPATH (e.g. export GOPATH=~/go)

	Using sync_ts_databases.go:

		1. go get github.com/influxdata/influxdb

		2. go build sync_ts_databases.go

		3. Run it
			- ./sync_ts_databases -help
			- required flags:
				- sourceUrl - The URL of the source database
				- targetUrl - The URL of the target database

			-optional flags:
				- database - The database to sync (default = sync all databases)
				- days - Days in the past to sync (default = sync all data)
				- sourceUser - The user of the source database
				- sourcePass - The password for the source database
				- targetUser - The user of the target database
				- targetPass - The password for the target database

			- example: ./sync_ts_databases -sourceUrl=http://influxdb-production-01.kabletown.net:8086 -targetUrl=http://influxdb-dev-01.kabletown.net:8086 -database=cache_stats -days=7 -sourceUser=joe sourcePass=mysecret


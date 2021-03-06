ElastAlert - Easy & Flexible Alerting With ElasticSearch
********************************************************

ElastAlert is a simple framework for alerting on anomalies, spikes, or other patterns of interest from data in Elasticsearch.

At Yelp, we use Elasticsearch, Logstash and Kibana for managing our ever increasing amount of data and logs.
Kibana is great for visualizing and querying data, but we quickly realized that it needed a companion tool for alerting
on inconsistencies in our data. Out of this need, ElastAlert was created.

If you have data being written into Elasticsearch in near real time and want to be alerted when that data matches certain patterns, ElastAlert is the tool for you.

Overview
========

We designed ElastAlert to be :ref:`reliable <reliability>`, highly :ref:`modular <modularity>`, and easy to :ref:`set up <tutorial>` and :ref:`configure <configuration>`.

It works by combining Elasticsearch with two types of components, rule types and alerts.
Elasticsearch is periodically queried and the data is passed to the rule type, which determines when
a match is found. When a match occurs, it is given to one or more alerts, which take action based on the match.

This is configured by a set of rules, each of which defines a query, a rule type, and a set of alerts.

Several rule types with common monitoring paradigms are included with ElastAlert:

- "Match where there are X events in Y time" (``frequency`` type)
- "Match when the rate of events increases or decreases" (``spike`` type)
- "Match when there are less than X events in Y time" (``flatline`` type)
- "Match when a certain field matches a blacklist/whitelist" (``blacklist`` and ``whitelist`` type)
- "Match on any event matching a given filter" (``any`` type)
- "Match when a field has two different values within some time" (``change`` type)

Currently, we have support built in for these alert types:

- Command
- Email
- JIRA
- OpsGenie
- SNS
- HipChat
- Slack
- Telegram
- Debug

Additional rule types and alerts can be easily imported or written. (See :ref:`Writing rule types <writingrules>` and :ref:`Writing alerts <writingalerts>`)

In addition to this basic usage, there are many other features that make alerts more useful:

- Alerts link to Kibana dashboards
- Aggregate counts for arbitrary fields
- Combine alerts into periodic reports
- Separate alerts by using a unique key field
- Intercept and enhance match data

To get started, check out :ref:`Running ElastAlert For The First Time <tutorial>`.

.. _reliability:

Reliability
===========

ElastAlert has several features to make it more reliable in the event of restarts or Elasticsearch unavailability:

- ElastAlert :ref:`saves its state to Elasticsearch <metadata>` and, when started, will resume where previously stopped
- If Elasticsearch is unresponsive, ElastAlert will wait until it recovers before continuing
- Alerts which throw errors may be automatically retried for a period of time

.. _modularity:

Modularity
==========

ElastAlert has three main components that may be imported as a module or customized:

Rule types
----------

The rule type is responsible for processing the data returned from Elasticsearch. It is initialized with the rule configuration, passed data
that is returned from querying Elasticsearch with the rule's filters, and outputs matches based on this data. See :ref:`Writing rule types <writingrules>`
for more information.

Alerts
------

Alerts are responsible for taking action based on a match. A match is generally a dictionary containing values from a document in Elasticsearch,
but may contain arbitrary data added by the rule type. See :ref:`Writing alerts <writingalerts>` for more information.

Enhancements
------------

Enhancements are a way of intercepting an alert and modifying or enhancing it in some way. They are passed the match dictionary before it is given
to the alerter. See :ref:`Enhancements` for more information.

.. _configuration:

Configuration
==============

ElastAlert has a global configuration file, ``config.yaml``, which defines several aspects of its operation:

``buffer_time``: ElastAlert will continuously query against a window from the present to ``buffer_time`` ago.
This way, logs can be back filled up to a certain extent and ElastAlert will still process the events. This
may be overridden by individual rules. This option is ignored for rules where ``use_count_query`` or ``use_terms_query``
 is set to true. Note that back filled data may not always trigger count based alerts as if it was queried in real time.

``es_host``: The host name of the Elasticsearch cluster where ElastAlert records metadata about its searches.
When ElastAlert is started, it will query for information about the time that it was last run. This way,
even if ElastAlert is stopped and restarted, it will never miss data or look at the same events twice. It will also specify the default cluster for each rule to run on.

``es_port``: The port corresponding to ``es_host``.

``use_ssl``: Optional; whether or not to connect to ``es_host`` using SSL; set to ``True`` or ``False``.

``es_username``: Optional; basic-auth username for connecting to ``es_host``.

``es_password``: Optional; basic-auth password for connecting to ``es_host``.

``es_url_prefix``: Optional; URL prefix for the Elasticsearch endpoint.

``es_send_get_body_as``: Optional; Method for querying Elasticsearch - ``GET``, ``POST`` or ``source``. The default is ``GET``

``es_conn_timeout``: Optional; sets timeout for connecting to and reading from ``es_host``; defaults to ``10``.

``rules_folder``: The name of the folder which contains rule configuration files. ElastAlert will load all
files in this folder, and all subdirectories, that end in .yaml. If the contents of this folder change, ElastAlert will load, reload
or remove rules based on their respective config files.

``scan_subdirectories``: Optional; Sets whether or not ElastAlert should recursively descend the rules directory - ``true`` or ``false``. The default is ``true``

``run_every``: How often ElastAlert should query Elasticsearch. ElastAlert will remember the last time
it ran the query for a given rule, and periodically query from that time until the present. The format of
this field is a nested unit of time, such as ``minutes: 5``. This is how time is defined in every ElastAlert
configuration.

``writeback_index``: The index on ``es_host`` to use.

``max_query_size``: The maximum number of documents that will be downloaded from Elasticsearch in a single query. The
default is 10,000, and if you expect to get near this number, consider using ``use_count_query`` for the rule. If this
limit is reached, ElastAlert will `scroll <https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-scroll.html>`_ through pages the size of ``max_query_size`` until processing all results.

``scroll_keepalive``: The maximum time (formatted in `Time Units <https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#time-units>`_) the scrolling context should be kept alive. Avoid using high values as it abuses resources in ElasticSearch, but be mindful to allow sufficient time to finish processing all the results. 

``max_aggregation``: The maximum number of alerts to aggregate together. If a rule has ``aggregation`` set, all
alerts occuring within a timeframe will be sent together. The default is 10,000.

``old_query_limit``: The maximum time between queries for ElastAlert to start at the most recently run query.
When ElastAlert starts, for each rule, it will search ``elastalert_metadata`` for the most recently run query and start
from that time, unless it is older than ``old_query_limit``, in which case it will start from the present time. The default is one week.

``disable_rules_on_error``: If true, ElastAlert will disable rules which throw uncaught (not EAException) exceptions. It
will upload a traceback message to ``elastalert_metadata`` and if ``notify_email`` is set, send an email notification. The
rule will no longer be run until either ElastAlert restarts or the rule file has been modified. This defaults to True.

``notify_email``: An email address, or list of email addresses, to which notification emails will be sent. Currently,
only an uncaught exception will send a notification email. The from address, SMTP host, and reply-to header can be set
using ``from_addr``, ``smtp_host``, and ``email_reply_to`` options, respectively. By default, no emails will be sent.

``from_addr``: The address to use as the from header in email notifications.
This value will be used for email alerts as well, unless overwritten in the rule config. The default value
is "ElastAlert".

``smtp_host``: The SMTP host used to send email notifications. This value will be used for email alerts as well,
unless overwritten in the rule config. The default is "localhost".

``email_reply_to``: This sets the Reply-To header in emails. The default is the recipient address.

``aws_region``: This makes ElastAlert to sign HTTP requests when using Amazon ElasticSearch Service. It'll use instance role keys to sign the requests.

``boto_profile``: Boto profile to use when signing requests to Amazon ElasticSearch Service, if you don't want to use the instance role keys.

.. _runningelastalert:

Running ElastAlert
==================

``$ python elastalert/elastalert.py``

Several arguments are available when running ElastAlert:

``--config`` will specify the configuration file to use. The default is ``config.yaml``.

``--debug`` will run ElastAlert in debug mode. This will increase the logging verboseness, change
all alerts to ``DebugAlerter``, which prints alerts and suppresses their normal action, and skips writing
search and alert metadata back to Elasticsearch.

``--start <timestamp>`` will force ElastAlert to begin querying from the given time, instead of the default,
querying from the present. The timestamp should be ISO8601, e.g.  ``YYYY-MM-DDTHH:MM:SS`` (UTC) or with timezone
``YYYY-MM-DDTHH:MM:SS-08:00`` (PST). Note that if querying over a large date range, no alerts will be
sent until that rule has finished querying over the entire time period. To force querying from the current time, use "NOW".

``--end <timestamp>`` will cause ElastAlert to stop querying at the specified timestamp. By default, ElastAlert
will periodically query until the present indefinitely.

``--rule <rule.yaml>`` will only run the given rule. The rule file may be a complete file path or a filename in ``rules_folder``
or its subdirectories.

``--silence <unit>=<number>`` will silence the alerts for a given rule for a period of time. The rule must be specified using
``--rule``. <unit> is one of days, weeks, hours, minutes or seconds. <number> is an integer. For example,
``--rule noisy_rule.yaml --silence hours=4`` will stop noisy_rule from generating any alerts for 4 hours.

``--verbose`` will increase the logging verboseness, which allows you to see information about the state
of queries.

``--es_debug`` will enable logging for all queries made to Elasticsearch.

``--es_debug_trace`` will enable logging curl commands for all queries made to Elasticsearch to a file.

``--end <timestamp>`` will force ElastAlert to stop querying after the given time, instead of the default,
querying to the present time. This really only makes sense when running standalone. The timestamp is formatted
as ``YYYY-MM-DDTHH:MM:SS`` (UTC) or with timezone ``YYYY-MM-DDTHH:MM:SS-XX:00`` (UTC-XX).

``--pin_rules`` will stop ElastAlert from loading, reloading or removing rules based on changes to their config files.

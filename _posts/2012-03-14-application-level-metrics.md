---
layout: post
title: A brief look at application level metrics
---

At work, like many organisations, we have a large collection of system metrics,
created by our techops teams but we (the developers) are less good at thinking
about application level metrics. I've decided to investigate different methods
for collecting metrics from our applications.

### A Very Quick Introduction To Application Metrics ###

Take a minute to think about monitoring and metric collection; what comes to mind...?
CPU usage? Memory usage? Free disk space?

These are all very important metrics to collect but they're obviously very
low level metrics.

There's an additional, higher level of metrics that can be collected; application
metrics. These could be things like:

- number of x processed (where x is an important function of the application)
- number of background jobs processed
- cache hits vs cache misses
- number of application errors

While system metrics can be used to notify you when something is about to,
or is already, going wrong; application metrics can be used as an actual pointer
to what specifically has gone wrong. Additionally, in continuous deployment
environments application metrics are often used as post-deploy health checks
for your application.

There are two obvious ways to collect metrics from applications:

- direct metric collection: adding calls into your applications to send metrics
directly to various metric collection systems
- log based metric collection: an alternative way to collect metrics from an
application would be to output events to log files and then parse those and
send values to metric collection systems

Most developers and organisations are used to logging important errors or
application events so using log based metric collection may be a lower friction
and lower cost approach to starting to collect application metrics. The downside
is that developers have to write various parsing scripts for different log formats.

Alternatively developers could litter their application with direct calls to metric
collection systems. This negates the need to parse log files but adds more noise
to the codebase, unless you
[utilise metaprogramming techniques](http://www.shopify.com/technology/3709232-statsd-at-shopify).

### Direct Metric Collection ###

#### Statsd ####

[StatsD](https://github.com/etsy/statsd) a tool by Etsy is described as:

    "A network daemon for aggregating statistics (counters and timers), rolling them up, then sending them to graphite."

StatsD is a network daemon that accepts packets over UDP (to minimise application
impact), which sends batch updates to Ganglia for trending and graphing.

There are client libraries available in many languages, including
[Perl](https://metacpan.org/module/Net::Statsd).

If you're starting to collect metrics from scratch, it's probably well worth
investigating StatsD (and graphite). However, at work, we already have the
infrastructure in place to collect metrics using Ganglia.

Short of modifying StatsD to send the metrics to Ganglia, or setting
up the infrastructure to collect metrics with Graphite (which I briefly tried
and failed at previously), we'll probably give StatsD a miss at work for now.

#### Gmetric ####

Ganglia has a command line client called gmetric that allows you to easily send
metric values straight to Ganglia (well roughly, there's some multi-node collection
details but they're not important in this discussion) and, again, there are
client libraries in many languages, including various
[implementations](https://metacpan.org/module/Ganglia::Gmetric)
[in](https://metacpan.org/module/Ganglia::Gmetric::PP)
[Perl](https://metacpan.org/module/Ganglia::Gmetric::XS).

Okay, sounds like a good alternative to StatsD. However, the thing that
attracted me to StatsD was the ability to increment counters, so every time
your application processed x you could fire off a StatsD packet incrementing the
counter, using Gmetric directly, you can't do this. You _have_ to send total values.
This would either require maintaining counters in your application, or sending
counts per time interval. This is now starting to sound less than ideal...

### Log Based Metric Collection ###

An alternative to direct metric collection from within an application is to parse
and filter the log files produced by the application.

#### Logster ####

[Logster](https://github.com/etsy/logster) is another Etsy tool, for tailing
log files and sending interesting info to ganglia or graphite. The idea is to
run it every x minutes under cron.

On the positive side: it's basic and simple, easy to install, and only
relies on one additional package (logcheck). It comes with a few example parsers
but you'll probably have to write your own custom parsers.

These are simple Python scripts that extend a base class.

The negatives are: I have to write Python :), the default configuration of logcheck
appears to send an hourly email about something or other (I haven't investigated
what exactly it's doing yet) and you'll probably end up with lots of grungy,
regex heavy, parser scripts lying around.

#### Logstash ####

[Logstash](http://logstash.net/) describes itself as tool to:

    "Ship logs from any source, parse them, get the right timestamp, index them, and search them."

Logstash is a bit like Logster but more comprehensive and feature heavy.
Conceptually, it considers the problem of gleaning metrics [from log files] to
have three parts:

- input: log files are just one of the input formats supported
- filter: parsing the data
- output: outputting the data somewhere for further processing or storage

You could still use it to output the data to Ganglia or Graphite, but it also
has its own web interface for log searching and viewing that uses an ElasticSearch
backend.

It also has baked in functionality to aggregate logs from multiple nodes.

Another positive, is that it uses a Ruby library called
[Grok](http://logstash.net/docs/1.0.17/filters/grok) to help write the
parsing scripts.  This allows you to write human readable, more reusable scripts.

A downside of logstash is that it's more complex. The requirements are either the
so called "monolithic JAR", which includes ElasticSearch, or MRI Ruby >= 1.9.2.

### Vague Conclusions ###

Time to draw some vague conclusions; I've already ruled out direct metric
collection, so that leaves me with the log parsing.

Logstash has the potential to be really useful but the time required to
investigate, configure and probably require help from others works against
its favour.

In our specific case (one application, running on one box) Logster looks like
the way forward because of its simplicity. It shouldn't take too long to get
it feeding Ganglia. We'll see...

### Further Reading ###

My little investigation has been quite brief and very specific to our situation;
for more general information about monitoring and metrics collection, take a look
at the following articles.

These are taken from https://github.com/monitoringsucks/:

- http://theoryandlogic.com/post/5890089120/the-ideal-monitoring-service
- http://www.conigliaro.org/2011/05/31/monitoring-as-code/
- http://lusislog.blogspot.com/2011/06/why-monitoring-sucks.html
- http://obfuscurity.com/2011/07/Monitoring-Sucks-Do-Something-About-It
- http://obfuscurity.com/2011/07/Robots-Are-Cool-and-Shit-But-Seriously
- http://lusislog.blogspot.com/2011/07/monitoring-sucks-watch-your-language.html

Some further links recommended by colleagues:

- http://www.jedi.be/blog/2012/01/03/monitoring-wonderland-survey-introduction/
- http://www.jedi.be/blog/2012/01/03/monitoring-wonderland-metrics-api-gateways/
- http://www.jedi.be/blog/2012/01/03/monitoring-wonderland-nagios-the-mighty-beast/
- http://www.jedi.be/blog/2012/01/04/monitoring-wonderland-moving-up-the-stack-application-user-metrics/

---
title: "The problem with Pingdom"
created_at: 2012-07-29 10:58:00 -0700
kind: article
---

I've been a Pingdom user for years now, as I'm sure most developers have. It's
ubiquitous; literally every organization I've worked with has used Pingdom in
some fashion.

It does a pretty good job - if your site is struck by another EC2 outage or a
critical hardware failure, Pingdom will very dutifully contact you in your
moment of need. It's basic, simple and works.

![Pingdom checks](/assets/images/2012-07-29-the-problem-with-pingdom/dashboard.png "Pingdom dashboard circa 2012")

*Pingdom dashboard circa 2012*

### Problem #1 - Outdated checks

Aside from the latest dashboard refresh, though, Pingdom hasn't changed much.
The same basic checks haven't changed with time:

* HTTP / HTTPS content match
* TCP & UDP port
* Ping
* DNS
* Email - SMTP / POP3 / IMAP

Who really uses email checks these days, anyways?

In my experience at Disqus, there's a big discrepancy in what I'd like to be
alerted on and what Pingdom can provide. Availability isn't just a matter of
receiving a **HTTP 200** header from one of our servers; consistently slower
response times are just as concerning as the binary UP/DOWN perspective that
Pingdom provides.

Also, as a third-party Javascript embeded service, a fast HTTP response from
our servers that still throws an exception (and breaks browser rendering) is
considered an outage. As sites move increasingly toward client-side rendering
your site could be totally unusable due to broken Javascript but Pingdom will
never alert you because the initial 200 response passes the content match.

*Solution*: The ideal uptime service would expand on these basic networking-/
regex-based checks and provide real insight into the functionality of your
site. As a site owner, I'd like to know immediately if a third-party widget
is throwing exceptions, and I'd like to be warned if my site is rendering
50% slower today than it has the previous 3 months.

Even the basic checks could be expanded to provide more useful alerting on
common cases every site faces:

* Domain expiration (an extension of DNS checks)
* SSL ceritificate expiration (an extension of HTTPS checks)
* Actual browser render times (an extension of HTTP checks)

### Problem #2 - Useless reporting

Despite the latest dashboard refresh, the reporting is still more or less
useless for operational use. If I'm in crisis mode, the last place I check
for data is Pingdom.

Furthermore, since the reporting is really only useful for measurement of
historical uptime, the percent % availability doens't tell the full story;
going back to the problems in #1, many outages don't stem from a 50x error
code. Instead, intermittent availability or a broken frontend isn't reflected
as the "real" uptime of a service.

In many instances, I can use a service like [GrabPERF](http://www.grabperf.org/scatter.php?test=605&hours=2)
to actually understand when a service interruption began and when it is
fully resolved. The color coding of the data points help quickly understand
periods of slowness, and the raw data linked beneath the graph can help
pinpoint where in the request cycle the slowness is occurring.

![GrabPERF](http://dl.dropbox.com/u/4479354/Screenshots/grabperf3.png)

*Useful scatter plots, with color coding*

*Solution*: Recognize that technical folks have a need for more detail
without the cruft of fancy animated jQuery graphs. They might not look
as sexy on the marketing page, but the ideal uptime service should be
a trusty tool when diagnosing service issues. GrabPERF-level charts
(or a similar representation that cleanly conveys useful timings) in
combination with the better checks listed above could provide just the
right level of information.

### Problem #3 - Integration

The only way to use Pingdom is with [PagerDuty](http://www.pagerduty.com) -
otherwise, you're doing it wrong. PagerDuty is a beautiful extension to the
contacts & alerting components of Pingdom, and it illustrates just how powerful
integrations can be.

Unfortunately, PagerDuty is an anomaly. There's more that Pingdom could be
doing to make the job of developers easier when building tools on top of
the measurement data stored behind their [REST API](http://www.pingdom.com/services/api-documentation-rest/).
The addition of web hooks for post-DOWN and post-UP alerting would help
integrate alerting with status pages and let users implement more advanced
automations on top of service notifications.

Furthermore, the addition of a realtime "firehose" of metric data would
help pipe data into other data stores. Internally, we use Graphite at Disqus
to capture measurements; integrating external measurement data with our
existing metrics could provide key insights and let us build better reporting.

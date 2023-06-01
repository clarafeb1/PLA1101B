# Improving All The Things

Focus on the biggest consumers of run time (= likely causing the highest load) and look for improvement opportunities.  
You can upload examples (searches, Job Inspector screenshots, etc.) here: TODO  
These will be shown on stage, so please don't include any personal info, internal data, etc.

We'll roam the room and work with you.

# Common Improvements

- Datamodel Acceleration
  - Restrict DMs to only relevant indexes
  - Disable unused DMA
  - Find eventtypes with inefficient filters
- Report Acceleration
  - Merge multiple RAs into one RA or one DMA
  - Disable unused RA
- Scheduled Searches
  - Question high frequency execution, especially with large timeranges
  - Merge similar searches into one, possibly using tokens in alerts
  - Configure a default `allow_skew` to flatten scheduler peaks at common minutes, e.g. 0-15-30-45 past the hour
- Dashboards
  - Question high frequency panel refreshs
  - Consider caching for dashboards used by many people at similar times
  - Merge similar searches from multiple panels into one base search with postprocessing (chain searches in Dashboard Studio)
  - Split up enormous dashboards, e.g. a short "monitoring" dashboard that gets used frequently and a long "troubleshooting" dashboard only opened when things go wrong
- General Searches
  - Find inefficient filters, e.g. high scanCount with low eventCount
  - Convert raw event searches to index-based tstats when using only indexed fields
  - Convert raw event searches to DMA-based tstats when a matching accelerated datamodel exists
  - Skip individual processing of the latest few minutes in DMA-based tstats with summariesonly=t if those few minutes are not significant for the result

# Resources

TODO

Clara-Fication: Finding and Improving Expensive Searches
https://conf.splunk.com/watch/conf-online.html?search=PLA1162B

Clara-Fication: More Tstats for Your Buckets
https://conf.splunk.com/watch/conf-online.html?search=tru1133B#/

Splunk: Clara-fication: Job Inspector
https://conf.splunk.com/watch/conf-online.html?search=TRU1143C#/

Fields, Indexed Tokens, and You
https://conf.splunk.com/watch/conf-online.html?search=FN1003#/

Job Inspector Blog Post
https://www.consist.de/de/unternehmen/blog/artikel/Splunk-Suchen-verstehen-mit-dem-Job-Inspector/

Clara-Fication Blog Posts
https://www.splunk.com/en_us/blog/author/cmerriman.html

Extended Search Reporting Dashboard
https://github.com/dpaper-splunk/public/blob/master/dashboards/extended_search_reporting.xml

Expensive Search Dashboard Documentation
https://docs.splunk.com/Documentation/SplunkCloud/latest/Admin/MonitoringSearch#Analyze_expensive_searches

Monitoring Console Documentation
https://docs.splunk.com/Documentation/Splunk/latest/DMC/DMCoverview

tstats Reference Docs
https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/Tstats 

pivot Reference Docs
https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/Pivot 

Additional Background on Map-Reduce
https://en.wikipedia.org/wiki/MapReduce

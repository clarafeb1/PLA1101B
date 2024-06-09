# Estimating Environment Load

## All these are in the Environment Load dashboard, but you can do them manually of course
  
1. Measure Load for SHs and IDXs
```
index=_introspection component=Hostwide host IN (sh-*, idx-*) earliest=-7d
| eval group = if(match(host, "^sh-"), "SH", "IDX") 
| timechart span=1h p90(data.normalized_load_avg_1min) by group
```
The host filter and group eval are written for SplunkCloud hosts, adjust it to fit your environment. On-prem you can view something close to this on the Monitoring Console's Resource Usage pages.  
Run this search to see how busy your indexers and search heads are from a very high-level perspective. This can provide hints if scaling changes need to be made on either tier, and to tell if usage changes had a big impact on the environment load.  
Expectation: Way more load on your indexers than on your search heads.

2. Measure Indexing Load
```
index=_internal host=idx-* source=*metrics.log* group=pipeline earliest=-7d
| timechart per_second(cpu_seconds) [by processor]
```
Run this search to see how many cores of your indexers in total are used for indexing. On-prem you can view something close to this on the Monitoring Console's Indexing Performance: Advanced page.  
Expectation: Not that many.

3. Measure Search Load
```
index=_introspection host=idx-* component=Hostwide earliest=-7d
| eval CPU = 'data.cpu_count'*('data.cpu_system_pct'+'data.cpu_user_pct')/100
| bin _time | stats avg(CPU) as CPU by _time host | timechart sum(CPU)
```
Run this search to see how many cores are used overall on your indexers.  
This technically isn't search load alone, but measuring impact of short-running searches precisely is hard. We'll just deduct the cores from the previous search mentally, as well as some "just running the system" overhead.  
Expectation: Way more cores used for searching than for indexing.


# Attributing Search Load

1. Check where your search load comes from
```
index=_audit host=sh-* action=search info=completed earliest=-7d
| stats sum(total_run_time) as seconds by [host | app | user | provenance]
| sort - seconds
```

Using search run time is a decent estimate for environment load in most situations, and is much easier to calculate.
Most searches put most of their workload on the indexers, therefore cutting down search run time will also cut down indexer load.  
There are exceptions, for example the TrackMe app will exhibit very long search run times but not actually put a lot of load on the indexers.

Play around with the different split-by fields, and drill into the biggest time sinks through filtering.

Key Fields:
- total_run_time in seconds
- host of the Searchhead that launched the search
- app containing the search
- user running the search
- provenance can be “scheduler”, “UI:Search” for ad-hoc, “UI:Dashboard:<name>”, “rest” for API calls, and more (ignore “N/A”)
  
# Improving All The Things

1. Common Improvements

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

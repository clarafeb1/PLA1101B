# Attributing Search Load

Check where your search load comes from
```
index=_audit host=sh-* action=search info=completed
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

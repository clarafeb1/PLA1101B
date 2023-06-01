### Measure Load for SHs and IDXs
This search will display the 90th percentile for the average normalized load on defined hosts Adjust host and group as needed
```
index=_introspection component=Hostwide host IN (<splunk_sh>*, <splunk_idx>*) earliest=-7d
| eval group = if(match(host, "^sh-"), "SH", "IDX") 
| timechart span=1h p90(data.normalized_load_avg_1min) by group
```
### Measure scheduler's execution intervals
This search will display how balanced the load of the scheduled searches are executed
```
index=_audit action=search info=granted provenance=scheduler host=<splunk_sh> earliest=-7d
| timechart span=10s count
| eval second = strftime(_time, "%S"), minute = strftime(_time, "%M")
| class = case(second=0 AND minute%5=0, "5m", second=0, "1m", true(), "")
| count_{class} = count | fields - class count second minute
```
### Measure overlapping timeranges
Are time windows overlapping with schedules?
```| rest splunk_server=<splunk_sh> servicesNS/-/-/saved/searches search="is_scheduled=1" search="disabled=0"
       f=cron_schedule f=dispatch.earliest_time f=dispatch.latest_time
| stats count by cron_schedule dispatch.earliest_time dispatch.latest_time
```
### Find scheduled searches using All Time
```
| rest splunk_server=<splunk_sh> /servicesNS/-/-/saved/searches search="is_scheduled=1" search="disabled=0" f=title f=author f=cron_schedule f=dispatch.earliest_time f=dispatch.latest_time f=eai:acl* f=updated f=qualifiedSearch f=next_scheduled_time f=splunk_server 
| search (dispatch.earliest_time="" OR dispatch.earliest_time="0")
|stats values(author) as author values(cron_schedule) as cron_schedule values(dispatch.earliest_time) as earliest_time values(dispatch.latest_time) as latest_time values(eai:acl.owner) as owner values(updated) as updated values(qualifiedSearch) as qualifiedSearch values(next_scheduled_time) as next_scheduled_time values(splunk_server) as splunk_server by title eai:acl.app
|rename eai:acl.app as app| regex qualifiedSearch="^\s*(search|tstats) " 
| rex field=qualifiedSearch "earliest=(?P<earliestTime>\S+)"|where isnull(earliestTime)
```
### Find scheduled searches without a defined index
```
| rest splunk_server=<splunk_sh> servicesNS/-/-/saved/searches f=disabled f=title f=search f=eai:*
|fields disabled search title splunk_server eai:acl.app
|search disabled=0 
| rex field=search "index((\s?\=\s?\"?)|(\sIN\s))(?<direct_index_searched>[^\s,|']*)"  max_match=0
|where isnull(direct_index_searched)|rex field=search "`(?<macro_searched>[^`]+)`" max_match=0
|rex field=search "eventtype\s?=\s?(?<eventtype_searched>[^\s]+)" max_match=0
|mvexpand macro_searched
|mvexpand eventtype_searched
|join macro_searched splunk_server type=left [| rest splunk_server=<splunk_sh> servicesNS/-/-/admin/macros f=title f=definition
| fields title definition splunk_server
| rex field=definition "index((\s?\=\s?\"?)|(\sIN\s))(?<macro_index_searched>[^\s|\"|']*)" max_match=0 
| rex field=definition "eventtype\s?\=\s?\"?(?<macro_defined_eventtype>[^\s|\"|']+)" max_match=0 
| rex field=definition "`(?<nested_macro>[^`]+)`" max_match=0 
| mvexpand nested_macro
| join nested_macro splunk_server type=left [| rest splunk_server=<splunk_sh> servicesNS/-/-/admin/macros f=title f=definition 
| fields title definition splunk_server 
| rex field=definition "index((\s?\=\s?\"?)|(\sIN\s))(?<nested_macro_index_searched>[^\s|\"|']*)" max_match=0
| rex field=definition "eventtype\s?\=\s?\"(?<nested_macro_defined_eventtype>[^\s]+)" max_match=0 
|fields - definition|rename title as nested_macro]
|eval macro_index_searched=case(isnotnull(macro_index_searched) AND isnotnull(nested_macro_index_searched), mvjoin(macro_index_searched,nested_macro_index_searched), isnotnull(macro_index_searched),macro_index_searched,1=1,nested_macro_index_searched)
|eval macro_defined_eventtype=case(isnotnull(macro_defined_eventtype) AND isnotnull(nested_macro_defined_eventtype),mvjoin(macro_defined_eventtype,nested_macro_defined_eventtype), isnotnull(macro_defined_eventtype),macro_defined_eventtype,1=1,nested_macro_defined_eventtype)
|fields title definition splunk_server macro_defined_eventtype macro_index_searched
|stats values(*) as * by title definition splunk_server
|rename title as macro_searched definition as macro_definition
|join macro_defined_eventtype splunk_server type=left [|rest splunk_server=<splunk_sh> /servicesNS/-/-/saved/eventtypes f=search f=title
|fields - author published id updated|stats values(*) as * by splunk_server title 
|rename title as macro_defined_eventtype splunk_server as splunk_server 
| rex field=search "index((\s?\=\s?\"?)|(\sIN\s))(?<macro_eventtype_index_searched>[^\s|\"|']*)" max_match=0 
|fields - search]]
|join eventtype_searched splunk_server type=left [|rest splunk_server=<splunk_sh> /servicesNS/-/-/saved/eventtypes f=search f=title
|fields - author published id updated
|stats values(*) as * by splunk_server title 
|rename title as eventtype_searched search as eventtype_definition
| rex field=eventtype_definition "index((\s?\=\s?\"?)|(\sIN\s))(?<eventtype_index_searched>[^\s|\"|']*)" max_match=0 
|rex field=eventtype_definition "eventtype\s?\=\s?\"?(?<nested_eventtype>[^\s|\"|']*)" max_match=0
|mvexpand nested_eventtype
|join nested_eventtype splunk_server type=left [|rest splunk_server=<splunk_sh> /servicesNS/-/-/saved/eventtypes f=search f=title
|fields - author published id updated
|stats values(*) as * by splunk_server title 
|rename title as nested_eventtype search as nested_eventtype_definition
| rex field=nested_eventtype_definition "index((\s?\=\s?\"?)|(\sIN\s))(?<nested_eventtype_index_searched>[^\s|\"|']*)" max_match=0 ]
|fields splunk_server eventtype_searched nested_eventtype_index_searched eventtype_index_searched]
|stats values(*) as * by title search|where isnull(eventtype_index_searched)
|where isnull(nested_eventtype_index_searched)|where isnull(macro_index_searched) 
|where isnull(macro_eventtype_index_searched)|fields splunk_server title search
```
### High memory or Long Running searches
```
(index=_audit sourcetype=audittrail (total_run_time=* OR search=*)) OR (index=_introspection data.search_props.sid::* sourcetype=splunk_resource_usage) host=<splunk_sh> earliest=-7d
| eval mem_used=round('data.mem_used',2), search_id=coalesce('data.search_props.sid',search_id)
| stats values(savedsearch_name) as search_name values(data.search_props.provenance) as provenance values(data.search_props.app) as app first(host) as host values(user) as user latest(info) as job_status max(mem_used) as "mem_used_mb" values(search) as raw_search values(total_run_time) as total_run_time values(api_et) as timepicker_start values(api_lt) as timepicker_end values(scan_count) as scan_count by search_id
|convert ctime(timepicker_start) ctime(timepicker_end)|search total_run_time>600 OR mem_used_mb>4000
```
### Search load
Gauge how your search load is distributed over various search types
```
index=_audit host=<splunk_sh> action=search info=completed earliest=-7d
| rex field=provenance "^(?<provenance_group>[^:]+(:[^:]+)?)"
| stats dc(app) as apps dc(user) as users count as searches sum(total_run_time) as seconds by provenance_group
| addinfo | eval concurrency_factor = round(seconds / (info_max_time - info_min_time), 2) | fields - info_* 
| sort - seconds
```
### Multiple SHs accelerating the same DM
```
index=_internal sourcetype=scheduler savedsearch_name="_ACCELERATE_DM_*" earliest=-7d
| stats values(host) as SHs by savedsearch_name
```
### Unused DM
Size & access information for accelerations
```
| rest splunk_server=<splunk_sh> /services/admin/summarization by_tstats=1 
| eval summary.access_time = strftime('summary.access_time', "%F %T")
| table title summary.access_count summary.access_time summary.size
```
### Expensive dashboards
Check resource consumption of dashboards
```
index=_audit host=<splunk_sh> action=search info=completed earliest=-7d
| stats dc(app) as apps dc(user) as users count as searches sum(total_run_time) as seconds by provenance
| addinfo | eval concurrency_factor = round(seconds / (info_max_time - info_min_time), 2) | fields - info_*
| sort - seconds
```
### Dashboard concurrency issues
Find dashboards that are running too many searches
```
index=_internal (sourcetype=splunkd OR sourcetype=scheduler) "The maximum number of concurrent" host=<splunk_sh> earliest=-7d
| rex field=id "(?<ssuser>[^_]+)__" 
| eval user = coalesce(user, username, ssuser), search_id = coalesce(savedsearch_id, id, "null")
| stats count as total_occurrences values(reason) as reason values(search_type) as search_type values(provenance) as provenance by host user search_id 
| where isnotnull(provenance) 
| stats values(search_id) as search_ids sum(total_occurrences) as total_occurrences values(reason) as reason by host user provenance
```
### Dashboards for Post-Processing
Find dashboards that could benefit from post-processing and base searches
```
|rest splunk_server=<splunk_sh> servicesNS/-/-/data/ui/views f=eai:data f=title f=eai:appName
|fields title eai:appName eai:data splunk_server author
|search eai:data="*<search>*"
|xpath outfield=base_id "//search/@id" field=eai:data
|xpath outfield=query "//query" field=eai:data
|rex field=query "\|?(?<generating_spl>[^\|]+)(\||.*)"
|eval total_query=mvcount(generating_spl)
|eval dc_query=mvdedup(generating_spl)
|eval distinct_query=mvcount(dc_query)
|stats values(total_query) as total_query values(distinct_query) as query_count values(base_id) as base_id list(generating_spl) as generating_spl by title eai:appName author splunk_server
|where total_query!=query_count
```

1. Measure Load for SHs and IDXs
```
index=_introspection component=Hostwide host IN (sh-*, idx-*) 
| eval group = if(match(host, "^sh-"), "SH", "IDX") 
| timechart span=1h p90(data.normalized_load_avg_1min) by group
```
The host filter and group eval are written for SplunkCloud hosts, adjust it to fit your environment. On-prem you can view something close to this on the Monitoring Console's Resource Usage pages.  
Run this search to see how busy your indexers and search heads are from a very high-level perspective. This can provide hints if scaling changes need to be made on either tier, and to tell if usage changes had a big impact on the environment load.  
Expectation: Way more load on your indexers than on your search heads.

2. Measure Indexing Load
```
index=_internal host=idx-* source=*metrics.log* group=pipeline
| timechart per_second(cpu_seconds) [by processor]
```
Run this search to see how many cores of your indexers in total are used for indexing. On-prem you can view something close to this on the Monitoring Console's Indexing Performance: Advanced page.  
Expectation: Not that many.

3. Measure Search Load
```
index=_introspection host=idx-* component=Hostwide
| eval CPU = 'data.cpu_count'*('data.cpu_system_pct'+'data.cpu_user_pct')/100
| bin _time | stats avg(CPU) as CPU by _time host | timechart sum(CPU)
```
Run this search to see how many cores are used overall on your indexers.  
This technically isn't search load alone, but measuring impact of short-running searches precisely is hard. We'll just deduct the cores from the previous search mentally, as well as some "just running the system" overhead.  
Expectation: Way more cores used for searching than for indexing.

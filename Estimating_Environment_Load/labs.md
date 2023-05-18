1. Measure Overall Load
```
index=_introspection component=Hostwide host IN (<search heads>,<indexers>) 
| eval group = if(match(host, "^sh-"), "SH", "IDX") 
| timechart span=1h p90(data.normalized_load_avg_1min) by group
```
Run this search to see if the indexers of search heads are taking on too much
The group eval is written for SplunkCloud hosts, it can be updated as needed
Track this often to see if changes need to be made on either tier

2. Measure Indexing Load
```
index=_internal host=idx-* source=*metrics.log* group=pipeline
| timechart per_second(cpu_seconds) [by processor]
```
Run this search to check out indexers more specifically

3. Measuring Search Load
```index=_introspection host=idx-* component=Hostwide
| eval CPU = 'data.cpu_count'*('data.cpu_system_pct'+'data.cpu_user_pct')/100
| bin _time | stats avg(CPU) as CPU by _time host | timechart sum(CPU)
```
Run this search to see the CPU cores over all indexers

4. Check where highest search load comes from
```
index=_audit host=sh-* action=search info=completed
| stats sum(total_run_time) as seconds by [host | app | user | provenance]
| sort - seconds
```

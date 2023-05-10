### Measure Load for SHs and IDXs
This search will display the 90th percentile for the average normalized load on defined hosts 
Adjust host and group as needed
```
index=_introspection component=Hostwide host IN (sh-*, idx-*) 
| eval group = if(match(host, "^sh-"), "SH", "IDX") 
| timechart span=1h p90(data.normalized_load_avg_1min) by group
```

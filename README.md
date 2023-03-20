# ncbiminute2303
Resources for NCBI Minute

# Example queries used in talk

### Count the enoki isolates we just looked at
```sql
SELECT count(*)
FROM `ncbi-pathogen-detect.pdbrowser.isolates`
WHERE isolation_source LIKE '%enoki%'
```

### Get info about a given isolate
```sql
SELECT taxgroup_name, isolate_identifiers, serovar, serotype, erd_group, computed_types
FROM `ncbi-pathogen-detect.pdbrowser.isolates`
WHERE biosample_acc = 'SAMN21357979'
```

### Find the AMRFinderPlus results for an isolate
```sql
SELECT element_symbol, element_name, subclass
FROM `ncbi-pathogen-detect.pdbrowser.microbigge`
WHERE biosample_acc = 'SAMN21357979'
```

### Find the most common SNP clusters with Salmonella Newport isolates
This requires you to count stuff so I'll show you how to use the count() function, GROUP BY, and ORDER BY clauses
```sql
SELECT erd_group, count(*) num_newport_isolates
FROM `ncbi-pathogen-detect.pdbrowser.isolates`
WHERE computed_types.serotype = 'Newport'
    AND taxgroup_name LIKE 'Salmonella%'
GROUP BY erd_group
ORDER BY num_newport_isolates DESC
LIMIT 5
```

### Isolates tested resistant to carbapenems
Here we filter based on one of the complex fields (`AST_phenotypes`)
```sql
SELECT target_acc
FROM `ncbi-pathogen-detect.pdbrowser.isolates` isolates
WHERE
(SELECT COUNT(1)
  FROM UNNEST(isolates.AST_phenotypes)
  WHERE antibiotic LIKE '%penem' AND phenotype = 'resistant'
) >= 1
```

### What are the most common AMR genes in _Salmonella_ Newport isolates
```sql
SELECT mb.element_symbol, mb.subclass, mb.scope, COUNT(DISTINCT(i.target_acc)) num_isolates
FROM `ncbi-pathogen-detect.pdbrowser.isolates` i
    JOIN `ncbi-pathogen-detect.pdbrowser.microbigge` mb
    ON i.target_acc = mb.target_acc
WHERE i.computed_types.serotype = 'Newport'
AND mb.subtype = 'AMR'
GROUP BY mb.element_symbol, mb.subclass, mb.scope
ORDER BY num_isolates DESC
```

### Get contig sequence for an element
### Find the AMRFinderPlus results for an isolate
```sql
SELECT element_symbol, element_name, subclass, contig_acc, isolation_source, contig_url
FROM `ncbi-pathogen-detect.pdbrowser.microbigge`
WHERE biosample_acc = 'SAMN21357979'
ORDER BY contig_acc, start_on_contig
```

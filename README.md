
ncbiminute2303
=================

Resources for NCBI Minute

Example queries used in talk
=============================

### Count the isolates from enoki mushrooms
```sql
-- How many isolates were isolated from "enoki"?
SELECT COUNT(*)
FROM `ncbi-pathogen-detect.pdbrowser.isolates`
WHERE isolation_source LIKE '%enoki%'
```

### Get info about a given isolate
```sql
-- Get info about a given isolate
SELECT taxgroup_name, isolate_identifiers, serovar, serotype, erd_group, computed_types
FROM `ncbi-pathogen-detect.pdbrowser.isolates`
WHERE biosample_acc = 'SAMN21357979'
```

### Find the really bad contigs with the most AMR genes
```sql
-- Find the contigs with the most AMR genes
    SELECT contig_acc, COUNT(*) num_genes, COUNT(COUNT(element_symbol)) num_unique_genes 
    FROM `ncbi-pathogen-detect.pdbrowser.microbigge` 
    WHERE subtype = 'AMR'
    GROUP BY contig_acc
    ORDER BY num_genes DESC
    LIMIT 10
```

### What are the genes on those contigs?
Looks like there were maybe some errors 
```sql
-- What are the genes on the contigs with the most AMR genes
SELECT mb.contig_acc, mb.element_symbol, mb.element_name, mb.subclass, top.num_unique_genes, top.num_genes
FROM `ncbi-pathogen-detect.pdbrowser.microbigge` mb
JOIN (
    SELECT contig_acc, COUNT(*) num_genes, COUNT(DISTINCT(element_symbol)) num_unique_genes 
    FROM `ncbi-pathogen-detect.pdbrowser.microbigge` 
    WHERE subtype != 'AMR'
    GROUP BY contig_acc
    ORDER BY num_genes DESC
    LIMIT 10
  ) top 
ON top.contig_acc = mb.contig_acc
ORDER BY num_genes DESC, start_on_contig
```

### Isolates tested resistant to carbapenems
Here we filter based on one of the complex fields (`AST_phenotypes`)
```sql
--- Find all the isolates tested resistant to carbapenems
SELECT target_acc
FROM `ncbi-pathogen-detect.pdbrowser.isolates` isolates
WHERE
(SELECT COUNT(*)
  FROM UNNEST(isolates.AST_phenotypes)
  WHERE antibiotic LIKE '%penem' AND phenotype = 'resistant'
) >= 1
```

### Bonus, won't explain in detail, find all the isolates tested resistant to carbapenems without a known carbapenem resistance gene or point mutation
```sql
--- find all the isolates tested resistant to carbapenems without a known 
--- carbapenem resistance gene or point mutation
SELECT isolates.target_acc,
  ARRAY(select AS STRUCT antibiotic, phenotype from UNNEST(AST_phenotypes) WHERE  antibiotic LIKE "%penem") AST,
  isolates.refgene_db_version, isolates.taxgroup_name, isolates.scientific_name
FROM `ncbi-pathogen-detect.pdbrowser.isolates` isolates
LEFT JOIN `ncbi-pathogen-detect.pdbrowser.microbigge` microbigge
  ON isolates.target_acc = microbigge.target_acc
  AND microbigge.subclass = 'CARBAPENEM' -- Only carbapenem genes / point mutations
WHERE
  (SELECT count(1) FROM unnest(AST_phenotypes) AS ast
    WHERE antibiotic like "%penem" AND phenotype = 'resistant') >= 1
  AND isolates.amrfinderplus_version IS NOT NULL -- AMRFinderPlus was run on this target
  AND isolates.asm_acc IS NOT NULL -- AMRFinderPlus results should be in MicroBIGG-E because assembly is public
  AND microbigge.subclass IS NULL -- There are no rows in MicroBIGG-E with subclass = CARBAPENEM
ORDER BY isolates.target_acc
```
Get contig sequence for an element
----------------------------------

### Find the AMRFinderPlus results for an isolate
```sql
SELECT element_symbol, element_name, subclass, contig_acc, isolation_source, contig_url
FROM `ncbi-pathogen-detect.pdbrowser.microbigge`
WHERE biosample_acc = 'SAMN21357979'
ORDER BY contig_acc, start_on_contig
```

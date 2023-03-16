# ncbiminute2303
Resources for NCBI Minute

# Example queries used in talk

### Count the enoki isolates we just looked at
```sql
select count(*)
from `ncbi-pathogen-detect.pdbrowser.isolates`
where isolation_source like '%enoki%'
```

### Get info about a given isolate
```sql
SELECT taxgroup_name, isolate_identifiers, serovar, serotype, erd_group, computed_types
FROM `ncbi-pathogen-detect.pdbrowser.isolates`
WHERE biosample_acc = 'SAMN11101132'
```

### Find the AMRFinderPlus results for an isolate
```sql
SELECT element_symbol, element_name, subclass
FROM `ncbi-pathogen-detect.pdbrowser.microbigge`
WHERE biosample_acc = 'SAMN11101132'
```

### Look at some of the complex fields
```sql
SELECT biosample_acc, asm_acc, target_acc, mindiff, AMR_genotypes,
    stress_genotypes, virulence_genotypes, computed_types
FROM `ncbi-pathogen-detect.pdbrowser.isolates`
WHERE biosample_acc = 'SAMN08848639'
```
Notice we found some fields that have multiple values. We'll get to how to query those fields a little later, that's a complex part of BigQuery sql that you may not have encountered before even if you're familiar with SQL

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

### Find the five most common AMR genes associated with quinolone resistance
So far this has all been from the Isolates Browser, lets see some of the things you can do with the MicroBIGG-E data
```sql
SELECT element_symbol, subclass, count(*) num_found
FROM `ncbi-pathogen-detect.pdbrowser.microbigge`
WHERE subclass like '%QUINOLONE%'
AND   subtype = 'AMR'
GROUP BY element_symbol, subclass
ORDER BY num_found DESC
LIMIT 5
```

### Genes are on contigs that sometimes contain other genes. Lets say you want to find out if two genes are likely linked
```sql
SELECT
    mb.contig_acc,
    mb.element_symbol,
    mb.scientific_name
FROM
    `ncbi-pathogen-detect.pdbrowser.microbigge` mb
    JOIN (
        SELECT DISTINCT
            mb1.contig_acc
        FROM
            `ncbi-pathogen-detect.pdbrowser.microbigge` mb1
            JOIN `ncbi-pathogen-detect.pdbrowser.microbigge` mb2
                ON mb1.element_symbol = 'blaTEM-1'
                    AND mb1.contig_acc = mb2.contig_acc
                    AND mb2.element_symbol = 'blaKPC-2') contigs
    ON contigs.contig_acc = mb.contig_acc
ORDER BY
    mb.contig_acc,
    mb.start_on_contig
```


### Now lets see how many are carbapenem resistant, but don't have either of the "common" carbapenemase genes blaKPC or blaNDM
```sql
SELECT target_acc
FROM `ncbi-pathogen-detect.pdbrowser.isolates` isolates
WHERE
(SELECT COUNT(1)
  FROM UNNEST(isolates.AST_phenotypes)
  WHERE antibiotic LIKE '%penem' AND phenotype = 'resistant'
) >= 1
AND
(SELECT COUNT(1)
  FROM UNNEST(isolates.AMR_genotypes)
  WHERE element LIKE 'blaKPC%' OR element LIKE 'blaNDM%'
) = 0
```


That's only two carbapenemases, what if we just look for isolates without any known carbapenmase genes or carbapenem resistance point mutations
### Find isolates that are carbapenem resistant, but don't have a known carbapenem resistance gene or allele
```sql
SELECT isolates.target_acc,
  ARRAY(select AS STRUCT antibiotic, phenotype from UNNEST(AST_phenotypes) WHERE  antibiotic LIKE "%penem") AST
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


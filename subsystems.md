# How BV-BRC genomes, features and subsystems connect

This page explains how the IDs used by the subsystem lookup in the [README](README.md#look-up-subsystems-for-the-hits) relate to each other, and how to tell why a lookup returns nothing. All commands need `module load bvbrc/1.048`.

## The chain

```
genome (genome_id)  ->  features (patric_id)  ->  subsystem rows
```

- A **genome** has a `genome_id`, for example `511145.12`.
- Each **feature** (gene) in that genome has a `patric_id` of the form `fig|<genome_id>.<type>.<number>`, for example `fig|511145.12.peg.1`. These are the IDs that BLAST reports as `sseqid`.
- Each **subsystem row** names one feature, a subsystem and the feature's role in it.

The commands follow the same links:

| Command | Input | Looks up |
|---|---|---|
| `p3-get-genome-features` | `genome_id` | features in those genomes |
| `p3-get-feature-subsystems` | `patric_id` | subsystem rows for those features |

Sources: BV-BRC CLI source, [`p3-get-genome-features.pl`](https://github.com/BV-BRC/p3_cli/blob/a152279/scripts/p3-get-genome-features.pl#L80) (line 80) and [`p3-get-feature-subsystems.pl`](https://github.com/BV-BRC/p3_cli/blob/a152279/scripts/p3-get-feature-subsystems.pl#L58) (line 58); BV-BRC API data types [`genome_feature`](https://www.bv-brc.org/api/doc/genome_feature) and [`subsystem`](https://www.bv-brc.org/api/doc/subsystem).

## A genome ID is not a taxonomy ID

In `511145.12`, the number before the dot (511145) is the genome's taxonomy ID. A taxonomy ID alone is not a genome. For example, 83333 is the taxonomy ID for *E. coli* K-12:

```bash
$ echo -e "taxon_id\n83333" | p3-get-taxonomy-data -a taxon_name -a taxon_rank
taxon_id	taxonomy.taxon_name	taxonomy.taxon_rank
83333	Escherichia coli K-12	strain
```

There is no genome `83333.1`:

```bash
$ p3-all-genomes --eq genome_id,83333.1 --attr genome_name
genome.genome_id	genome.genome_name
```

So an ID such as `fig|83333.1.peg.4` does not exist.

## Positive control: thrA and dnaK in E. coli K-12 MG1655

1. Find the genome ID of the reference MG1655 genome:

   ```bash
   $ p3-all-genomes --eq "genome_name,Escherichia coli str. K-12 substr. MG1655" --eq reference_genome,Reference --attr genome_name
   genome.genome_id	genome.genome_name
   511145.12	Escherichia coli str. K-12 substr. MG1655
   ```

   Without `--eq reference_genome,Reference`, this name matches dozens of genomes.

2. Get the feature IDs for two genes:

   ```bash
   $ echo -e "genome_id\n511145.12" | \
       p3-get-genome-features --in gene,thrA,dnaK -a patric_id -a gene -a product > features.tsv
   $ cat features.tsv
   genome_id	feature.patric_id	feature.gene	feature.product
   511145.12	fig|511145.12.peg.1	thrA	Aspartokinase (EC 2.7.2.4) / Homoserine dehydrogenase (EC 1.1.1.3)
   511145.12	fig|511145.12.peg.12	dnaK	Chaperone protein DnaK
   ```

3. Look up their subsystems. `-c feature.patric_id` names the feature-ID column:

   ```bash
   $ p3-get-feature-subsystems -c feature.patric_id -a class -a subsystem_name < features.tsv
   genome_id	feature.patric_id	feature.gene	feature.product	subsystemItem.class	subsystemItem.subsystem_name
   511145.12	fig|511145.12.peg.1	thrA	Aspartokinase (EC 2.7.2.4) / Homoserine dehydrogenase (EC 1.1.1.3)	Amino Acids and Derivatives	Diaminopimelate Synthesis
   511145.12	fig|511145.12.peg.1	thrA	Aspartokinase (EC 2.7.2.4) / Homoserine dehydrogenase (EC 1.1.1.3)	Amino Acids and Derivatives	Aspartate to Homoserine module
   511145.12	fig|511145.12.peg.1	thrA	Aspartokinase (EC 2.7.2.4) / Homoserine dehydrogenase (EC 1.1.1.3)	Amino Acids and Derivatives	Aspartate to Homoserine module
   511145.12	fig|511145.12.peg.1	thrA	Aspartokinase (EC 2.7.2.4) / Homoserine dehydrogenase (EC 1.1.1.3)	Amino Acids and Derivatives	Aspartate to Threonine Module
   511145.12	fig|511145.12.peg.1	thrA	Aspartokinase (EC 2.7.2.4) / Homoserine dehydrogenase (EC 1.1.1.3)	Clustering-based subsystems	Peptidase clustering with DAP
   511145.12	fig|511145.12.peg.1	thrA	Aspartokinase (EC 2.7.2.4) / Homoserine dehydrogenase (EC 1.1.1.3)	Amino Acids and Derivatives	Aspartate to Threonine Module
   511145.12	fig|511145.12.peg.1	thrA	Aspartokinase (EC 2.7.2.4) / Homoserine dehydrogenase (EC 1.1.1.3)	Amino Acids and Derivatives	Lysine DAP biosynthetic pathway
   511145.12	fig|511145.12.peg.12	dnaK	Chaperone protein DnaK	Stress Response, Defense and Virulence	Heat shock dnaK gene cluster extended
   511145.12	fig|511145.12.peg.12	dnaK	Chaperone protein DnaK	Protein Fate (folding, modification, targeting, degradation)	Protein chaperones
   511145.12	fig|511145.12.peg.12	dnaK	Chaperone protein DnaK	Protein Fate (folding, modification, targeting, degradation)	Chaperones GroEL GroES and Thermosome
   ```

   One feature can belong to several subsystems, one per row. Some rows appear twice; the duplicates can be ignored.

## Why a lookup returns only the header

`p3-get-feature-subsystems` prints only the header line, with no error message, for an ID that does not exist **and** for one that exists but has no subsystem rows. To tell these apart, check the IDs with `p3-get-feature-data`:

```bash
$ printf 'id\nfig|511145.12.peg.4\nfig|83333.1.peg.4\n' > check.tsv
$ p3-get-feature-data -c id -a product < check.tsv
id	feature.product
fig|511145.12.peg.4	Uncharacterized protein YaaX
$ p3-get-feature-subsystems -c id -a subsystem_name < check.tsv
id	subsystemItem.subsystem_name
```

- `fig|511145.12.peg.4` exists (an uncharacterized protein) but has no subsystem rows.
- `fig|83333.1.peg.4` is missing from the `p3-get-feature-data` output, so it does not exist.

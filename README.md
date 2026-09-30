# BV-BRC CLI on the BU Shared Computing Cluster (SCC)

Submitting [BV-BRC](https://www.bv-brc.org/) BLAST jobs from the SCC with the `bvbrc/1.048` module. More steps will be added as they are tested.

## Log in

Requires a free BV-BRC account (https://www.bv-brc.org/register).

```bash
$ module load bvbrc/1.048
$ p3-login <username>
$ p3-login --status
```

## Create a workspace folder

The output folder must already exist in your BV-BRC workspace:

```bash
$ p3-mkdir /<username>@bvbrc/home/SCC_CLI
$ p3-ls /<username>@bvbrc/home
```

## Take a subset of a large input file

Assume your protein FASTA file is named `proteins.faa`.

Count the proteins (FASTA headers, not lines), then write the first 500 to a new file:

```bash
$ grep -c '^>' proteins.faa
$ awk '/^>/{n++} n>500{exit} {print}' proteins.faa > sample500.faa
$ grep -c '^>' sample500.faa
```

## Submit a BLAST job against the website's database

On the website, the BLAST database "Reference and representative genomes (bacteria, archaea)" is named `bacteria-archaea`. `p3-submit-BLAST --db-database` only accepts `BV-BRC`, `REFSEQ`, `Plasmids` or `Phages`, and it rejects `bacteria-archaea`. Using `BV-BRC` with `--db-type faa` fails on the server with `Unable to find database`.

To work around this, submit the job parameters directly with `appserv-start-app`:

1. Upload the protein FASTA to the workspace:

   ```bash
   $ p3-cp -m faa=feature_protein_fasta sample500.faa ws:/<username>@bvbrc/home/SCC_CLI/
   $ p3-ls -l /<username>@bvbrc/home/SCC_CLI
   ```

2. Download [`examples/params.json`](examples/params.json) and edit these fields (example username `jdoe`):

   ```bash
   $ wget https://raw.githubusercontent.com/aramp10/bvbrc-cli-scc/main/examples/params.json
   ```

   | Field | Original | Change to |
   |---|---|---|
   | `input_fasta_file` | `/<username>@bvbrc/home/SCC_CLI/sample500.faa` | `/jdoe@bvbrc/home/SCC_CLI/sample500.faa` |
   | `output_path` | `/<username>@bvbrc/home/SCC_CLI` | `/jdoe@bvbrc/home/SCC_CLI` |
   | `output_file` | `sample500` | `pilot500` (any job name) |

   For a different input file, change the file name at the end of `input_fasta_file` to match the uploaded file. Leave the other fields as they are. The JSON came from `p3-submit-BLAST --dry-run`, with `db_precomputed_database` changed to `bacteria-archaea`.

3. Submit, then check status:

   ```bash
   $ appserv-start-app --id-file job.taskid Homology params.json
   $ p3-job-status $(cat job.taskid)
   ```

   The job also appears on the website's Jobs page.

## Download the results

Outputs go to a hidden folder named after `output_file` (here `.pilot500`):

```bash
$ p3-ls -l /<username>@bvbrc/home/SCC_CLI/.pilot500
$ p3-cp ws:/<username>@bvbrc/home/SCC_CLI/.pilot500/blast_out.txt .
$ p3-cp ws:/<username>@bvbrc/home/SCC_CLI/.pilot500/blast_headers.txt .
```

`blast_out.txt` is tab-separated, up to 10 hits per query, with columns:

```
qseqid  sseqid  pident  length  mismatch  gapopen  qstart  qend  sstart  send  evalue  bitscore  qlen  slen
```

`qseqid` is the ID from your FASTA header. `sseqid` is the BV-BRC feature ID (`fig|...`).

A 500-protein test job took about 1 h 15 min from submission to completion.

## Look up subsystems for the hits

Subsystems are not in the BLAST output files. Look them up from the `sseqid` feature IDs.

1. Keep the top hit per query:

   ```bash
   $ awk -F'\t' '!seen[$1]++ {print $1"\t"$2}' blast_out.txt > best_hits.tsv
   ```

2. Look up subsystems. `-c id` names the feature-ID column; the `query_id` column is kept in the output:

   ```bash
   $ (echo -e "query_id\tid"; cat best_hits.tsv) | \
       p3-get-feature-subsystems -c id -a class -a subclass -a subsystem_name -a role_name \
       > subsystems_best_hits.tsv
   ```

   `subsystems_best_hits.tsv` has every class and subclass for each top hit.

3. Optional: filter by class, for example:

   ```bash
   $ awk -F'\t' 'NR==1 || $3=="Stress Response, Defense and Virulence"' \
       subsystems_best_hits.tsv > stress_best_hits.tsv
   ```

A query can appear on several rows, because a feature can belong to more than one subsystem. Subclass names within a class vary (for example `Stress Response: Heat/cold shock`, or blank), so check column 4 before filtering on subclass.

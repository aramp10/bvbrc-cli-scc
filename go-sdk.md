# BV-BRC Go-SDK on the SCC

The [BV-BRC Go-SDK](https://github.com/BV-BRC/BV-BRC-Go-SDK) is a newer, separate release of the BV-BRC command-line tools, with the same `p3-*` command names. It is an alternative to the `bvbrc/1.048` module used in the [README](README.md). You install it in your own conda environment.

## On BU SCC

First time using conda on SCC? Set up your `.condarc` first, following [Miniconda Modules](https://www.bu.edu/tech/support/research/software-and-programming/common-languages/python/python-software/miniconda-modules/) (`setup_scc_condarc.sh`).

**Warning:** without a properly set up `.condarc`, conda stores environments in your SCC home directory (10 GB quota), which a few environments can easily fill.

## Install

`bv-brc` is BV-BRC's conda channel. This installs the latest release:

```bash
$ module load miniconda/25.3.1
$ mamba create -y -n bvbrc_go_env
$ mamba activate bvbrc_go_env
$ conda install -y -c bv-brc bvbrc-cli
$ p3-ls --version
```

Tested 2026-10-22; this installed version 2.0.17.

In later sessions, only activate the environment:

```bash
$ module load miniconda/25.3.1
$ mamba activate bvbrc_go_env
```

For the list of commands, see the [Go-SDK Command Reference](https://github.com/BV-BRC/BV-BRC-Go-SDK#command-reference).

## Known issue: subsystem lookup fails

In versions 2.0.16 and 2.0.17, `p3-get-feature-subsystems` returns only a header line and this error:

```
Error: querying subsystems for feature fig|511145.12.peg.1: API error: 400 Bad Request - A Database Error Occured:
...
    "msg": "undefined field patric_id",
```

The same lookup works with the `bvbrc/1.048` module. As a workaround, leave the conda environment and load the module:

```bash
$ conda deactivate
$ module load bvbrc/1.048
```

Then follow [Look up subsystems for the hits](README.md#look-up-subsystems-for-the-hits) in the README.

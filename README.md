# Vagrant asmap

Generates an ip_asn.map file for bitcoin-core,
to support [early testing](https://blog.bitmex.com/call-to-action-testing-and-improving-asmap/)
of the [bitcoin-asmap PR](https://github.com/bitcoin/bitcoin/pull/18573)

## System Requirements

You will need:
- a fast internet connection
- 7GB of disk space for the dumps
- 12GB of ram for the analysis
- patience

## Usage

- First time run:

```
vagrant up
```

- Run again the VM is already up:

```
vagrant provision
```

- To force the asmap-rs dump re-download:

```
FORCE_DOWNLOAD=true vagrant provision
```

- To force the asmap-rs bottleneck re-analysis:

```
FORCE_ANALYSIS=true vagrant provision
```

## Output

All files are output in a `asmap` subdirectory of the directory where `vagrant ...` commands are run.

- `asmap/`
  - `dumps/*`: gzipped asn dumps
  - `bottlenecks/*`: bottleneck analysis result files
  - `ip_asn.map`: file to be passed to bitcoind's `-asnmap` command-line argument

## Links

- <https://jonatack.github.io/articles/how-to-compile-bitcoin-core-and-run-the-tests>
- <https://jonatack.github.io/articles/how-to-review-pull-requests-in-bitcoin-core>
- <https://github.com/bitcoin/bitcoin/blob/master/doc/build-unix.md>
- <https://github.com/bitcoin/bitcoin/blob/master/doc/productivity.md>


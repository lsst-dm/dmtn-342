[![Website](https://img.shields.io/badge/dmtn--342-lsst.io-brightgreen.svg)](https://dmtn-342.lsst.io)
[![CI](https://github.com/lsst-dm/dmtn-342/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-dm/dmtn-342/actions/workflows/ci.yaml)

# APDB Backup Strategy and Implementation Options

## DMTN-342

The data in Alert Production Database (APDB) is critical for Alert Production pipeline and it needs to be backed up. This note explains motivation behind APDB backup strategy, its existing implementation and options for its future extension.

**Links:**

- Publication URL: https://dmtn-342.lsst.io
- Alternative editions: https://dmtn-342.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-342
- Build system: https://github.com/lsst-dm/dmtn-342/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

```sh
git clone https://github.com/lsst-dm/dmtn-342
cd dmtn-342
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://dmtn-342.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-342.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.

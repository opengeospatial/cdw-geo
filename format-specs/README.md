# GeoParquet Standard

The GeoParquet standard is specified in this directory. For the clearest overview of the requirements see [`geoparquet.md`](geoparquet.md). It is the 'latest' version of the specification, and you can see its version in the [Version and Schema](geoparquet.md#version-and-schema) section of the document. If it has `-dev` in the suffix then it is an unreleased version of the standard. For the stable versions view the `geoparquet.md` file in the git tree tagged in [the releases](https://github.com/opengeospatial/geoparquet/releases), for example [v1.0.0/format-specs/geoparquet.md](https://github.com/opengeospatial/geoparquet/blob/v1.0.0/format-specs/geoparquet.md) for version 1.0.0.

The official OGC GeoParquet standard is also contained in this directory, and it will often lag behind the markdown document. The target version of the GeoParquet standard can be found in the Preface of the [front material document](sections/clause_0_front_material.adoc). The OGC standard is built from the various other documents in this directory. They are `.adoc` files, in the [asciidoc](https://asciidoc.org/) format. They all automatically get built into a single pdf and published at [docs.ogc.org/DRAFTS/24-013.html](https://docs.ogc.org/DRAFTS/24-013.html) by a cron job running on OGC's infrastructure. The 'official' OGC version will be proposed from that draft, and when accepted by the OGC Technical Committee (TC) will become the official 1.0.0 version of the specification.

[Released versions](https://github.com/opengeospatial/geoparquet/releases) of GeoParquet (from the markdown file in this repository) will not be changed when OGC officially releases GeoParquet 1.0.0, so if changes are needed for OGC approval, then the  will be released with a new version number. There will continue to be releases from this repository, which will technically remain 'draft' standards until the OGC TC has officially accepted the next version.

## In this directory

The key files and folders in this directory are as follows:

* [`geoparquet.md`](geoparquet.md) - The latest specification overview, which may run ahead of the standard. It consists of narrative explanations and clear tables for people to get a clear idea of all that needs to be done to implement GeoParquet.
* [`schema.json`](schema.json) - The definitive schema that validates GeoParquet metadata to ensure complaince with the standard.
* [`compatible-parquet.md`](compatible-parquet.md) - A set of guidelines for those would like to produce geospatial Parquet data but are using tools that are not yet fully implementing GeoParquet metadata. Not an official part of the standard.
* [`document.adoc`](document.adoc) - The main standard document which sets the order of the other sections. This is less 'human-readable', as it is designed to be an official 'standard', with specific language to detail testable requirements.
* [`sections/`](sections/) - Each section of the standard document is a separate document in this folder. The order in the official standard is determined by the `document.adoc`. Most of these documents are boilerplate.
* [`sections/clause_6_normative_text.adoc`](sections/clause_6_normative_text.adoc) - The main text of the standard. Similar to the
`geoparquet.md`, but links to the definitive `requirements`.
* [`requirements/`](requirements/) - directory for requirements and requirement classes to be referenced in the normative text.
* [`abstract_tests/`](abstract_tests/) - the Abstract Test Suite comprising one test for every requirement.

There are a number of other folders, that are currently all empty, but are potentially used for the standard. These are retained for potential future use, but all are currently empty (except for template readmes)

* [`figures`](figures/) - Any figures needed for the standard go in this folder.
* [`images`](images/) - Image files for graphics in the standard go in this folder. Image files for figures go in the `figures` directory. Only place in here images not used in figures (e.g., as parts of tables, as logos, etc.)
* [`code`](code/) - Sample code to accompany the standard, if desired

More information about the document template is [here](https://github.com/opengeospatial/templates/tree/master/standard#readme).

## Authoring the Specification

The GeoParquet markdown file will naturally be a bit 'ahead' of the OGC standard defined in asciidocs. For now the way to author the spec is to just focus on pull requests to the markdown file. The 'community' release will be cut from the markdown file, and then the 'official' OGC release will follow. A volunteer will update all the asciidoc text and requirements to reflect the release, and submit to OGC for official voting.

This may shift in the future, requiring PR's to the markdown to also update the asciidocs, but for now there will just be 'batch' processing of the changes.

An authoring guide for the metanorma / asciidoc editing of the standard is available at [metanorma.org](https://www.metanorma.org/author/ogc/authoring-guide/).

## Building the OGC standard

A local version of the OGC standard can be created by running `docker run -v "$(pwd)":/metanorma -v ${HOME}/.fontist/fonts/:/config/fonts  metanorma/metanorma  metanorma compile --agree-to-terms -t ogc -x html document.adoc`.

## Auto built document

A daily built document is available at [OGC Document DRAFTS](https://docs.ogc.org/DRAFTS/).

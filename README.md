# Mozilla Pipeline Schemas

This repository contains schemas for Mozilla's data ingestion pipeline and data
lake outputs.

The JSON schemas are used to validate incoming submissions at ingestion time.
The [RapidJSON](http://rapidjson.org/md_doc_schema.html) library is used for
JSON Schema Validation. This has implications for what kinds of string patterns
are supported, see the `Conformance` section in the linked document for further
details.

To learn more about writing JSON Schemas,
[Understanding JSON Schema](https://spacetelescope.github.io/understanding-json-schema/index.html)
is a great resource.

The [Parquet-MR](https://github.com/apache/parquet-format/blob/master/LogicalTypes.md)
schemas are used for direct to parquet output; some examples of Parquet-MR
schemas can be found here:
[Parquet Schema Examples](https://mozilla-services.github.io/lua_sandbox_extensions/parquet/io_modules/lpeg/parquet.html)

## Adding a new schema

- Create the JSON Schema in the `templates` directory first. Make use of common schema components from the `templates/include` directory where possible, including things like the telemetry `environment`, `clientId`, `application` block, or UUID patterns. The filename should be `templates/<namespace>/<doctype>/<doctype>.<version>.schema.json`.
- If the data will be saved in parquet form, also add a Parquet-MR schema at `templates/<namespace>/<doctype>/<doctype>.<version>.parquetmr.txt`.
- Build the rendered schemas using the instructions below, and check those artifacts (in the `schemas` directory) in to the git repo as well. See the rationale for this in the "Notes" section below.
- Add one or more example JSON documents to the `validation` directory.
- Run the tests (either via Docker or directly) using the instructions below.
- Once all tests pass, submit a PR to the github repository against the `dev` branch.

## Build

### Prerequisites

* CMake (3.0+) - http://cmake.org/cmake/resources/software.html
* jq (1.5+) - https://github.com/stedolan/jq

### CMake Build Instructions

    git clone https://github.com/mozilla-services/mozilla-pipeline-schemas.git
    cd mozilla-pipeline-schemas
    mkdir release
    cd release

    cmake ..  # this is the build process (the schemas are built with cmake templates)

### Running Tests via Docker

The tests expect example pings to be in the `validation/<namespace>/` subdirectory, with files named
in the form `<ping type>.<version>.<test name>.pass.json` for documents expected to be valid, or
`<ping type>.<version>.<test name>.fail.json` for documents expected to fail validation.
The `test name` should match the pattern `[0-9a-zA-Z_]+`

To run the tests:

    # build the container with the pipeline schemas
    docker build -t mps .

    # run the tests
    docker run --rm mps

### Packaging and integration tests (optional)

Follow the CMake Build Instructions above, then:

    cpack -G TGZ # (DEB|RPM|ZIP)

    # Integration Tests (run on schema-test EC2 instance)
      # If running locally
        # The following RPM's must be installed:
          # luasandbox, hindsight, luasandbox-lfs, luasandbox-lpeg, luasandbox-rjson, luasandbox-cjson, luasandbox-parquet
        # The following external libraries must be installed
          # parquet-cpp
    make # this sets up the tests in the release directory
    ctest -V -C hindsight # loads all the schemas and tests the inputs in the validation directory against them

## Releases

* The master branch is the current release and is considered stable at all
  times.
* New versions can be released as frequently as every two weeks (our sprint
  cycle). The only exception would be for a high priority patch.
* New releases occur the day after the sprint finishes.
  * The version in the dev branch is updated
  * The changes are merged into master
  * A new tag is created

## Contributions

* All pull requests must be made against the `dev` branch, direct commits to
  `master` are not permitted.
* All non trivial contributions should start with an issue being filed (if it is
  a new feature please propose your design/approach before doing any work as not
  all feature requests are accepted).

### Notes

All schemas are generated from the 'templates' directory and written into the
'schemas' directory (i.e., the artifacts are generated/saved back into the
repository) and validated against the [draft 4 schema](http://json-schema.org/draft-04/schema)
a [copy](https://github.com/mozilla-services/mozilla-pipeline-schemas/blob/master/tests/hindsight/jsonschema.4.json)
of which resides in the 'tests' directory. The reason for this is twofold:

1. It lets us easily see and refer to complete schemas as they are actually used.
This means that the schemas can be referenced directly in bugs and such,
as well as being fetched directly from the repo for testing other schema
consumers (test being important here, as any production use should be using the
installable packages).
1. It gives us a changelog for each schema, rather than having to reason about
changes to templated external pieces and when/how that impacted a given
doctype's schema over time. This means that it should be easy to look back in
time for the provenance of different parts of the schema for each doctype.

# Tutorial: Getting Started with Connector Architecture

## Introduction

[connector architecture](https://github.com/TREEcg/connector-architecture/wiki) is a lightweight data streaming tool designed to
facilitate the real-time processing of data as it is generated or received. 

In this tutorial, we will explore how to bootstrap a pipeline with Connector Architecture to collect data from a SPARQL endpoint and transform it into Linked Data Fragments for substring search[1].

## Prerequisites

```shell
# 1. create your own processor GitHub repository and clone it to local
git clone https://github.com/YOUR-USERNAME/YOUR-REPOSITORY

# 2. set Connector-Architecture up in your local environment
git clone https://github.com/TREEcg/connector-architecture.git

# 3. create a new branch of Connector Architecture repo for PR and add a new processor
cd connector-architecture
git branch <new-branch>
cd processor
git submodule add https://github.com/YOUR-USERNAME/YOUR-REPOSITORY
```

## TL;DR
A Connector Architecture processor refers to a module function that emits an event, which may include a series of actions, performs to achieve dynamic data being generated on a continual basis.
Typically, a processor, except for the one used to acquire data from an endpoint or ingest data into a database, expects Readable stream(s) as input and outputs Writeable stream(s).

Connector Architecture implements its own Stream types. Keep in mind that they are not derived from NodeJS stream! More details can be found [connector-types](https://github.com/TREEcg/connectors/tree/main/packages/types).

In practice, data chunks of a Connector Architecture stream in a processor are better to be of type *string* in oder to make the processor reusable beyond JavaScript.
- Connector Architecture Readable stream - `Stream`
    - listen to the `Stream.on('data')` event to process each data chunk
    - use `Stream.on('end')` event to
        - do data post-processing e.g. making numbers count
        - close the `Writer` writeable stream, to which processed data have been pushed, if desired.

- Connector Architecture Write stream - `Writer`
    - use `Writer.push()` to push a chunk of data

The synergy between readable and writeable streams is a fundamental concept in data streaming. In Connector Architecture, stream piping are achieved through [pipeline](https://github.com/TREEcg/connector-architecture/wiki/Pipeline) configuration. The output Writeable stream of a process will become the input Readable stream of another, when the two processors are bundled in a pipeline.

### Processors

The pipeline we are about to implement consists of 5 JS/TS processors. Long story short, you do not really need to do anything except for simply compiling them into your pipeline. 


- [SPARQL query processor](-bucketizer-index-proc/blob/main/steps/querySparql.ttl)
    - fetch SPARQL query result using Comunica queryEngine
- [Sdsity processor](https://github.com/ajuvercr/sds-processors/blob/master/sdsify.ttl)
    - serialize quads with SDS vocabularies
- [Bucketization processor](https://github.com/ajuvercr/sds-processors/blob/master/2_bucketstep.ttl)
    - bucketize quads based on the value of propertyPath defined in `config.json` for substring search
- [SDS to TREE processor](https://github.com/jiaoxlong/substring-bucketizer-index-proc/blob/main/steps/sds_to_tree.ttl)
    - remodel the quads according to the TREE specification
- [Ingestion processor](https://github.com/jiaoxlong/substring-bucketizer-index-proc/blob/main/steps/ingest.ttl)
    - per chunk of quads from stream
        - asynchronously allocate to its corresponding bucket (fragment)
        - add quads of tree:remainingItems count
        - write/append to file, defaults to turtle format.
        
### Runner and Chanel

In this tutorial, we will only use JS runner to execute the processors. Naturally, Connector Architecture does not limit to this. The main focus of Connector Architecture is on enabling different data processors to work together effectively, 
regardless of the specific technologies or frameworks they are built upon (e.g. a JS stream instance that feeds into a Websoket instance).

### Pipeline

```shell
cp sparql-sdsify-bucketizer-tree-file-pipeline.ttl <YOUR-REPOSITORY>
cd connector-architecture/runner/js-runner; node ./bin/js-runner.js ../../<YOUR-REPOSITORY>/sparql-sdsify-bucketizer-tree-file-pipeline.ttl 

```


[1]: Dedecker, R., Delva, H., Colpaert, P., Verborgh, R. (2021). A File-Based Linked Data Fragments Approach to Prefix Search. In: Brambilla, M., Chbeir, R., Frasincar, F., Manolescu, I. (eds) Web Engineering. ICWE 2021. Lecture Notes in Computer Science(), vol 12706. Springer, Cham. https://doi.org/10.1007/978-3-030-74296-6_5

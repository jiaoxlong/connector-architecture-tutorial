# Tutorial: Getting Started with Connector Architecture

## Introduction

[connector architecture](https://github.com/TREEcg/connector-architecture/wiki) is a lightweight data streaming tool designed to
facilitate the real-time processing of streaming data as it is generated or received. 

In this tutorial, we will explore how to bootstrap a pipeline with Connector Architecture to collect data from a SPARQL endpoint and transform it into Linked Data Fragments for substring search[1]. A live demo, which consumes the end result, is available at [Autocomplete over TREE structured fragmentations](https://tree.linkeddatafragments.org/demo/autocompletion/?datasets%5B%5D=https%253A%252F%252Ftreecg.github.io%252Fdemo_data%252Fera.ttl)

## Preliminaries

A Connector Architecture processor refers to a module function that emits an event, which may include a series of actions, performs to achieve dynamic data being generated on a continual basis.
Typically, a processor, except for the one used to acquire data from an endpoint or ingest data into a database, expects Readable stream(s) and Writeable stream(s) as input and output, respectively.

---
**How to create a Connector Architecture processor?**

Assume an example processor function *func()*

```typescript
/**
 * func() encodes URI strings received from a readable Stream 
 * and generate a writeable Stream with the encoded URL strings
 * @param readableStream
 * @param writeableStream
 */
async function func(readableStream:Stream<string>, writeableStream:Writer<string>){
    readableStream.on('data', async str=>{
        writeableStream.push(encodeURIComponent(str))
    })
    readableStream.on('end',()=> {
        console.log("writeable stream is closed");
        writeableStream.end()
    })
}

```
Within Connector Architecture, every processor needs to be configured, by means of introducing a configuration file coded in [Turtle](https://www.w3.org/TR/turtle/), 
as demonstrated below,  in order to be plugged into a pipeline. 

*Processor configuration*

1. [Predefined Namespace Prefixes](https://www.w3.org/TR/turtle/#prefixed-name)
> ```
> @prefix js: <https://w3id.org/conn/js#> .
> @prefix fno: <https://w3id.org/function/ontology#> .
> @prefix fnom: <https://w3id.org/function/vocabulary/mapping#> .
> @prefix xsd: <http://www.w3.org/2001/XMLSchema#> .
> @prefix : <https://w3id.org/conn#> .
> @prefix sh: <http://www.w3.org/ns/shacl#> .
> @prefix owl: <http://www.w3.org/2002/07/owl#> .
> ...

2. Import additional ontology 

Note that you may want to open the link to browse the set of concepts and categories, introduced in 
the Connector Architecture, shows their properties and the relations between them.
> ```
> <> owl:imports <https://raw.githubusercontent.com/ajuvercr/js-runner/master/ontology.ttl>.
> ```
3. Package invocation i.e. in which package the func() is introduced
>```
> <> :install [
>  a :GitInstall;
>  :url <https://github.com/USERNAME/REPOSITORY>;
>  :build "npm install; npm run build";
>].
>```
>
4. Processor arguments

> ```
> js:Func a js:JsProcess;
>  # PATH/TO/Module 
>  js:file <../src/processors.js>;
>  # declare func() is a instance of js:Func, which is a js:JsProcess
>  js:function "func";
>  # relative path to root of JS program
>  js:location <../>;
>  # js:func parameter mappings
>  js:mapping [
>    a fno:Mapping;
>    fno:parameterMapping [
>      a fnom:PositionParameterMapping ;
>      fnom:functionParameter js:readableStream;
>      fnom:implementationParameterPosition "0"^^xsd:int
>    ], [
>      a fnom:PositionParameterMapping ;
>      fnom:functionParameter js:writeableStream ;
>      fnom:implementationParameterPosition "1"^^xsd:int
>    ]
>  ] .
> ```

5. Processor Shape
> ```
> # SHACL shape restriction on the js:JSProcess resource instance
> js:JsProcessorShape a sh:NodeShape; 
>  sh:targetClass js:JsProcess;
>  sh:property [
>    sh:dataType xsd:string;
>    sh:path js:file;
>    sh:name "Path to main main entry point"
>  ], [
>    sh:dataType xsd:iri; 
>    sh:path js:location;
>    sh:name "Location to root of JS program"
>  ], [
>    sh:dataType xsd:string;
>    sh:path js:function;
>    sh:name "Name of the JS function to execute"
>  ], [
>    sh:class fno:Mapping;
>    sh:path js:mapping;
>    sh:name "Mapping for the function arguments"
>  ].
>```

Concatenate 1-5 codes into a file named with .ttl extension. Now the func processor is configured. 

**How to pipe Connector Architecture processors?** 

The synergy between readable and writeable streams is a fundamental concept in data streaming.
In Connector Architecture, stream piping are achieved through [pipeline](https://github.com/TREEcg/connector-architecture/wiki/Pipeline) configuration. The output Writeable stream of a process will become the input Readable stream of another, when the two processors are bundled in a pipeline.

*pipeline configuration*

```
#<--PREFIX-->
#<--Processor import>
#<--Stream bundle-->
[] a js:JsChannel;
  :reader <pre-func-process/reader-js>;
  :writer <pre-func-process/writer-js>.

[] a js:JsChannel;
  :reader <func-process/reader-js>;
  :writer <func-process/writer-js>.

<pre-func-process/reader-js> a :JsReaderChannel.
<pre-func-process/writer-js> a :JsWriterChannel.

<func-process/reader-js> a :JsReaderChannel.
<func-process/writer-js> a :JsWriterChannel

[] a js:PreFunc;
  js:output <pre-func-process/writer-js>.

[] a js:Func;
  js:readableStream <pre-func-process/reader-js>;
  js:writeableStream <func-process/writer-js>.

[] a js:PostFunc;
  js:input <func-process/reader-js>.
```
To interpret the above in words, processor js:PreFunc communicates with processor js:Func via JSChannel (js:JsChannel).
js:PreFunc generates <pre-func-process/writer-js> data stream as output to be consumed by processor js:Func
as input <pre-func-process/reader-js>, and likewise js:Func outputs <func-process/writer-js> for processor js:PostFunc.

*Runner and Chanel*

In this tutorial, we will only use JS runner to execute the processors. Naturally, Connector Architecture does not limit to this. The main focus of Connector Architecture is on enabling different data processors to work together effectively,
regardless of the specific technologies or frameworks they are built upon (e.g. a JS stream instance that feeds into a Websoket instance).

More detail can be found at [Connector Architecture Wiki](https://github.com/TREEcg/connector-architecture/wiki)

**Side notes**

> Caution: Connector Architecture implements its own Stream types. Keep in mind that they are not derived from NodeJS stream! More details can be found [connector-types](https://github.com/TREEcg/connectors/tree/main/packages/types).

> In practice, data chunks of a Connector Architecture stream in a processor are better to be of type *string* in oder to make the processor reusable beyond JavaScript.

> - Connector Architecture Readable stream - `Stream`
    >    - listen to the `Stream.on('data')` event to process each data chunk
    >    - use `Stream.on('end')` event to
           >        - do data post-processing e.g. making numbers count
           >        - close the `Writer` writeable stream, to which processed data have been pushed, if desired.
> - Connector Architecture Write stream - `Writer`
    >    - use `Writer.push()` to push a chunk of data
---

##Substring fragmentation pipeline##

The substring fragmentation pipeline we are about to implement consists of 5 JS/TS processors. 
Long story short, you do not really need to create any processor on your own but simply compile the existing ones into your pipeline.

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

### Substring fragmentation processors

- [SPARQL query processor](-bucketizer-index-proc/blob/main/steps/querySparql.ttl)
    - fetch SPARQL query result from [ERA SPARQL endpoint](https://linked.ec-dataplatform.eu/sparql) using Comunica queryEngine
    >```
    > # an exmaple Writeable data chunk
    > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://www.w3.org/2000/01/rdf-schema#label> "FR9900004812" .
    > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://www.w3.org/2000/01/rdf-schema#label> "Technical change" .
    > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://data.europa.eu/949/opName> "Technical change" .
    > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://data.europa.eu/949/uopid> "FR9900004812" .
    >```
    
- [Sdsity processor](https://github.com/ajuvercr/sds-processors/blob/master/sdsify.ttl)
    - serialize quads with SDS vocabularies
    >```
    > # an exmaple Writeable data chunk
    > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://www.w3.org/2000/01/rdf-schema#label> "FR9900004812" .
    > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://www.w3.org/2000/01/rdf-schema#label> "Technical change" .
    > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://data.europa.eu/949/opName> "Technical change" .
    > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://data.europa.eu/949/uopid> "FR9900004812" .
    > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://www.w3.org/2000/01/rdf-schema#label> "PL04438" .
    > _:n3-4532 <https://w3id.org/sds#payload> <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> .
    > _:n3-4532 <https://w3id.org/sds#stream> <https://w3id.org/sds#Stream> .
    >```
- [Bucketization processor](https://github.com/ajuvercr/sds-processors/blob/master/2_bucketstep.ttl)
  - bucketize quads based on the value of propertyPath defined in `config.json` for substring search
  >```
  > # an exmaple Writeable data chunk
  > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://www.w3.org/2000/01/rdf-schema#label> "FR9900004812" .
  > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://www.w3.org/2000/01/rdf-schema#label> "Technical change" .
  > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://data.europa.eu/949/opName> "Technical change" .
  > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://data.europa.eu/949/uopid> "FR9900004812" .
  > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/PL04438> <http://www.w3.org/2000/01/rdf-schema#label> "PL04438" .
  > _:n3-4532 <https://w3id.org/sds#payload> <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> .
  > _:n3-4532 <https://w3id.org/sds#stream> <https://w3id.org/sds#Stream> .
  > _:df_2_573 <https://w3id.org/sds#payload> <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> .
  > _:df_2_573  <https://w3id.org/sds#bucket> <t> .
  > _:df_2_573  <https://w3id.org/sds#stream> <http://example.com/test> .
  > _:df_2_573  <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <https://w3id.org/sds#Member> .
  >```

- [SDS to TREE processor](https://github.com/jiaoxlong/substring-bucketizer-index-proc/blob/main/steps/sds_to_tree.ttl)
    - remodel the quads according to the TREE specification
>```
  > # an exmaple Writeable data chunk
  > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://www.w3.org/2000/01/rdf-schema#label> "FR9900004812" .
  > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://www.w3.org/2000/01/rdf-schema#label> "Technical change" .
  > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://data.europa.eu/949/opName> "Technical change" .
  > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> <http://data.europa.eu/949/uopid> "FR9900004812" .
  > <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/PL04438> <http://www.w3.org/2000/01/rdf-schema#label> "PL04438" .
  > _:n3-4532 <https://w3id.org/sds#payload> <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> .
  > _:n3-4532 <https://w3id.org/sds#stream> <https://w3id.org/sds#Stream> .
  > _:df_2_2019 <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <https://w3id.org/tree#SubstringRelation> .
  > _:df_2_2019 <https://w3id.org/tree#node> <https://treecg.github.io/demo_data/era/t> .
  > _:df_2_2019 <https://w3id.org/tree#value> "t" .
  > _:df_2_2020 <https://w3id.org/sds#payload> <http://data.europa.eu/949/functionalInfrastructure/operationalPoints/FR9900004812> .
  > _:df_2_2020 <https://w3id.org/sds#bucket> <https://treecg.github.io/demo_data/era/t> .
  > _:df_2_2020 <https://w3id.org/sds#stream> <http://example.com/test> .
  > _:df_2_2020 <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <https://w3id.org/sds#Member> .
  > <https://treecg.github.io/demo_data/era/t> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <https://w3id.org/tree#Node> .
  > _:df_2_2019 <https://w3id.org/tree#path> <http://data.europa.eu/949/opName> .
  >```
- [Ingestion processor](https://github.com/jiaoxlong/substring-bucketizer-index-proc/blob/main/steps/ingest.ttl)
    - per chunk of quads from stream
        - asynchronously allocate to its corresponding bucket (fragment)
        - add quads of tree:remainingItems count
        - write/append to file, defaults to turtle format.


### Pipeline execution

```shell
# copy the pre-prepared pipeline configuration and the config file to your repository
cp pipeline.ttl config.json <YOUR-REPOSITORY>
# run pipeline
cd connector-architecture/runner/js-runner; node ./bin/js-runner.js ../../<YOUR-REPOSITORY>/pipeline.ttl 
```
By now you should have a sense about how Connector Architecture works.

## Contact 

Do you have a question? Please do not hesitate to contact us or create an [issue](https://github.com/TREEcg/connector-architecture/issues).

- Arthur.Vercruysse@UGent.be
- Jiao.Long@UGent.be

[1]: Dedecker, R., Delva, H., Colpaert, P., Verborgh, R. (2021). A File-Based Linked Data Fragments Approach to Prefix Search. In: Brambilla, M., Chbeir, R., Frasincar, F., Manolescu, I. (eds) Web Engineering. ICWE 2021. Lecture Notes in Computer Science(), vol 12706. Springer, Cham. https://doi.org/10.1007/978-3-030-74296-6_5

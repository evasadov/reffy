# Reffy

<img align="right" width="256" height="256" src="images/reffy-512.png" alt="Reffy, represented as a brave little worm with a construction helmet, ready to crawl specs">

Reffy is a **Web spec crawler** tool. It is notably used to update [Webref](https://github.com/w3c/webref#webref) every 6 hours.

The code features a generic crawler that can fetch Web specifications and generate machine-readable extracts out of them. Created extracts include lists of CSS properties, definitions, IDL, links and references contained in the specification.

## How to use

### Pre-requisites

To install Reffy, you need [Node.js](https://nodejs.org/en/) 14 or greater.

### Installation

Reffy is available as an NPM package. To install the package globally, run:

```bash
npm install -g reffy
```

This will install Reffy as a command-line interface tool.

The list of specs crawled by default evolves regularly. To make sure that you run the latest version, use:

```bash
npm update -g reffy
```

### Launch Reffy

Reffy crawls requested specifications and runs a set of processing modules on the content fetched to create relevant extracts from each spec. Which specs get crawled, and which processing modules get run depend on how the crawler gets called. By default, the crawler crawls all specs defined in [browser-specs](https://github.com/w3c/browser-specs/) and runs all core processing modules defined in the [`browserlib`](https://github.com/w3c/reffy/tree/main/src/browserlib) folder.

Crawl results will either be returned to the console or saved in individual files in a report folder when the `--output` parameter is set.

Examples of information that can be extracted from the specs:

1. Generic information such as the title of the spec or the URL of the Editor's Draft. This information is typically copied over from [browser-specs](https://github.com/w3c/browser-specs/).
2. The list of terms that the spec defines, in a format suitable for ingestion in cross-referencing tools such as [ReSpec](https://respec.org/xref/).
3. The list of IDs, the list of headings and the list of links in the spec.
4. The list of normative/informative references found in the spec.
5. Extended information about WebIDL term definitions and references that the spec contains
6. For CSS specs, the list of CSS properties, descriptors and value spaces that the spec defines.

The crawler can be fully parameterized to crawl a specific list of specs and run a custom set of processing modules on them. For example:

- To extract the raw IDL defined in Fetch, run:
  ```bash
  reffy --spec fetch --module idl
  ```
- To retrieve the list of specs that the HTML spec references, run (noting that crawling the HTML spec takes some time due to it being a multipage spec):
  ```bash
  reffy --spec html --module refs
  ```
- To extract the list of CSS properties defined in CSS Flexible Box Layout Module Level 1, run:
  ```bash
  reffy --spec css-flexbox-1 --module css
  ```
- To extract the list of terms defined in WAI ARIA 1.2, run:
  ```bash
  reffy --spec wai-aria-1.2 --module dfns
  ```
- To run an hypothetical `extract-editors.mjs` processing module and create individual spec extracts with the result of the processing under an `editors` folder for all specs in browser-specs, run:
  ```bash
  reffy --output reports/test --module editors:extract-editors.mjs
  ```

You may add `--terse` (or `-t`) to the above commands to access the extracts directly.

Run `reffy -h` for a complete list of options and usage details.


Some notes:

* The crawler may take a few minutes, depending on the number of specs it needs to crawl.
* The crawler uses a local cache for HTTP exchanges. It will create and fill a `.cache` subfolder in particular.
* If you cloned the repo instead of installing Reffy globally, replace `reffy` width `node reffy.js` in the above example to run Reffy.


## Additional tools

Additional CLI tools in the `src/cli` folder complete the main specs crawler.


### WebIDL parser

The **WebIDL parser** takes the relative path to an IDL extract and generates a JSON structure that describes WebIDL term definitions and references that the spec contains. The parser uses [WebIDL2](https://github.com/darobin/webidl2.js/) to parse the WebIDL content found in the spec. To run the WebIDL parser: `node src/cli/parse-webidl.js [idlfile]`

To create the WebIDL extract in the first place, you will need to run the `idl` module in Reffy, as in:

```bash
reffy --spec fetch --module idl > fetch.idl
```

### Parsed WebIDL generator

The **Parsed WebIDL generator** takes the results of a crawl as input and applies the WebIDL parser to all specs it contains to create JSON extracts in an `idlparsed` folder. To run the generator: `node src/cli/generate-idlparsed.js [crawl folder] [save folder]`


### WebIDL names generator

The **WebIDL names generator** takes the results of a crawl as input and creates a report per referenceable IDL name, that details the complete parsed IDL structure that defines the name across all specs. To run the generator: `node src/cli/generate-idlnames.js [crawl folder] [save folder]`


### Crawl results merger

The **crawl results merger** merges a new JSON crawl report into a reference one. This tool is typically useful to replace the crawl results of a given specification with the results of a new run of the crawler on that specification. To run the crawl results merger: `node src/cli/merge-crawl-results.js [new crawl report] [reference crawl report] [crawl report to create]`


### Analysis tools

Starting with Reffy v5, analysis tools that used to be part of Reffy's suite of tools to study extracts and create human-readable reports of potential spec anomalies migrated to a companion tool named [Strudy](https://github.com/w3c/strudy). The actual reports get published in a separate [w3c/webref-analysis](https://github.com/w3c/webref-analysis) repository as well.


### WebIDL terms explorer

See the related **[WebIDLPedia](https://dontcallmedom.github.io/webidlpedia)** project and its [repo](https://github.com/dontcallmedom/webidlpedia).


## Technical notes

Reffy should be able to parse most of the W3C/WHATWG specifications that define CSS and/or WebIDL terms (both published versions and Editor's Drafts), and more generally speaking specs authored with one of [Bikeshed](https://tabatkins.github.io/bikeshed/) or [ReSpec](https://respec.org/docs/). Reffy can also parse certain IETF specs to some extent, and may work with other types of specs as well.

### List of specs to crawl

Reffy crawls specs defined in [w3c/browser-specs](https://github.com/w3c/browser-specs/). If you believe a spec is missing, please check the [Spec selection criteria](https://github.com/w3c/browser-specs/#spec-selection-criteria) and create an issue (or prepare a pull request) against the [w3c/browser-specs](https://github.com/w3c/browser-specs/) repository.

### Crawling a spec

Given some spec info, the crawler basically goes through the following steps:

1. Load the URL through Puppeteer.
2. If the document contains a "head" section that includes a link whose label looks like "single page", go back to step 2 and load the target of that link instead. This makes the crawler load the single page version of multi-page specifications such as HTML5.
3. If the document is a multi-page spec without a "single page" version, load the individual subpage and add their content to the bottom of the first page to create a single page version.
4. If the document uses ReSpec, let ReSpec finish its generation work.
5. Run internal tools on the generated document to build the relevant information.

The crawler processes 4 specifications at a time. Network and parsing errors should be reported in the crawl results.

### Config parameters

The crawler reads parameters from the `config.json` file. Optional parameters:

* `cacheRefresh`: set this flag to `never` to tell the crawler to use the cache entry for a URL directly, instead of sending a conditional HTTP request to check whether the entry is still valid. This parameter is typically useful when developing Reffy's code to work offline.
* `resetCache`: set this flag to `true` to tell the crawler to reset the contents of the local cache when it starts.


## Contributing

Authors so far are [François Daoust](https://github.com/tidoust/) and [Dominique Hazaël-Massieux](https://github.com/dontcallmedom/).

Additional ideas, bugs and/or code contributions are most welcome. Create [issues on GitHub](https://github.com/w3c/reffy/issues) as needed!


## Licensing

The code is available under an [MIT license](LICENSE).

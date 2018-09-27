# Hypermedia Pipeline

This project provides helper functions and default implementations for creating Hypermedia Processing Pipelines.

It uses reducers and continuations to create a simple processing pipeline that can pre-and post-process HTML, JSON, and other hypermedia.

# Status

[![codecov](https://img.shields.io/codecov/c/github/adobe/hypermedia-pipeline.svg)](https://codecov.io/gh/adobe/hypermedia-pipeline)
[![CircleCI](https://img.shields.io/circleci/project/github/adobe/hypermedia-pipeline.svg)](https://circleci.com/gh/adobe/hypermedia-pipeline)
[![GitHub license](https://img.shields.io/github/license/adobe/hypermedia-pipeline.svg)](https://github.com/adobe/hypermedia-pipeline/blob/master/LICENSE.txt)
[![GitHub issues](https://img.shields.io/github/issues/adobe/hypermedia-pipeline.svg)](https://github.com/adobe/hypermedia-pipeline/issues)
[![npm](https://img.shields.io/npm/dw/@adobe/hypermedia-pipeline.svg)](https://www.npmjs.com/package/@adobe/hypermedia-pipeline) [![Greenkeeper badge](https://badges.greenkeeper.io/adobe/hypermedia-pipeline.svg)](https://greenkeeper.io/)

## Anatomy of a Pipeline

A pipeline consists of following main parts:

- pre-processing functions
- the main response generating function
- an optional wrapper function
- post-processing functions

Each step of the pipeline is processing a single payload object, that will slowly accumulate the `return` values of the functions above through `Object.assign`.

See below for the anatomy of a payload.

Typically, there is one pipeline for each content type supported and pipeline are identified by file name, e.g.

- `html.pipe.js` – creates HTML documents with the `text/html` content-type
- `json.pipe.js` – creates JSON documents with the `application/json` content-type

### Building a Pipeline

A pipeline builder can be created by creating a CommonJS module that exports a function `pipe` which accepts following arguments and returns a Pipeline function.

- `cont`: the main function that will be executed as a continuation of the pipeline
- `params`: a map of parameters that are interpreted at runtime
- `secrets`: a map of protected configuration parameters like API keys that should be handled with care. By convention, all keys in `secret` are in ALL_CAPS_SNAKE_CASE.
- `logger`: a [Winston](https://www.github.com/winstonjs/winston) logger

This project's main entry provides a helper function for pipeline construction and a few helper functions, so that a basic pipeline can be constructed like this:

```javascript
// the pipeline itself
const pipeline = require("@adobe/hypermedia-pipeline");
// helper functions and log
const { adaptOWRequest, adaptOWResponse, log } = require('@adobe/hypermedia-pipeline/src/defaults/default.js');

module.exports.pipe = function(cont, params, secrets, logger = log) {
    logger.log("debug", "Constructing Custom Pipeline");

    return pipeline()
        .pre(adaptOWRequest)   // optional: turns OpenWhisk-style arguments into a proper payload
        .once(cont)            // required: execute the continuation function
        .post(adaptOWResponse) // optional: turns the Payload into an OpenWhisk-style response
}
```

In a typical pipeline, you will add additional processing steps as `.pre(require('some-module'))` or as `.post(require('some-module'))`.

### The Main Function

The main function is typically a pure function that converts the `request`, `context`, and `content` properties of the payload into a `response` object.

In most scenarios, the main function is compiled from a template in a templating language like HTL, JST, or JSX.

Typically, there is one template (and thus one main function) for each content variation of the file type. Content variations are identified by a selector (the piece of the file name before the file extension, e.g. in `example.navigation.html` the selector would be `navigation`). If no selector is provided, the template is the default template for the pipeline.

Examples of possible template names include:

- `html.jsx` (compiled to `html.js`) – default for the HTML pipeline
- `html.navigation.jst` (compiled to `html.navigation.js`) – renders the navigation
- `dropdown.json.js` (not compiled) – creates pure JSON output
- `dropdown.html.htl` (compiled to `dropdown.html.js`) – renders the dropdown component


### (Optional) The Wrapper Function

Sometimes it is neccessary to pre-process the payload in a template-specific fashion. This wrapper function (often called "Pre-JS" for brevity sake) allows the full transformation of the pipeline's payload.

Compared to the pipeline-specific pre-processing functions which handle the request, content, and response, the focus of the wrapper function is implementing business logic needed for the main template function. This allows for a clean separation between:

1. presentation (in the main function, often expressed in declarative templates)
2. business logic (in the wrapper function, often expressed in imperative code)
3. content-type specific implementation (in the pipeline, expressed in functional code)

A simple implementation of a wrapper function would look like this:

```javascript
// All wrapper functions must export `pre`
// The functions takes following arguments:
// - `cont` (the continuation function, i.e. the main template function)
// - `payload` (the payload of the pipeline)
module.exports.pre = (cont, payload) => {
    const {request, content, context, response} = payload;
    
    // modifying the payload content before invoking the main function
    content.hello = 'World';
    const modifiedpayload = {request, content, context, response};

    // invoking the main function with the new payload. Capturing the response
    // payload for further modification

    const responsepayload = cont(modifiedpayload);

    // Adding a value to the payload response
    const modifiedresponse = modifiedpayload.response;
    modifiedresponse.hello = 'World';

    return Object.assign(modifiedpayload, modifiedresponse);
}
```

### Pre-Processing Functions

Pre-Processing functions are meant to:

- parse and process request parameters
- fetch and parse the requested content
- transform the requested content

### Post-Processing Functions

Post-Processing functions are meant to:

- process and transform the response

## Anatomy of the Payload

Following main properties exist:

- `request`
- `content`
- `response`
- `context`
- `error`

### The `request` object

- `params`: a map of request parameters
- `headers`: a map of HTTP headers

### The `content` object

- `body`: the unparsed content body as a `string`
- `mdast`: the parsed [Markdown AST](https://github.com/syntax-tree/mdast)
- `meta`: a map metadata properties, including
  - `title`: title of the document
  - `intro`: a plain-text introduction or description
  - `type`: the content type of the document
  - `image`: the URL of the first image in the document
- `htast`: the HTML AST
- `sections[]`: The main sections of the document, as an enhanced MDAST
- `html`: a string of the content rendered as HTML
- `children`: an array of top-level elements of the HTML-rendered content

### `content.sections` in Detail

The default pipeline extracts sections from a Markdown document, using both "thematic breaks" like `***` or `---` and embedded YAML blocks as section markers. If no sections can be found in the document, the entire `content.mdast` will be identically to `content.sections[0]`.

`content.sections` is an Array of `section` nodes, with `type` (String) and `children` (array of Node) properties. In addition, each section has a `types` attribute, which is an array of derived content types. Project Helix (and Hypermedia Pipeline) uses implied typing over declared content typing, which means it is not the task of the author to explicitly declare the content type of a section or document, but rather have the template interpret the contents of a section to understand the type of content it is dealing with.

The `types` property is an array of string values that describes the type of the section based on the occurence of child nodes. This makes it easy to copy the value of `types` into the `class` attribute of an HTML element, so that CSS expressions matching types of sections can be written with ease. Following patterns of `type` values can be found:

- `has-<type>`: for each type of content that occurs at least once in the section, e.g. has-heading
- `is-<type>-only`: for sections that only have content of a single type, e.g. is-image-only
- `is-<type-1>-<type-2>-<type3>`, `is-<type-1>-<type-2>`, and `is-<type-1>` for the top 3 most frequent types of children in the section. For instance a gallery with a heading and description would be `is-image-paragraph-heading`. You can infer additional types using [`utils.types`](#infer-content-types-with-utilstypes).

Each section has additional content-derived metadata properties, in particular:

- `title`: the value of the first headline in the section
- `intro`: the value of the first paragraph in the section
- `image`: the URL of the first image in the section
- `meta`: the parsed YAML metadata of the section (as an object)



### The `response` object

- `body`: the unparsed response body as a `string`
- `headers`: a map of HTTP response headers
- `status`: the HTTP status code

### The `context` object

TBD: used for stuff that is neither content, request, or response

### The `error` object

This object is only set when there has been an error during pipeline processing. Any step in the pipeline may set the `error` object. Subsequent steps should simply skip any processing if they encounter an `error` object.

Alternatively, steps can attempt to handle the `error` object, for instance by generating a formatted error message and leaving it in `response.body`.

The only known property in `error` is

- `message`: the error message

## Utilities

### Infer Content Types with `utils.types`

In addition to the automatically inferred content types for each section, `utils.types` provides a `TypeMatcher` utility class that allows matching section content against a simple expression language and thus enrich the `section[].types` values.


```javascript
const TypeMatcher = require('@adobe/hypermedia-pipeline').utils.types;

const matcher = new TypeMatcher(content.sections);
matcher.match('^heading', 'starts-with-heading');
content.sections = matcher.process();
```

In the example above, all sections that have a `heading` as the first child will get the value `starts-with-heading` appended to the `types` array. `^heading` is an example of the content expression language, which allows matching content against a simple regular expression-like syntax.

##### Content Expression Language

* `^heading` – the first element is a `heading`
* `paragraph$` – the last element is a `paragraph`
* `heading image+` – a `heading` followed by one or more `image`s
* `heading? image` – an optional `heading` followed by one `image`
* `heading paragraph* image` – a `heading` followed by any number of `paragraph`s (also no paragraphs at all), followed by an `image`
* `(paragraph|list)` – a `paragraph` or a `list`
* `^heading (image paragraph)+$` – one `heading`, followed by pairs of `image` and `paragraph`, but at least one
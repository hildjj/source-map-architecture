# Source-map Architecture

This document will describe some potential ways forward for the
[source-map](https://github.com/mozilla/source-map) code base.

## Current State

Describe the current state:

- One big codebase
- Async-only
- WASM not adequately easy to build
- Summarize major [bugs](https://github.com/mozilla/source-map/issues)
- Summarize open [pull requests](https://github.com/mozilla/source-map/pulls)
- No release in 3 years
- Consumer interface built in to node since (v13.7.0, v12.17.0), but **very** slow, still marked experimental, and requires a command-line flag.

## Requirements

- Works in supported major browsers up to N years back
- Works in [maintained versions](https://github.com/nodejs/Release) of node.js
  (version 14.x as of the moment this was written)
- Different parts of the system can be used independently
- Minimal impact to existing users
- Performance matters
- Synchronous path needed, can have worse performance
- Overall size matters somewhat; for example, don't require the Unicode
  IdnaMappingTable to be bundled unless absolutely required.  It's currently
  pulled in by [tr46](https://github.com/jsdom/tr46/), required by
  [whatwg-url](https://github.com/jsdom/whatwg-url), required by
  [./lib/url-browser.js](https://github.com/mozilla/source-map/blob/58819f09018d56ef84dc41ba9c93f554e0645169/lib/url-browser.js).
  None of that should be pulled in for Node, and the browsers should really
  fix the bugs
  ([Firefox](https://bugzilla.mozilla.org/show_bug.cgi?id=1374505),
  [Chrome](https://bugs.chromium.org/p/chromium/issues/detail?id=734880)) that
  require it at all.
- Any use of WASM should be easier on the web than it currently is

## Use Cases

- Generation: tools that create JS from some original input
- Consumption: tools that need to map JS back to original input
- Generate/consume without formatting: tools that need to generate output from
  input, then use the mapping immediately in the same process, without needing
  to go through the JSON format.  Example: eslint preprocessors
- Multiple levels of mapping for source that is transformed more than once?

## Sub-libraries

One approach would be to break the existing code into sub-libraries.

- In-memory data structure based on the guts of SourceMapGenerator
- Generate data structure, without formatting.  Based on SourceNode.
- Format generated data structure to JSON
- Parse JSON to data structure
- Consume data structure, based on the guts of SourceMapConsumer
- ESlint configuration for all of the above
- Benchmarks

It might be that the consumption interface is small enough on top of the data
structure that those interfaces can be combined.

There will likely need to be several different parsers that can be swapped in.
That's where the WASM is used at the moment.  It may be that a JS-only parser
is fast enough for most cases, and vastly easier to package and use
synchronously.

The existing project could then be turned into glue logic that ties all of
those packages together, smoothing over the new interfaces into something that
is as small a change to the API as possible from the last released version.

Alternately, starting a whole new set of projects would allow for all of the
breaking changes at once, then we could just deprecate `source-map`.

Once we have something working and have ensured it is fast, we could
contribute a patch to Node to fix their perf issue.  Someone might ping the
upstream Chromium owners to see if they care as well.  Note: this code is
currently BSD-licensed to the Mozilla Foundation.

## Testing

Consider how to test each of the modules without having all of the modules
have devDependencies on each other, or making those dependencies not terribly
confusing.

## Other changes

- ES2021 as syntax baseline
- Move to esm modules? Will make any WASM more straightforward with top-level
  await.
- Might be "easiest" to have monorepo for all of the projects?
- `master` -> `main`

## Notes

I went down the rathole of looking at different compression mechanisms for
`tr46`, and though I could save some space, it wasn't enough to be worth the
complexity, imo.  The path was to put all of the replacement strings into a
flat JSON array, where `offset` is the offset into that array of a given
replacement.  Then using
[unicode-trie](https://github.com/foliojs/unicode-trie/) encoding the mapping
codepoint->(offset|(8192-type)).

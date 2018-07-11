# Project References: Built-in Scalability for TypeScript

## Introduction

TypeScript has scaled up to projects of hundreds of thousands of lines, but doesn't natively support this kind of scale as a built-in behavior. Teams have developed workarounds of varying effectiveness and there isn't a standardized way to represent large projects. While we've greatly improved the performance of the typechecker over time, there are still hard limits in terms of how fast TS can reasonably get,
and constraints like 32-bit address space which prevent the language service from scaling up "infinitely" in interactive scenarios.

Intuitively, changing one line of JSX in a front-end component should *not* require re-typechecking the entire core business logic component of a 500,000 LOC project. The goal of *project references* is to give developers tools to *partition* their code into smaller blocks. By enabling tools to operate on smaller chunks of work at a time, we can improve responsiveness and tighten the core development loop.

We believe our current architecture has very few remaining "free lunches" in terms of drastic improvmenets in performance or memory consumption. Instead, this partitioning is an *explicit trade-off* that increases speed at the expense of some upfront work. Developers will have to spend some time reasoning about the dependency graph of their system, and certain interactive features (e.g. cross-project renames) may be unavailable until we further enhance tooling.

We'll identify key constraints imposed by this system and establish guidelines for project sizing, directory structure, and build patterns.

## Scenarios

There are three main scenarios to consider.

### Relative Modules

Some projects extensively use *relative imports*. These imports unambiguously resolve to another file on disk. Paths like `../../core/utils/otherMod` would be common to find, though flatter directory structures are usually preferred in these repos.

#### Example
 Here's an example from Khan Academy's perseus project:

Adapted from https://github.com/Khan/perseus/blob/master/src/components/graph.jsx
```js
const Util = require("../util.js");
const GraphUtils = require("../util/graph-utils.js");
const {interactiveSizes} = require("../styles/constants.js");
const SvgImage = require("../components/svg-image.jsx");
```

#### Observations

While the directory structure *implies* a project stucture, it's not necessarily definitive. In the Khan Academy sample above, will can infer that `util`, `styles`, and `components` would probably be their own project. But it's also possible that these directories are quite small and would actually be grouped into one build unit.

### Mono-repo

A mono-repo consists of a number of modules that are imported via *non-relative* paths. Imports from sub-modules (e.g. `import * as C from 'core/thing`) may be common. Usually, but not always, each root module is actually published on NPM.

#### Example

Adapted from https://github.com/angular/angular/blob/master/packages/forms/src/validators.ts
```js
import {InjectionToken, ɵisObservable as isObservable, ɵisPromise as isPromise} from '@angular/core';
import {forkJoin} from 'rxjs/observable/forkJoin';
import {map} from 'rxjs/operator/map';
import {AbstractControl, FormControl} from './model';
```

#### Observations

The unit of division is *not* necessarily the leading part of the module name. `rxjs`, for example, actually compiles its subparts (`observable`, `operator`) separately, as does any scoped package (e.g. `@angular/core`).

### Outfile

TypeScript can concatenate its input files into a single output JavaScript file. Reference directives, or file ordering in tsconfig.json, create a deterministic output order for the resulting file. This is rarely used for new projects, but is still prevelant among older codebases (including TypeScript itself).

#### Example

https://github.com/Microsoft/TypeScript/blob/master/src/compiler/tsc.ts
```ts
/// <reference path="program.ts"/>
/// <reference path="watch.ts"/>
/// <reference path="commandLineParser.ts"/>
```
https://github.com/Microsoft/TypeScript/blob/master/src/harness/unittests/customTransforms.ts
```ts
/// <reference path="..\..\compiler\emitter.ts" />
/// <reference path="..\harness.ts" />
```

#### Observations

Some solutions using this configuration will be loading each `outFile` via a separate `script` tag (or equivalent), but others (e.g. TypeScript itself) require concatenation of the prior files because they're building monolithic outputs.

## Project References: A new unit of isolation

Some critical observations from interacting with real projects:
 * TypeScript is usually "fast" (< 5-10s) when checking projects under 50,000 LOC of implementation (non-.d.ts) code
 * .d.ts files, especially under `skipLibCheck`, are almost "free" in terms of their typechecking and memory cost
 * Almost all software can be subdivided into components smaller than 50,000 LOC
 * Almost all large projects already impose some structuring of their files by directory in a way that produces moderately-sized subcomponents
 * Most edits occur in leaf-node or near-leaf-node components that do not require re-checking or re-emitting of the entire solution

Putting these together, if it were possible to only ever be typechecking one 50,000 LOC chunk of implementation code at once, there would be almost no "slow" interactions in an interactive scenario, and we'd almost never run out of memory.

We introduce a new concept, a *project reference*, that declares a new kind of dependency between two TypeScript compilation units where the dependent unit's implementation code is *not* checked; instead we simply load its `.d.ts` output from a deterministic location.

### Syntax

A new `references` option (TODO: Bikeshed!) is added to `tsconfig.json`:
```
{
  "extends": "../tsproject.json",
  "compilerOptions": {
    "outDir": "../bin",
    "references": [
      { "path": "../otherProject" }
    ]
  }
}
```

The `references` array specifies a set of other projects to reference from this project.
Each `references` object's `path` points to a `tsconfig.json` file or a folder containing a `tsconfig.json` file.
Other options may be added to this object as we discover their needs.

### Semantics

Project references change the following behavior:
 * When module resolution would resolve to a `.ts` file in a subdirectory of a project's `rootDir`, it *instead* resolves to a `.d.ts` file in that project's `outDir`
   * If this resolution fails, we can probably detect it and issue a smarter error, e.g. `Referenced project "../otherProject" is not built` rather than a simple "file not found"
 * Nothing else (TODO: so far?)

### Restrictions for Performance of Referenced Projects

To meaningfully improve build performance, we need to be sure to restrict the behavior of TypeScript when it sees a project reference.

Specifically, the following things should be true:
* **Never** read or parse the input .ts files of a referenced project
* **Only** the `tsconfig.json` of a referenced project should be read from disk
* Up-to-date checking should **not** require violating the above constraints

To keep these promises, we need to impose some restrictions on projects you reference.

 * `declaration` is automatically set to `true`. It is an error to try to override this setting
 * `rootDir` defaults to `"."` (the directory containing the `tsconfig.json` file), rather than being inferred from the set of input files
 * If a `files` array is provided, it must provide the names of *all* input files
   * Exception: Files included as part of type references (e.g. those in `node_modules/@types`) do not need to be specified
 * Any project that is referenced must itself have a `references` array (which may be empty).

#### Why `"declaration": true` ?

Project references improve build speed by using declaration files (.d.ts) in place of their implementation files (.ts).
So, naturally, any referenced project must have the `declaration` setting on.
This is implied by `"project": true`

#### Why alter `rootDir` ?

The `rootDir` controls how input files map to output file names. TypeScript's *default* behavior is to compute the *common source directory* of the complete graph of input files. For example, the set of input files `["src/a.ts", "src/b.ts"]` will produce the output files `["a.js", "b.js"]`,
but the set of input files `["src/a.ts", "b.ts"]` will produce the output files `["src/a.js", "b.js"]`.

Computing the set of input files requires parsing every root file and all of its references recursively,
which is expensive in a large project. But we can't change this behavior today without breaking existing projects in a bad way, so this change only occurs when the `references` array is provided.

### No Circularity

Naturally, projects may not form a graph with any circularity. (TODO: What problems does this *actually* cause, other than build orchestration nightmares?) If this occurs, you'll see an error message that indicates the circular path that was formed:

```
TS6187: Project references may not form a circular graph. Cycle detected:
    C:/github/project-references-demo/core/tsconfig.json ->
    C:/github/project-references-demo/zoo/tsconfig.json ->
    C:/github/project-references-demo/animals/tsconfig.json ->
    C:/github/project-references-demo/core/tsconfig.json
```

# `tsbuild`

This proposal is intentionally vague on how it would be used in a "real" build system. Very few projects scale past the 50,000 LOC "fast" boundary without introducing something other than `tsc` for compiling .ts code.

The user scenario of "You can't build `foo` because `bar` isn't built yet" is an obvious "Go fetch that" sort of task that a computer should be taking care of, rather than a mental burden on developers.

We expect that tools like `gulp`, `webpack`, etc, (or their respective TS plugins) will build in understanding of project references and correctly handle these build dependencies, including up-to-date checking.

To ensure that this is *possible*, we'll provide a *reference implementation* for a TypeScript build orchestration tool that demonstrates the following behaviors:
 * Fast up-to-date checking
 * Project graph ordering
 * Build parallelization
 * (TODO: others?)

This tool should use only public APIs, and be well-documented to help build tool authors understand the correct way to implement project references.

# TODO

Sections to fill in to fully complete this proposal
 * How you'd transition an existing project
   * Basically just drop `tsconfig.json` files in and then add references needed to fix build errors
 * Impact on `baseUrl`
   * Makes implementation difficult but effectively no end-user impact
 * Brief discussion of nesting projects (TL;DR it needs to be allowed)
 * Outline scenario of subdividing a projects that has grown "too large" into smaller projects without breaking consumers
 * Figure out the Lerna scenario
   * Available data point (N = 1) says they wouldn't need this because their build is already effectively structured this way
   * Find more examples or counterexamples to better understand how people do this
 * Do we need a `dtsEmitOnly` setting for people who are piping their JS through e.g. webpack/babel/rollup?
   * Maybe `references` + `noEmit` implies this


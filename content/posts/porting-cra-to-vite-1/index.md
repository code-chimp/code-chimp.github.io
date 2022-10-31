---
title: "Porting a Create React App application to Vite Pt. 1: Base Project"
description: Porting a Create React App application to Vite part one - base project
summary: As of this writing, Vite promises a faster and more performant developer experience over the tried-and-true Create React App template. This is my experience porting one of my vanilla projects to Vite.
date: 2022-10-31T00:01:01-05:00
draft: true

categories:
- Project Architecture
- New Tech

tags:
- configuration
- NodeJS
---

## Why?

[Vite's][vit] primary value proposition is a faster, more performant developer experience. Vite is powered by [Rollup][rol] and
[ESBuild][esb], a JavaScript bundler written in [Go](https://go.dev), while [Create React App][cra] leverages the stable,
battle tested, combination of [WebPack][webp] and the [Babel][bab] transpiler.

**So, if I am happy with Create React App then why this port?**

- Evaluate the Hype - I like to try out the new tech and see if it can improve my workflow
- Pet Peeve - CRA does not give you full control of your Jest configuration, plus it pollutes `package.json` with the configuration values that you are allowed to specify
- To do it - I learn a lot from little experiments like this

I am breaking this into two parts in an attempt to keep the information digestible. This first part will focus on getting
the application to run in the browser, identical to the CRA version, along with the same quality assurance tooling that
mostly comes out of the box with CRA.  Part two will focus on unit tests and enforcing test coverage.

As always use your own judgement before pursuing the next-shiny-thing, because for all I know CRA may soon be able to
leverage the new [TurboPack][tpack] proposed WebPack successor and re-claim the speed title.

## Preflight

### Create a Clean Shell

Let's begin by scaffolding [the new project][prj] from Vite's minimal [React + TypeScript][tmp] template:

```shell
yarn create vite vite-redux-seed --template react-ts

cd vite-redux-seed

yarn
```

[**>> First Commit**][s0]

### Add Static Code Analysis

Setting up some basic static analysis tooling is a fairly easy, and inexpensive, way of ensuring basic code quality and
consistency. As an added bonus there are plugins for ESLint such as [jsx-a11y][a11y] to alert me when I am inadvertently
creating accessibility issues in my application. I highly recommend the [typesync][tsyn] command to save time by
automatically finding any missing typings for your packages.

```shell
yarn add --dev npm-run-all browserslist sass prettier eslint stylelint \
@typescript-eslint/eslint-plugin @typescript-eslint/parser eslint-config-prettier \
eslint-plugin-jsx-a11y eslint-plugin-react eslint-plugin-react-hooks \
stylelint-config-prettier stylelint-config-recommended-scss stylelint-scss \
postcss

# add missing typings to package.json
npx typesync

# install the typings found by typesync
yarn
```

Now we can pull most of our code-quality configuration files straight over from the old project:

- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-1/.browserslistrc"
      title=".browserslistrc" >}}: browsers we are promising to support
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-1/.editorconfig"
      title=".editorconfig" >}}: basic text editor settings such as default line endings
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-1/.eslintignore"
      title=".eslintignore" >}}: files/folders we do not want ESLint to analyze
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-1/.prettierrc"
      title=".prettierrc" >}}: source code format rules
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-1/.prettierignore"
      title=".prettierignore" >}}: files/folders Prettier should ignore
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-1/.stylelintrc.json"
      title=".stylelintrc.json" >}}: stylesheet format rules

Create React App sets many defaults for ESLint that we will have to explicitly mimic ourselves in a new custom ESLint
config file:

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-1/.eslintrc.json"
           title=".eslintrc.json" >}}*
{{< highlight json >}}
{
  "root": true,
  "env": {
    "browser": true,
    "node": true
  },
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": "latest",
    "sourceType": "module"
  },
  "plugins": ["@typescript-eslint", "react", "react-hooks", "jsx-a11y"],
  "extends": [
    "eslint:recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    "plugin:jsx-a11y/recommended",
    "prettier"
  ],
  "rules": {
    "no-magic-numbers": [2, { "ignore": [-1, 0, 1, 2, 10, 100, 1000] }],
    "no-unused-vars": [2, { "vars": "local", "args": "after-used", "argsIgnorePattern": "_" }],
    "no-console": [
      "error",
      {
        "allow": ["error", "info", "warn"]
      }
    ]
  },
  "settings": {
    "react": {
      "version": "detect"
    }
  }
}
{{< /highlight >}}

Now that our tools are installed and configured let's artificially create an accessibility error by removing an **alt**
property from our image tag and verify it causes a linting error:

{{< figure
    src="linting_works.png"
    alt="displaying eslint accessibility error by removing `alt` prop from image tag"
    caption="Linting works"
    default="true" >}}

[**>> Second Commit**][s1]

As we can now verify that the linter is working why don't we take the next step and explicitly enforce our choices with
a [git hook][ghk]. First we will need a couple of packages:

1. [Husky][hsk] to easily create hooks for this project
2. [Pretty-Quick][pq] to automatically apply our predefined code formatting standards

```shell
yarn add --dev pretty-quick husky

npx husky install

npm pkg set scripts.prepare="husky install"
```

I have added a few scripts to help with project maintenance - the two highlighted below will ensure that all code is in
our defined format, and that any linting issues will cause an error:

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-1.5/package.json"
           title="package.json" >}}*
{{< highlight json "hl_lines=4 7" >}}
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "format": "pretty-quick --staged",
    "format:check": "prettier --check .",
    "format:fix": "prettier --write .",
    "lint": "run-p lint:*",
    "lint:ts": "eslint ./src/**/**.ts*",
    "lint:styles": "stylelint \"./src/**/*.scss\"",
    "fix:styles": "stylelint \"./src/**/*.scss\" --fix",
    "prepare": "husky install"
  },
{{< /highlight >}}

Once they are added to a **pre-commit** hook any commit that is attempted with style or code linting errors will
automatically be aborted:

```shell
npx husky add .husky/pre-commit "npm run format"
npx husky add .husky/pre-commit "npm run lint"
```

[**>> Third Commit**][s1_5]

## Porting the Application

Let's first make sure that the project will run, by replacing the contents of the `src` folder with everything
from [the vanilla CRA project][prj] - excluding tests and snapshots for now. We will need to rename `src/index.tsx`,
Create React App's default root component, to `src/main.tsx` which is Vite's default. Before trying to run it we
need to install all of our missing packages:

```shell
yarn add @fortawesome/fontawesome-svg-core @fortawesome/free-brands-svg-icons \
@fortawesome/free-regular-svg-icons @fortawesome/free-solid-svg-icons \
@fortawesome/react-fontawesome @reduxjs/toolkit bootstrap @popperjs/core \
react-bootstrap react-redux react-router-dom redux uuid

# add missing typings to package.json
npx typesync

# install the typings found by typesync
yarn
```

### .env Changes

I utilized [Create React App's .env][dotcra] file to store a sample third party API URL:

file: *{{< newtabref
           href="https://github.com/code-chimp/vanilla-redux-template/blob/main/.env"
           title=".env" >}}* **(old app)**
```text
REACT_APP_API_URI=https://jsonplaceholder.typicode.com
```

It will need to be changed to [Vite's format][dotvit]:

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-2/.env"
           title=".env" >}}*
```text
VITE_API_URI=https://jsonplaceholder.typicode.com
```

Let's also [add type information][dotsens] for our custom environment variable(s):

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-2/src/env.d.ts"
           title="/src/env.d.ts" >}}*
{{< highlight typescript >}}
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URI: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
{{< /highlight >}}

And finally update the reference in the service helper:

file: *{{< newtabref
           href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/helpers/service.ts"
           title="/src/helpers/service.ts" >}}*
{{< highlight typescript >}}
...
const apiUri = process.env.REACT_APP_API_URI;
...
{{< /highlight >}}

becomes:

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-2/src/helpers/service.ts"
           title="/src/helpers/service.ts" >}}*
{{< highlight typescript >}}
...
const apiUri = import.meta.env.VITE_API_URI;
...
{{< /highlight >}}

### First Test Run

Let's see how far that has gotten us:

```shell
yarn dev
```
{{< figure
    src="bootstrap_error.png"
    alt="browser displays Sass error, does not understand '~' import"
    caption="Ew, yuck"
    default="true" >}}

### Fix Bootstrap's Sass Import

It appears that we need to tell Vite how to resolve `~bootstrap` from `/src/styles/global.scss`:

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-2/vite.config.ts"
           title="vite.config.ts" >}}*
{{< highlight typescript "hl_lines = 1 8-12" >}}
import { resolve } from 'path';
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '~bootstrap': resolve(__dirname, 'node_modules/bootstrap'),
    },
  },
});
{{< /highlight >}}

But now I have angered WebStorm
{{< figure
    src="missing_node_types.png"
    alt="WebStorm showing errors for the __dirname variable and `path` import"
    caption="I missed something again"
    default="true" >}}

Lucky for me this is an easy fix by adding the NodeJS typings to the project:

```shell
yarn add --dev @types/node
```

### Second Test

Now if we refresh or re-run ```yarn dev``` the running application looks okay:
{{< figure
    src="second_test.png"
    alt="screen capture of the users route working in the browser"
    caption="The application appears to work"
    default="true" >}}

### Quality Assurance Sweep

Better verify what the linter sees:

```shell
➜  vite-redux-seed (main) ✗ yarn lint
yarn run v1.22.19
$ run-p lint:*
$ eslint ./src/**/**.ts*
$ stylelint "./src/**/*.scss"

/home/tgoshinski/work/lab/react/vite-redux-seed/src/@enums/AlertTypes.ts
  1:6  error  'AlertTypes' is defined but never used  no-unused-vars
  2:3  error  'Error' is defined but never used       no-unused-vars
  3:3  error  'Info' is defined but never used        no-unused-vars
  4:3  error  'Success' is defined but never used     no-unused-vars
  5:3  error  'Warning' is defined but never used     no-unused-vars

/home/tgoshinski/work/lab/react/vite-redux-seed/src/@enums/AppRoutes.ts
  1:6  error  'AppRoutes' is defined but never used      no-unused-vars
  2:3  error  'Home' is defined but never used           no-unused-vars
  3:3  error  'Users' is defined but never used          no-unused-vars
  4:3  error  'Notifications' is defined but never used  no-unused-vars
  5:3  error  'Fallthrough' is defined but never used    no-unused-vars

/home/tgoshinski/work/lab/react/vite-redux-seed/src/@enums/AsyncStates.ts
  1:6  error  'AsyncStates' is defined but never used  no-unused-vars
  2:3  error  'Idle' is defined but never used         no-unused-vars
  3:3  error  'Pending' is defined but never used      no-unused-vars
  4:3  error  'Success' is defined but never used      no-unused-vars
  5:3  error  'Fail' is defined but never used         no-unused-vars

/home/tgoshinski/work/lab/react/vite-redux-seed/src/@enums/FetchMethods.ts
  1:6  error  'FetchMethods' is defined but never used  no-unused-vars
  2:3  error  'Delete' is defined but never used        no-unused-vars
  3:3  error  'Get' is defined but never used           no-unused-vars
  4:3  error  'Patch' is defined but never used         no-unused-vars
  5:3  error  'Post' is defined but never used          no-unused-vars
  6:3  error  'Put' is defined but never used           no-unused-vars

/home/tgoshinski/work/lab/react/vite-redux-seed/src/@enums/HttpStatusCodes.ts
   2:6  error  'HttpStatusCodes' is defined but never used      no-unused-vars
   3:3  error  'Ok' is defined but never used                   no-unused-vars
   4:3  error  'Created' is defined but never used              no-unused-vars
   5:3  error  'Accepted' is defined but never used             no-unused-vars
   6:3  error  'NoContent' is defined but never used            no-unused-vars
   7:3  error  'MovedPermanently' is defined but never used     no-unused-vars
   8:3  error  'Redirect' is defined but never used             no-unused-vars
   9:3  error  'BadRequest' is defined but never used           no-unused-vars
  10:3  error  'Unauthorized' is defined but never used         no-unused-vars
  11:3  error  'Forbidden' is defined but never used            no-unused-vars
  12:3  error  'NotFound' is defined but never used             no-unused-vars
  13:3  error  'InternalServerError' is defined but never used  no-unused-vars
  14:3  error  'NotImplemented' is defined but never used       no-unused-vars
  15:3  error  'BadGateway' is defined but never used           no-unused-vars

/home/tgoshinski/work/lab/react/vite-redux-seed/src/@enums/ToastTypes.ts
  1:6  error  'ToastTypes' is defined but never used  no-unused-vars
  2:3  error  'Error' is defined but never used       no-unused-vars
  3:3  error  'Info' is defined but never used        no-unused-vars
  4:3  error  'Success' is defined but never used     no-unused-vars
  5:3  error  'Warning' is defined but never used     no-unused-vars

/home/tgoshinski/work/lab/react/vite-redux-seed/src/helpers/service.ts
  23:17  error  'RequestInit' is not defined  no-undef

✖ 41 problems (41 errors, 0 warnings)

error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
ERROR: "lint:ts" exited with 1.
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.

```

Ouch, that is a lot of errors. One thing I see that I forgot was to extend the ESLint TypeScript plugin's recommended:

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-2/.eslintrc.json"
           title=".eslintrc.json" >}}*
{{< highlight json "hl_lines = 18" >}}
{
  "root": true,
  "env": {
    "browser": true,
    "node": true
  },
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": "latest",
    "sourceType": "module"
  },
  "plugins": ["@typescript-eslint", "react", "react-hooks", "jsx-a11y"],
  "extends": [
    "eslint:recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    "plugin:jsx-a11y/recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier"
  ],
  "rules": {
    "no-magic-numbers": [2, { "ignore": [-1, 0, 1, 2, 10, 100, 1000] }],
    "no-unused-vars": [2, { "vars": "local", "args": "after-used", "argsIgnorePattern": "_" }],
    "no-console": [
      "error",
      {
        "allow": ["error", "info", "warn"]
      }
    ]
  },
  "settings": {
    "react": {
      "version": "detect"
    }
  }
}
{{< /highlight >}}

Hooray - more errors, not less:

```shell
/home/tgoshinski/work/lab/react/vite-redux-seed/src/@interfaces/IApiBaseResponse.ts
  7:10  warning  Unexpected any. Specify a different type  @typescript-eslint/no-explicit-any

/home/tgoshinski/work/lab/react/vite-redux-seed/src/helpers/service.ts
  74:66  warning  Unexpected any. Specify a different type  @typescript-eslint/no-explicit-any

✖ 42 problems (40 errors, 2 warnings)

```

Personally I choose to turn off `no-explicit-any` since I do find them useful in very limited circumstances.
<abbr title="Your Mileage May Vary">YMMV</abbr> but I find that extraneous abuse of `any` is best called out in code
reviews depending on the individual team norms.  Next it seems that ESLint's general `no-unused-vars` is overriding
the TypeScript plugin's `no-unused-vars` which is causing all of my enumerations to fail. We should be able to fix these
with a couple of simple adjustments to our **rules** section:


file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-2/.eslintrc.json"
           title=".eslintrc.json" >}}*
{{< highlight json "hl_lines = 23 30-34" >}}
{
  "root": true,
  "env": {
    "browser": true,
    "node": true
  },
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": "latest",
    "sourceType": "module"
  },
  "plugins": ["@typescript-eslint", "react", "react-hooks", "jsx-a11y"],
  "extends": [
    "eslint:recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    "plugin:jsx-a11y/recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier"
  ],
  "rules": {
    "no-magic-numbers": [2, { "ignore": [-1, 0, 1, 2, 10, 100, 1000] }],
    "no-unused-vars": "off",
    "no-console": [
      "error",
      {
        "allow": ["error", "info", "warn"]
      }
    ],
    "@typescript-eslint/no-explicit-any": "off",
    "@typescript-eslint/no-unused-vars": [
      2,
      { "vars": "local", "args": "after-used", "argsIgnorePattern": "_" }
    ]
  },
  "settings": {
    "react": {
      "version": "detect"
    }
  }
}
{{< /highlight >}}

One more time:

```shell
➜  vite-redux-seed (main) ✗ yarn lint
yarn run v1.22.19
$ run-p lint:*
$ stylelint "./src/**/*.scss"
$ eslint ./src/**/**.ts*
Done in 2.79s.
```

All better.

[**>> Fourth Commit**][s2]

## End of Part One

Getting the application to run was not too difficult. It mostly involved reading a little documentation around Vite and
around the ESLint settings that CRA hides with its custom plugins. Admittedly I am new to Vite so would appreciate any
advice on something you think I could be doing better.

[van]: https://github.com/code-chimp/vanilla-redux-template 'My base template for new Redux projects'
[prj]: https://github.com/code-chimp/vite-redux-seed 'My new base template for new Redux projects'
[s0]: https://github.com/code-chimp/vite-redux-seed/tree/stage-0 'Fresh Vite project'
[s1]: https://github.com/code-chimp/vite-redux-seed/tree/stage-1 'Static Analysis: eslint, stylelint, and prettier config'
[s1_5]: https://github.com/code-chimp/vite-redux-seed/tree/stage-1.5 'Static Analysis: husky pre-commit hook'
[s2]: https://github.com/code-chimp/vite-redux-seed/tree/stage-2 'Working application'
[tmp]: https://github.com/vitejs/vite/tree/main/packages/create-vite 'Vite project templates'
[vit]: https://vitejs.dev 'Next Generation Frontend Tooling'
[cra]: https://create-react-app.dev 'Set up a modern web app by running one command'
[esb]: https://esbuild.github.io/ 'extremely fast JavaScript bundler'
[rol]: https://rollupjs.org/guide/en/ 'JavaScript module bundler'
[bab]: https://babeljs.io/ 'The JavaScript Compiler'
[webp]: https://webpack.js.org/ 'Static module bundler'
[tsyn]: https://github.com/jeffijoe/typesync#readme 'Automatically find missing TypeScript typings for your project dependencies'
[hsk]: https://typicode.github.io/husky/#/ 'Modern native git hooks made easy'
[bserr]: https://getbootstrap.com/docs/5.2/getting-started/vite/ 'Bootstrap & Vite'
[a11y]: https://github.com/jsx-eslint/eslint-plugin-jsx-a11y 'static evaluation of the JSX to spot accessibility issues in React apps'
[ghk]: https://githooks.com/ 'scripts that Git executes before or after events such as: commit, push, and receive'
[pq]: https://github.com/azz/pretty-quick#readme 'Apply Prettier settings to your changed files'
[dotcra]: https://create-react-app.dev/docs/adding-custom-environment-variables/#adding-development-environment-variables-in-env 'Craete React App: Adding Development Environment Variables'
[dotvit]: https://vitejs.dev/guide/env-and-mode.html#env-variables 'Vite: Env Variables and Modes'
[dotsens]: https://vitejs.dev/guide/env-and-mode.html#intellisense-for-typescript 'Typing .env variables'
[tpack]: https://turbo.build/ 'an incremental bundler and build system optimized for JavaScript and TypeScript, written in Rust'

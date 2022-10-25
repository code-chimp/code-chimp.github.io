---
title: "Porting a Create React App application to Vite"
description: The pros, cons, and methodology for porting a Create React App application to Vite
date: 2022-10-28T00:01:01-05:00
draft: true

categories:
- Project Architecture

tags:
- configuration
- NodeJS
---

## Why?

ESBuild speed vs Webpack+Babel

## Groundwork

### Base Setup

Let's by scaffolding [the new project][prj] from Vite's minimal [React + TypeScript][tmp] template:

```shell
yarn create vite vite-redux-seed --template react-ts

cd vite-redux-seed

yarn
```

### Static Code Analysis

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
      href="https://github.com/code-chimp/vite-redux-seed/blob/main/.browserslistrc"
      title=".browserslistrc" >}}: browsers we are promising to support
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/main/.editorconfig"
      title=".editorconfig" >}}: basic text editor settings such as default line endings
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/main/.eslintignore"
      title=".eslintignore" >}}: files/folders we do not want ESLint to analyze
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/main/.prettierrc"
      title=".prettierrc" >}}: source code format rules
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/main/.prettierignore"
      title=".prettierignore" >}}: files/folders Prettier should ignore
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/main/.stylelintrc.json"
      title=".stylelintrc.json" >}}: stylesheet format rules

Create React App sets many defaults for ESLint that we will have to explicitly mimic ourselves in a new custom ESLint
config file:

file: *{{< newtabref
href="https://github.com/code-chimp/vite-redux-seed/blob/main/.eslintrc.json"
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
           href="https://github.com/code-chimp/vite-redux-seed/blob/main/package.json"
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

## The Port

Let's first make sure that the project will run, by replacing the contents of the `src` folder with everything
from [the vanilla CRA project][prj] - excluding tests and snapshots for now. We will need to rename `src/index.tsx`,
Create React App's default root component, to `src/main.tsx` which is Vite's default. Before trying to run it we will
also need to install all of our missing packages:

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

2. .env changes
3. stupid `RequestInit`
4. bootstrap config
5. ???
6. profit

[van]: https://github.com/code-chimp/vanilla-redux-template 'My base template for new Redux projects'
[prj]: https://github.com/code-chimp/vite-redux-seed 'My new base template for new Redux projects'
[s0]: https://github.com/code-chimp/vite-redux-seed/tree/stage-0 'Fresh Vite project'
[s1]: https://github.com/code-chimp/vite-redux-seed/tree/stage-1 'Static Analysis: eslint, stylelint, and prettier config'
[s1_5]: https://github.com/code-chimp/vite-redux-seed/tree/stage-1.5 'Static Analysis: husky pre-commit hook'
[tmp]: https://github.com/vitejs/vite/tree/main/packages/create-vite 'Vite project templates'
[esb]: https://esbuild.github.io/ 'extremely fast JavaScript bundler'
[tsyn]: https://github.com/jeffijoe/typesync#readme 'Automatically find missing TypeScript typings for your project dependencies'
[hsk]: https://typicode.github.io/husky/#/ 'Modern native git hooks made easy'
[bserr]: https://getbootstrap.com/docs/5.2/getting-started/vite/ 'Bootstrap & Vite'
[a11y]: https://github.com/jsx-eslint/eslint-plugin-jsx-a11y 'static evaluation of the JSX to spot accessibility issues in React apps'
[ghk]: https://githooks.com/ 'scripts that Git executes before or after events such as: commit, push, and receive'
[pq]: https://github.com/azz/pretty-quick#readme 'Apply Prettier settings to your changed files'

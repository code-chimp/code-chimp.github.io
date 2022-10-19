---
title: "PSA: Cleaning Up package.json"
description: Explaining a small set of modifications I make to achieve a more readable `package.json` file.
date: 2022-10-07T21:22:28-05:00
draft: false

categories:
- Development
- Project Architecture

tags:
- configuration
- NodeJS
---

I have a minor peeve, maybe it's just me, but I really dislike random chunks of configuration cluttering up my **package.json**
file. Project generators offered by the likes of [Nest][nest] and [Create React App][cra] still leverage the classic pattern
of embedding third party configuration values in the **package.json**, which makes it feel cluttered to me. Really I am
just looking to see the dependencies, development dependencies, NPM scripts, and basic project metadata in that file.

To illustrate I have just spun up a fresh React project with [Create React App][cra] and found a configuration section for
[ESLint][esl] and one for [Browserslist][browl]:

{{< highlight json "hl_lines=11-28" >}}
{
  "name": "whats-new",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "eslintConfig": {
    "extends": [
      "react-app",
      "react-app/jest"
    ]
  },
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  },
  "dependencies": {
    "@testing-library/jest-dom": "^5.14.1",
    "@testing-library/react": "^13.0.0",
    "@testing-library/user-event": "^13.2.1",
    "@types/jest": "^27.0.1",
    "@types/node": "^16.7.13",
    "@types/react": "^18.0.0",
    "@types/react-dom": "^18.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-scripts": "5.0.1",
    "typescript": "^4.4.2",
    "web-vitals": "^2.1.0"
  }
}
{{< /highlight >}}

Most are aware by now that the ESLint configuration can be split out into its [own configuration file][eslc] like so:

*.eslintrc.json*
{{< highlight json >}}
{
  "extends": [
    "react-app",
    "react-app/jest"
  ]
}
{{< /highlight >}}

which, to be fair, does not look like it buys you much. However this is completely barebones before you decided to wire
in your testing framework, ensure accessibility, define project coding standards, etc.

{{< figure src="one_hour_later.png" alt="Spongebob: One Hour Later" >}}

All of the new config values in this **.eslintrc.json** would have been a lot of excess content weighing the **package.json**
down distracting from the content you are actually looking to see there.
{{< highlight json >}}
{
  "root": true,
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": "latest",
    "sourceType": "module"
  },
  "env": {
    "browser": true,
    "es2021": true,
    "jest/globals": true,
    "node": true
  },
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/eslint-recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:prettier/recommended",
    "plugin:react/recommended",
    "plugin:jest/recommended",
    "plugin:jsx-a11y/recommended"
  ],
  "overrides": [
    {
      "files": ["*.ts", "*.tsx"],
      "rules": {
        "@typescript-eslint/no-unused-vars": [2, { "args": "none" }]
      }
    }
  ],
  "plugins": ["@typescript-eslint", "react", "react-hooks", "jest", "jsx-a11y"],
  "rules": {
    "react/self-closing-comp": [
      "error",
      {
        "component": true,
        "html": true
      }
    ],
    "react/no-array-index-key": 2,
    "react/no-danger": 1,
    "react/no-deprecated": 2,
    "react/no-did-mount-set-state": 1,
    "react/no-did-update-set-state": 1,
    "react/no-direct-mutation-state": 2,
    "react/no-find-dom-node": 1,
    "react/no-is-mounted": 1,
    "react/no-multi-comp": 2,
    "react/no-redundant-should-component-update": 2,
    "react/no-render-return-value": 2,
    "react/no-typos": 1,
    "react/react-in-jsx-scope": 1,
    "react/jsx-handler-names": "off",
    "react/jsx-no-duplicate-props": 2,
    "react/jsx-fragments": 2,
    "react/jsx-pascal-case": 2,
    "react/jsx-boolean-value": 2,
    "no-unused-vars": [2, { "vars": "local", "args": "after-used", "argsIgnorePattern": "_" }],
    "no-magic-numbers": [2, { "ignore": [-1, 0, 1, 2, 10, 100, 3000, 3001] }],
    "react-hooks/exhaustive-deps": "warn",
    "react-hooks/rules-of-hooks": "error",
    "no-prototype-builtins": "off",
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

If you were not aware the Browserslist settings can also be broken out into a separate file

*.browserslistrc*
{{< highlight toml >}}
[production]
> 0.25%
not dead
not op_mini all

[development]
last 1 chrome version
last 1 firefox version
last 1 safari version
{{< /highlight >}}

Likewise [Nest][nest] tends to put [Jest][jest] configuration in the **package.json** while it can just as easily go
into its own JavaScript file. **Note:** As far as I can tell [Create React App][cra] powered projects **will not**
recognize Jest settings outside of the **package.json**, but here is a working example from one of my NodeJS projects:

*jest.config.js*
{{< highlight javascript >}}
module.exports = {
  testRegex: '.*\\.test\\.tsx?$',
  transform: {
    '^.+\\.(t|j)sx?$': 'ts-jest',
  },
  setupFilesAfterEnv: ['./setupTests.ts'],
  collectCoverage: true,
  collectCoverageFrom: [
    '**/*.{js,ts,tsx}',
    '!src/api/**',
    '!coverage/**',
    '!data-scripts/**',
    '!node_modules/**',
    '!**/@enums/**',
    '!**/@interfaces/**',
    '!**/@mocks/**',
    '!**/@types/**',
    '!**/**/index.ts',
    '!**/**.d.ts',
    '!src/client/index.tsx',
    '!src/client/services/**',
    '!scripts/average-work.ts',
    '!jest.config.js',
    '!setupTests.ts',
  ],
  coverageThreshold: {
    global: {
      branches: 85,
      functions: 95,
      statements: 85,
    },
  },
};
{{< /highlight >}}

So any time I find a new section of key-value pairs that I do not think belong in the **package.json** I consult the
documentation to see if an option exists to move it out into a separate file. For me, personally, having my project
configuration more focused into digestible chunks is worth the minor expense of a few extra files in my project root.

{{< figure src="project_tree.png" alt="project root" caption="a fairly average project root" >}}

[nest]: https://nestjs.com/ 'progressive Node.js framework for server-side applications written in a familiar Angular-like syntax'
[cra]: https://create-react-app.dev/ 'Set up a modern web app by running one command'
[esl]: https://eslint.org/ 'A pluggable linting utility for early identification of problems in your code'
[browl]: https://github.com/browserslist/browserslist 'config to share target browsers and Node.js versions between different front-end tools'
[eslc]: https://eslint.org/docs/latest/user-guide/configuring/configuration-files 'ESLint configuration file formats'
[jest]: https://jestjs.io/ 'JavaScript test framework'


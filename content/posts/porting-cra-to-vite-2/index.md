---
title: "Porting a Create React App application to Vite Pt. 2: Unit Testing"
date: 2022-11-02T00:01:01-05:00
draft: false

categories:
- Project Architecture
- New Tech

tags:
- configuration
- testing
- NodeJS
---

For unit tests we will still be using the excellent [Testing Library][tstl], but we will swap [Vitest][vtst] in place of
[Jest][jest]. Vitest promises Jest compatibility without having to duplicate a bunch of configuration to get Jest to
function correctly with a Vite project.

## Configure Testing

Let's install the packages that we will need for creating unit tests:

```shell
yarn add --dev vitest @vitest/coverage-istanbul jsdom @testing-library/jest-dom \
@testing-library/react @testing-library/react-hooks @testing-library/user-event \
redux-mock-store

# add missing typings to package.json
npx typesync

# install the typings found by typesync
yarn
```

We can just lift
{{< newtabref
    href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/setupTests.ts"
    title="/src/setupTests.ts" >}} as-is from [our previous project][van].


Since I like to explicitly separate configurations based on usage I chose [the third option from the documentation][config]
for creating a `vitest.config.ts` file. The `coverage.include` and `coverage.exclude` values are translated from the
`jest.collectCoverageFrom` section of my <abbr title="Create React App">CRA</abbr> project's
{{< newtabref
    href="https://github.com/code-chimp/vanilla-redux-template/blob/main/package.json"
    title="package.json" >}}. `coverage.branches/function/statements` were pulled from `jest.coverageThreshold.global`
values in the same file.



file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/vitest.config.ts"
           title="vitest.config.ts" >}}*
{{< highlight typescript >}}
import { mergeConfig } from 'vite';
import { defineConfig } from 'vitest/config';
import viteConfig from './vite.config';

export default mergeConfig(
  viteConfig,
  defineConfig({
    test: {
      environment: 'jsdom',
      // we do not want to have to import 'expect', etc. on every file
      globals: true,
      setupFiles: './src/setupTests.ts',
      coverage: {
        provider: 'istanbul',
        // if 'false' does not show our uncovered files
        all: true,
        include: ['src/**/*.{ts,tsx}'],
        exclude: [
          'src/@enums/**',
          'src/@interfaces/**',
          'src/@mocks/**',
          'src/@types/**',
          'src/services/**',
          'src/**/index.{ts,tsx}',
          'src/main.tsx',
          'src/routes.tsx',
        ],
        branches: 85,
        functions: 90,
        statements: 90,
      },
    },
  }),
);
{{< /highlight >}}

The TypeScript compiler will need to know of Vitest's globals:

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/tsconfig.json"
           title="tsconfig.json" >}}*
{{< highlight json "hl_lines=18" >}}
{
  "compilerOptions": {
    "target": "ESNext",
    "useDefineForClassFields": true,
    "lib": ["DOM", "DOM.Iterable", "ESNext"],
    "allowJs": false,
    "skipLibCheck": true,
    "esModuleInterop": false,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "ESNext",
    "moduleResolution": "Node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "types": ["vitest/globals"]
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
{{< /highlight >}}

Define a couple of NPM scripts for convenience:

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/package.json"
           title="package.json" >}}*
{{< highlight json "hl_lines=12-13" >}}
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
    "test": "vitest --run",
    "test:cov": "vitest run --coverage",
    "prepare": "husky install"
  },
{{< /highlight >}}

Finally add our coverage thresholds to the pre-commit hook:

```shell
npx husky add .husky/pre-commit "npm run test:cov"
```
## Port a Simple Test

For the first test I want to bring over something really simple so I believe my service helpers will be a good place to
start. We should be able to get away with only modifying the one line dealing with pulling a value from the `.env` file:

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/helpers/service.test.ts"
           title="/src/helpers/service.test.ts" >}}*
{{< highlight typescript "hl_lines=16" >}}
import {
  createDeleteRequest,
  createGetRequest,
  createPatchRequest,
  createPostRequest,
  createPutRequest,
  processApiResponse,
  unwrapServiceError,
} from './service';
import FetchMethods from '../@enums/FetchMethods';

import HttpStatusCodes from '../@enums/HttpStatusCodes';
import IApiBaseResponse from '../@interfaces/IApiBaseResponse';
import { GENERIC_SERVICE_ERROR } from '../constants';

const adminApiUri = import.meta.env.VITE_API_URI;
const fakeEndpoint = '/api/22.19/bubbas/burger/barn';

describe('helpers / service', () => {
  describe('createGetRequest', () => {
    it('should generate a Request with a `GET` method', () => {
      const result: Request = createGetRequest(fakeEndpoint);

      expect(result.method).toBe(FetchMethods.Get);
      // there will be an underscore (_) query param appended to the url since we
      //   have cache set to false for all requests
      expect(result.url.startsWith(`${adminApiUri}${fakeEndpoint}`)).toBe(true);
      expect(result.headers.has('Authorization')).toBe(false);
      expect(result.headers.has('Content-Type')).toBe(false);
    });

    ...

    it('should return a standard Error from a custom thrown error', () => {
      const sut = {
        message: 'yeah, no',
        errors: ['you should see me', 'but not me'],
      };
      const methodName = 'unwrapped';

      const serviceError: any = unwrapServiceError(methodName, sut);

      expect(serviceError.message).toBe(`${methodName} error: ${sut.errors[0]}`);
      expect(serviceError.message).not.toBe(`${methodName} error: ${sut.errors[1]}`);
    });
  });
});
{{< /highlight >}}

Looks like a success:

```shell
➜  vite-redux-seed (main) ✗ yarn test
 RUN  v0.24.4 /home/tgoshinski/work/lab/react/vite-redux-seed

 ✓ src/helpers/service.test.ts (15)

Test Files  1 passed (1)
     Tests  15 passed (15)
  Start at  22:24:59
  Duration  2.26s (transform 681ms, setup 217ms, collect 99ms, tests 31ms)


Process finished with exit code 0.
```

## Port Redux Slice Tests

May as well start alphabetically with the **alerts** slice tests. Mocking with Vitest is very similar to mocking in Jest,
so there were only a few modifications that needed to be made to get this test working. First we need to import the
**Mock** interface from Vitest and replace all `jest.Mock<any, any>` with just `Mock<any, any>`. Next `jest.mock('uuid')`
becomes `vi.mock('uuid')`:

file: *{{< newtabref
           href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/store/slices/alerts.test.ts"
           title="/src/store/slices/alerts.test.ts" >}}*
{{< highlight typescript "hl_lines=1 7 20" >}}
import { Mock } from 'vitest';
import { alerts } from './alerts';
import AlertTypes from '../../@enums/AlertTypes';
import IAlert from '../../@interfaces/IAlert';

import { v4 } from 'uuid';
vi.mock('uuid');

describe('store / slices / alerts', () => {
  describe('reducer(s)', () => {
    const initialState: Array<IAlert> = [
      { id: 'foo', type: AlertTypes.Error, text: 'i broke it' },
      { id: 'bar', type: AlertTypes.Info, text: 'it was brokded when I got here' },
      {
        id: 'baz',
        type: AlertTypes.Warning,
        text: "keep your fingers away from Lenny's mouth",
      },
    ];
    const mockedUuid = v4 as Mock<any, any>;
    const mockUuidValue = 'some-unique-guidy-thing';

    it('should remove an alert by id', () => {
      const { removeAlert } = alerts.actions;
      const alertId = 'bar';
      const expected: Array<IAlert> = initialState.filter(a => a.id !== alertId);

      const state = alerts.reducer(initialState, removeAlert(alertId));

      expect(state.some(_ => _.id === alertId)).toBe(false);
      expect(state).toEqual(expected);
    });

    ...

    it('should add a warning alert with a generated uuid', () => {
      mockedUuid.mockImplementationOnce(() => mockUuidValue);
      const { addWarningAlert } = alerts.actions;

      const payload: IAlert = {
        id: 'really-why-do-you-read-these',
        type: AlertTypes.Warning,
        text: 'do not put that in your eye',
      };
      const expected: Array<IAlert> = [...initialState, { ...payload, id: mockUuidValue }];

      const state = alerts.reducer(initialState, addWarningAlert(payload));

      expect(state).toEqual(expected);
    });
  });
});
{{< /highlight >}}

Run the slice tests:

```shell
➜  vite-redux-seed (main) ✗ yarn test
 RUN  v0.24.4 /home/tgoshinski/work/lab/react/vite-redux-seed

 ✓ src/store/slices/alerts.test.ts (5)

Test Files  1 passed (1)
     Tests  5 passed (5)
  Start at  22:47:57
  Duration  2.24s (transform 736ms, setup 195ms, collect 163ms, tests 7ms)


Process finished with exit code 0.
```

Now we can bring over the rest of the store slice tests:

- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/store/slices/counter.test.ts"
      title="/src/store/slices/counter.test.ts" >}}: required no modification
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/store/slices/toasts.test.ts"
      title="/src/store/slices/toasts.test.ts" >}}: required the same modifications as **alerts**
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/store/slices/user.test.ts"
      title="/src/store/slices/user.test.ts" >}}: required the same modifications as **alerts**, also remember to copy
this `/src/@mocks` folder from [the CRA project][van]

## Port Component Tests

The good news is I did not need to modify any of my component tests - they all just worked as originally written for Jest:

- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/components/app/AppAlerts/AppAlerts.test.tsx"
      title="/src/components/app/AppAlerts/AppAlerts.test.tsx" >}}
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/components/app/AppAlerts/Alert/Alert.test.tsx"
      title="/src/components/app/AppAlerts/Alert/Alert.test.tsx" >}}
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/components/app/AppToasts/AppToasts.test.tsx"
      title="/src/components/app/AppToasts/AppToasts.test.tsx" >}}
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/components/app/AppToasts/Toast/Toast.test.tsx"
      title="/src/components/app/AppToasts/Toast/Toast.test.tsx" >}}
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/pages/Counter/Counter.test.tsx"
      title="/src/pages/Counter/Counter.test.tsx" >}}
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/pages/Users/Users.test.tsx"
      title="/src/pages/Users/Users.test.tsx" >}}

## Final Sweep

Putting it all together let's commit all of our hard work:

```shell
➜  vite-redux-seed (main) ✗ git add -A
➜  vite-redux-seed (main) ✗ git commit -m ':white_check_mark: unit tests'

> vite-redux-seed@0.1.0 format
> pretty-quick --staged

🔍  Finding changed files since git revision 6560904.
🎯  Found 17 changed files.
✅  Everything is awesome!


 RUN  v0.24.4 /home/tgoshinski/work/lab/react/vite-redux-seed
      Coverage enabled with istanbul

 ✓ src/helpers/service.test.ts (15)
 ✓ src/components/app/AppToasts/Toast/Toast.test.tsx (7) 546ms
 ✓ src/helpers/service.test.ts (15)
 ✓ src/components/app/AppToasts/Toast/Toast.test.tsx (7) 546ms
 ✓ src/components/app/AppToasts/AppToasts.test.tsx (1) 325ms
 ✓ src/pages/Counter/Counter.test.tsx (4) 306ms
 ✓ src/components/app/AppAlerts/Alert/Alert.test.tsx (9) 376ms
 ✓ src/components/app/AppAlerts/AppAlerts.test.tsx (1)
 ✓ src/pages/Users/Users.test.tsx (5) 315ms
 ✓ src/store/slices/user.test.ts (8)
 ✓ src/store/slices/toasts.test.ts (5)
 ✓ src/store/slices/alerts.test.ts (5)
 ✓ src/store/slices/counter.test.ts (3)

Test Files  11 passed (11)
     Tests  63 passed (63)
  Start at  08:42:58
  Duration  11.68s (transform 1.56s, setup 2.98s, collect 6.56s, tests 2.25s)

 % Coverage report from istanbul
------------------------------------|---------|----------|---------|---------|-------------------
File                                | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
------------------------------------|---------|----------|---------|---------|-------------------
All files                           |   85.99 |    83.67 |   79.72 |   86.66 |
 src                                |     100 |    93.33 |     100 |     100 |
  helpers                           |     100 |    93.33 |     100 |     100 | 53
 src/components/app/AppAlerts       |     100 |      100 |     100 |     100 |
  AppAlerts.tsx                     |     100 |      100 |     100 |     100 |
 src/components/app/AppAlerts/Alert |   95.23 |      100 |      75 |   95.23 |
  Alert.tsx                         |   95.23 |      100 |      75 |   95.23 | 28
 src/components/app/AppToasts       |     100 |      100 |     100 |     100 |
  AppToasts.tsx                     |     100 |      100 |     100 |     100 |
 src/components/app/AppToasts/Toast |   96.66 |      100 |      75 |   96.66 |
  Toast.tsx                         |   96.66 |      100 |      75 |   96.66 | 31
 src/components/app/Navigation      |       0 |      100 |       0 |       0 |
  Navigation.tsx                    |       0 |      100 |       0 |       0 | 7-8
 src/helpers                        |     100 |      100 |     100 |     100 |
  hooks.ts                          |     100 |      100 |     100 |     100 |
 src/layouts/MainLayout             |       0 |      100 |       0 |       0 |
  MainLayout.tsx                    |       0 |      100 |       0 |       0 | 10-11
 src/pages/Counter                  |     100 |      100 |     100 |     100 |
  Counter.tsx                       |     100 |      100 |     100 |     100 |
 src/pages/FourOhFour               |       0 |        0 |       0 |       0 |
  FourOhFour.tsx                    |       0 |        0 |       0 |       0 | 6-9
 src/pages/NotificationsDemo        |       0 |      100 |       0 |       0 |
  NotificationsDemo.tsx             |       0 |      100 |       0 |       0 | 18-45
 src/pages/Users                    |     100 |    76.92 |     100 |     100 |
  Users.tsx                         |     100 |    76.92 |     100 |     100 | 44-46
 src/store/slices                   |   98.61 |       75 |   97.36 |     100 |
  alerts.ts                         |     100 |      100 |     100 |     100 |
  counter.ts                        |     100 |      100 |     100 |     100 |
  toasts.ts                         |     100 |      100 |     100 |     100 |
  user.ts                           |   96.42 |    66.66 |      90 |     100 | 28,61
------------------------------------|---------|----------|---------|---------|-------------------
ERROR: Coverage for functions (79.72%) does not meet global threshold (90%)
ERROR: Coverage for statements (85.99%) does not meet global threshold (90%)
ERROR: Coverage for branches (83.67%) does not meet global threshold (85%)
husky - pre-commit hook exited with code 1 (error)
```

### ESBuild Problem and Workaround

We should have met our coverage thresholds since we have pulled all of the same tests from the previous [CRA project][van],
but on the bright side we have confirmed that our pre-commit hook works.

It appears our istanbul directives (ex: __/* istanbul ignore file */__), for files like `NotificationsDemo.tsx` are not
being honored. Searching the issues tracker on Vitest's Github I found [the issue][issue] appears to be ESBuild is
stripping those directive comments out when transpiling the file. The workaround that seemed the least invasive to me
was explicitly telling ESBuild to preserve those comments by changing `/* istanbul ignore file */` to
`/* istanbul ignore file -- @preserve */`.

See files:

- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/components/app/Navigation/Navigation.tsx"
      title="/src/components/app/Navigation/Navigation.tsx" >}}
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3.1/src/helpers/hooks.ts"
      title="/src/helpers/hooks.ts" >}}
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/layouts/MainLayout/MainLayout.tsx"
      title="/src/layouts/MainLayout/MainLayout.tsx" >}}
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/pages/FourOhFour/FourOhFour.tsx"
      title="/src/pages/FourOhFour/FourOhFour.tsx" >}}
- {{< newtabref
      href="https://github.com/code-chimp/vite-redux-seed/blob/stage-3/src/pages/NotificationsDemo/NotificationsDemo.tsx"
      title="/src/pages/NotificationsDemo/NotificationsDemo.tsx" >}}


With those files being excluded from coverage we should now be well within our coverage thresholds and able to commit
our files:

```shell
➜  vite-redux-seed (main) ✗ git add -A
➜  vite-redux-seed (main) ✗ git commit -m ':white_check_mark: unit tests'

> vite-redux-seed@0.1.0 format
> pretty-quick --staged

🔍  Finding changed files since git revision 6560904.
🎯  Found 21 changed files.
✅  Everything is awesome!


 RUN  v0.24.4 /home/tgoshinski/work/lab/react/vite-redux-seed
      Coverage enabled with istanbul

 ✓ src/components/app/AppToasts/Toast/Toast.test.tsx (7) 477ms
 ✓ src/components/app/AppAlerts/Alert/Alert.test.tsx (9)
 ✓ src/components/app/AppToasts/Toast/Toast.test.tsx (7) 477ms
 ✓ src/components/app/AppAlerts/Alert/Alert.test.tsx (9)
 ✓ src/components/app/AppToasts/AppToasts.test.tsx (1) 352ms
 ✓ src/pages/Users/Users.test.tsx (5)
 ✓ src/pages/Counter/Counter.test.tsx (4)
 ✓ src/components/app/AppAlerts/AppAlerts.test.tsx (1)
 ✓ src/helpers/service.test.ts (15)
 ✓ src/store/slices/user.test.ts (8)
 ✓ src/store/slices/toasts.test.ts (5)
 ✓ src/store/slices/alerts.test.ts (5)
 ✓ src/store/slices/counter.test.ts (3)

Test Files  11 passed (11)
     Tests  63 passed (63)
  Start at  09:38:03
  Duration  10.97s (transform 1.82s, setup 3.09s, collect 6.26s, tests 2.06s)

 % Coverage report from istanbul
------------------------------------|---------|----------|---------|---------|-------------------
File                                | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
------------------------------------|---------|----------|---------|---------|-------------------
All files                           |   98.34 |    87.23 |   95.16 |   98.83 |
 src                                |     100 |    93.33 |     100 |     100 |
  helpers                           |     100 |    93.33 |     100 |     100 | 53
 src/components/app/AppAlerts       |     100 |      100 |     100 |     100 |
  AppAlerts.tsx                     |     100 |      100 |     100 |     100 |
 src/components/app/AppAlerts/Alert |   95.23 |      100 |      75 |   95.23 |
  Alert.tsx                         |   95.23 |      100 |      75 |   95.23 | 28
 src/components/app/AppToasts       |     100 |      100 |     100 |     100 |
  AppToasts.tsx                     |     100 |      100 |     100 |     100 |
 src/components/app/AppToasts/Toast |   96.66 |      100 |      75 |   96.66 |
  Toast.tsx                         |   96.66 |      100 |      75 |   96.66 | 31
 src/helpers                        |     100 |      100 |     100 |     100 |
  hooks.ts                          |     100 |      100 |     100 |     100 |
 src/pages/Counter                  |     100 |      100 |     100 |     100 |
  Counter.tsx                       |     100 |      100 |     100 |     100 |
 src/pages/Users                    |     100 |    76.92 |     100 |     100 |
  Users.tsx                         |     100 |    76.92 |     100 |     100 | 44-46
 src/store/slices                   |   98.61 |       75 |   97.36 |     100 |
  alerts.ts                         |     100 |      100 |     100 |     100 |
  counter.ts                        |     100 |      100 |     100 |     100 |
  toasts.ts                         |     100 |      100 |     100 |     100 |
  user.ts                           |   96.42 |    66.66 |      90 |     100 | 28,61
------------------------------------|---------|----------|---------|---------|-------------------
[main 2da7f0b] :white_check_mark: unit tests
 29 files changed, 2902 insertions(+), 34 deletions(-)
 create mode 100644 src/@mocks/users.ts
 create mode 100644 src/components/app/AppAlerts/Alert/Alert.test.tsx
 create mode 100644 src/components/app/AppAlerts/Alert/__snapshots__/Alert.test.tsx.snap
 create mode 100644 src/components/app/AppAlerts/AppAlerts.test.tsx
 create mode 100644 src/components/app/AppAlerts/__snapshots__/AppAlerts.test.tsx.snap
 create mode 100644 src/components/app/AppToasts/AppToasts.test.tsx
 create mode 100644 src/components/app/AppToasts/Toast/Toast.test.tsx
 create mode 100644 src/components/app/AppToasts/Toast/__snapshots__/Toast.test.tsx.snap
 create mode 100644 src/components/app/AppToasts/__snapshots__/AppToasts.test.tsx.snap
 create mode 100644 src/helpers/service.test.ts
 create mode 100644 src/pages/Counter/Counter.test.tsx
 create mode 100644 src/pages/Counter/__snapshots__/Counter.test.tsx.snap
 create mode 100644 src/pages/Users/Users.test.tsx
 create mode 100644 src/pages/Users/__snapshots__/Users.test.tsx.snap
 create mode 100644 src/setupTests.ts
 create mode 100644 src/store/slices/alerts.test.ts
 create mode 100644 src/store/slices/counter.test.ts
 create mode 100644 src/store/slices/toasts.test.ts
 create mode 100644 src/store/slices/user.test.ts
 create mode 100644 vitest.config.ts
```

## Final Thoughts

Porting a small demo application was not really difficult and I am happy with the results. I would actually have to
tackle a real-world project to determine if Vite is a better alternative for me than Create React App but at least I have
determined that I will not have to sacrifice any of the code quality tooling I have been relying on.

If I had to choose something to complain about it would be that Vitest's integration with WebStorm is not as seamless as
Jest's. Jest's test runner's coverage analysis integration is second to none and I really miss that with the [Vitest test
runner][run]:

{{< figure
    src="jest_coverage.png"
    alt="displaying WebStorm's `Coverage` window"
    caption="Note the subtle gutter indicators for covered lines in **alerts.ts**"
    default="true" >}}

The debugger appears to work as well with Vitest as Jest though, so that is a plus:

{{< figure
    src="test_debugger.png"
    alt="displaying WebStorm's `Debug` window stepping through a Vitest test"
    caption="Very similar to debugging tests written with Jest"
    default="true" >}}

If Vite continues to gain traction I am sure WebStorm's tooling will catch up just as it did when newer tech like
Tailwind CSS became a hit. I am not a big user of Visual Studio Code so I can not speak to plugin support on that
particular IDE. From the command line everything works just as well as using Jest so really it is just a GUI-user's
annoyance more than any kind of hindrance.

Unless Create React App is a hard requirement I will likely give Vite a chance with the next project I get to spin up
from scratch. Thank you for stopping by, I hope you found something useful here.


[vtst]: https://vitest.dev/ 'Vite-native unit test framework'
[tstl]: https://testing-library.com/ 'Simple and complete testing utilities that encourage good testing practices'
[jest]: https://jestjs.io/ 'Delightful JavaScript Testing Framework with a focus on simplicity'
[config]: https://vitest.dev/config/ 'Using the third option for creating a test configuration'
[issue]: https://github.com/vitest-dev/vitest/issues/2021 'github issue: ESBuild stripping istanbul directives'
[int]: https://vitest.dev/guide/ide.html#intellij-webstorm-community 'Vitest recommended integration'
[run]: https://plugins.jetbrains.com/plugin/19220-vitest-runner 'Vitest Runner'
[van]: https://github.com/code-chimp/vanilla-redux-template 'My base template for new Redux projects'


# TSC Build Cache Invalidation

This project is a minimal reproduction of a cache invalidation issue related to subprojects and incremental compilation in the typescript compiler as of version 5.1.3.

## Reproduction steps
1. Install project dependencies
    ```
    npm ci
    ```
1. Build the project
    ```
    npx tsc --build
    ```
1. Check out the first commit
    ```
    git checkout 53e42026f1c20bb4e7370
    ```
1. Build the project again
    ```
    npx tsc --build
    ```
1. Check out `main`
    ```
    git checkout main
    ```
1. Build the project again
    ```
    npx tsc --build
    ```

For convenience, here are all the steps in a single block:
```
npm ci
npx tsc --build
git checkout 53e4202
npx tsc --build
git checkout main
npx tsc --build
```

## Result

The final build produces the following error output:
```
build/hello/hello.d.ts:1:23 - error TS2307: Cannot find module 'main/const' or its corresponding type declarations.

1 import { LogFn } from 'main/const';
                        ~~~~~~~~~~~~


Found 1 error.
```

## Expected

No error. The error will go away if you clean tsc's output and do a rebuild:
```
npx tsc --build --clean
npx tsc --build
```

## Interpretation

The second commit in this repo ([201a7af](https://github.com/jwmerrill/tsc-build-cache-invalidation/commit/201a7af7d38bbfc22e0a3aa34ab0368ab2b6a35c)) creates a new subproject in the `hello` directory, and moves `main/const.ts` into `hello/const.ts` so that it becomes part of this subproject. This also requires updating an import in `hello/hello.ts` to import `const.ts` from the new location.

The first build on the `main` branch creates a `build/hello/tsconfig.tsbuildinfo` cache file for the `hello` subproject.

The next build, on the initial commit ([53e4202](https://github.com/jwmerrill/tsc-build-cache-invalidation/commit/53e42026f1c20bb4e7370739530b788b25f78b35)), updates `build/hello/hello.d.ts` to import from `main/const`. This commit has no awareness of the `hello` subproject, and it does not touch `build/hello/tsconfig.tsbuildinfo`.

The final build, back on `main`, does not rebuild the `hello` subproject because it sees `build/hello/tsconfig.tsbuildinfo` and sees that the newest input of the `hello` subproject is older than this cache file. You can see this by running `npx tsc --build --verbose`

Output:
```
> npx tsc --build --verbose
[12:44:45 PM] Projects in this build:
    * src/hello/tsconfig.json
    * src/main/tsconfig.json
    * tsconfig.json

[12:44:45 PM] Project 'src/hello/tsconfig.json' is up to date because newest input 'src/hello/const.ts' is older than output 'build/hello/tsconfig.tsbuildinfo'

[12:44:45 PM] Project 'src/main/tsconfig.json' is out of date because buildinfo file 'build/main/tsconfig.tsbuildinfo' indicates that some of the changes were not emitted

[12:44:45 PM] Building project '/Users/jason/src/tsc-build-cache-invalidation/src/main/tsconfig.json'...

build/hello/hello.d.ts:1:23 - error TS2307: Cannot find module 'main/const' or its corresponding type declarations.

1 import { LogFn } from 'main/const';
                        ~~~~~~~~~~~~


Found 1 error.
```

This is incorrect because one of the outputs of the `hello` project is stale, even though none of the inputs are newer than the build cache file.

## Notes

This bug is a reduced version of a bug found by [Desmos Studio](https://www.desmos.com) in the process of migrating the Desmos Graphing Calculator to TS subprojects.

Our developers found that they were having to frequently clean the TS build directory when switching back and forth between branches that were either newer than or older than the migration to subprojects.

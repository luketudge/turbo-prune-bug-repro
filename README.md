# Overview

This repo was created to provide a reproduction of a possible bug in Turborepo.

Specifically, it covers a special-case regression of a bug already reprexed in https://github.com/erj826/turbo-prune-bug-repro-11-21

## new case

This repo is based on that one, with some crucial modifications:

### yarn

Moves to yarn 4.5.1 (was 3.3.0). I am not absolutely certain this is critical, it just happens to be the version of yarn that I am using and for which I can most easily and for sure reproduce the issue.

### more complex dependency tree

An added dependency on `"@nestjs/graphql": "13.2.5"`. I am fairly sure this is a real-world example of the specific special case that is regressed. Note that this package has a double-dependency on different major versions of `ws`, one directly and one via `subscriptions-transport-ws` (before forcing any specific resolutions):

```shell
yarn why ws --recursive
```
```
└─ web@workspace:apps/web
   └─ @nestjs/graphql@npm:13.2.5 [e2932] (via npm:13.2.5 [e2932])
      ├─ subscriptions-transport-ws@npm:0.11.0 [dcc91] (via npm:0.11.0 [dcc91])
      │  └─ ws@npm:7.5.13 [7bfeb] (via npm:^5.2.0 || ^6.0.0 || ^7.0.0 [7bfeb])
      └─ ws@npm:8.20.0 [dcc91] (via npm:8.20.0 [dcc91])
```

### corresponding complex resolution

In order to force-resolve both versions of `ws` to higher (or equal) respectively compatible _minor_ versions of each major, there are two resolution paths in the monorepo project root `"resolutions"` directive, targeting the two dependency paths to `ws`:

```json
"resolutions": {
  "subscriptions-transport-ws/ws": "^7.5.13",
  "ws": "^8.21.0"
}
```

## steps to reproduce

Generate the pruned output for the target 'web' app that depends on `@nestjs/graphql`:

```shell
yarn turbo prune web
```

Verify that in the [out](out/) directory, there is a pruned lockfile that is missing one of the resolutions of `ws`. This can also be tested by attempting an immutable install, which should be possible if the lockfile is compatible with the pruned package.json:

```shell
cd out
yarn install --immutable
```
```
➤ YN0028: │ +"ws@npm:^7.5.13":
➤ YN0028: │ +  version: 7.5.13
➤ YN0028: │ +  resolution: "ws@npm:7.5.13"
➤ YN0028: │ +  peerDependencies:
➤ YN0028: │ +    bufferutil: ^4.0.1
➤ YN0028: │ +    utf-8-validate: ^5.0.2
➤ YN0028: │ +  peerDependenciesMeta:
➤ YN0028: │ +    bufferutil:
➤ YN0028: │ +      optional: true
➤ YN0028: │ +    utf-8-validate:
➤ YN0028: │ +      optional: true
➤ YN0028: │ +  languageName: node
➤ YN0028: │ +  linkType: hard
➤ YN0028: │ +
➤ YN0000: │  "ws@npm:^8.21.0":
➤ YN0000: │    version: 8.21.3
➤ YN0000: │    resolution: "ws@npm:8.21.3"
➤ YN0000: │    peerDependencies:
➤ YN0000: │
➤ YN0028: │ The lockfile would have been modified by this install, which is explicitly forbidden.
```

Rinse and repeat after switching the root package.json to varying versions of turbo, showing a regression from 2.10.4 to 2.10.5 and still present in the current canary:

* `"turbo": "2.10.13-canary.4"` X
* `"turbo": "2.10.5"` X
* `"turbo": "2.10.4"` OK

---
id: configuration
title: Configuration
type: reference
---

# Configuration

Lerna's configuration is split into two files: `lerna.json` and `nx.json`.

# Lerna.json

### npmClient

It is important to set this value if you are not using `npm` as your package manager (e.g. if you are using `yarn` or `pnpm`) so that lerna can adjust some of its internal logic when resolving configuration and packages. This is particularly true in the case of `pnpm` because it uses a separate `pnpm-workspaces.yaml` file to define its workspaces configuration.

### packages

By default, lerna will try and reuse any `workspaces` configuration you may have from your package manager of choice. If you prefer to specify a subset of your available packages for lerna to operate on, you can use the `packages` property which will tell Lerna where to look for `package.json` files.

```json title="lerna.json"
{
  "packages": ["packages/*"]
}
```

### version

Lerna has two modes of publishing packages: `fixed` and `independent`. When using the fixed mode, all the affected packages will be published using the same version. The last published version is recorded in `lerna.json` as follows:

```json title="lerna.json"
{
  "version": "1.2.0"
}
```

When using the independent mode, every package is versioned separately, and `lerna.json` will look as follows:

```json title="lerna.json"
{
  "version": "independent"
}
```

See the [version and publish docs](../features/version-and-publish.md) for more details.

### commands

The `lerna.json` files can also encode options for each command like so:

```json
{
  "command": {
    "version": {
      "allowBranch": "main",
      "conventionalCommits": true
    }
  }
}
```

Find the available options in [the API docs](/docs/api-reference/commands).

# Nx.json

> NOTE: "{projectRoot}" and "{workspaceRoot}" are special syntax supported by the task-runner, which will be appropriately interpolated internally when the command runs. You should therefore not replace "{projectRoot}" or "{workspaceRoot}" with fixed paths as this makes your configuration less flexible.

```json title="nx.json"
{
  "namedInputs": {
    "default": ["{projectRoot}/**/*"],
    "prod": ["!{projectRoot}/**/*.spec.tsx"]
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["prebuild", "^build"],
      "inputs": ["prod", "^prod"],
      "outputs": ["{projectRoot}/dist"],
      "cache": true
    },
    "test": {
      "inputs": ["default", "^prod", "{workspaceRoot}/jest.config.ts"],
      "cache": true
    }
  }
}
```

## Target Defaults

Targets are npm script names. You can add metadata associated with say the build script of each project in the repo in
the `targetDefaults` section.

### cache

When set to `true`, tells Nx to cache the results of running the script. In most repos all
non-long running tasks (i.e., not `serve`) should be cacheable.

### dependsOn

Targets can depend on other targets. A common scenario is having to build dependencies of a project first before
building the project. The `dependsOn` property can be used to define the dependencies of an individual target.

`"dependsOn": [ "prebuild", "^build"]` tells Nx that every build script requires the prebuild script of the same
project and the build script of all the dependencies to run first.

### inputs & namedInputs

The `inputs` array tells Nx what to consider to determine whether a particular invocation of a script should be a cache
hit or not. There are three types of inputs:

_Filesets_

Examples:

- `{projectRoot}/**.*.ts`
- same as `{fileset: "{projectRoot}/**/*.ts"}`
- `{workspaceRoot}/jest.config.ts`
- same as `{fileset: "{workspaceRoot}/jest.config.ts}`

_Runtime Inputs_

Examples:

- `{runtime: "node -v"}`

Node the result value is hashed, so it is never displayed.

_Env Variables_

Examples:

- `{env: "MY_ENV_VAR"}`

Node the result value is hashed, so it is never displayed.

_Named Inputs_

Examples:

- `inputs: ["prod"]`
- same as `inputs: [{input: "prod", projects: "self"}]`

Often the same glob will appear in many places (e.g., prod fileset will exclude spec files for all projects).. Because
keeping them in sync is error-prone, we recommend defining named inputs, which you can then reference in all of those
places.

#### Using ^

Examples:

- `inputs: ["^prod"]`
- same as `inputs: [{input: "prod", projects: "dependencies"}]`

Similar to `dependsOn`, the "^" symbols means "dependencies". This is a very important idea, so let's illustrate it with
an example.

```
"test": {
  "inputs": [ "default", "^prod" ]
}
```

The configuration above means that the test target depends on all source files of a given project and only prod
sources (non-test sources) of its dependencies. In other words, it treats test sources as private. If your `remixapp`
project depends on the `header` library, changing the `header` tests will not have any effect on the `remixapp` test
target.

### outputs

`"outputs": ["{projectRoot}/dist"]` tells Nx where the build script is going to create file artifacts. The provided
value is actually the default, so we can omit it in this case. `"outputs": []` tells Nx that the test target doesn't
create any artifacts on disk. You can list as many outputs as you many. You can also use globs or individual files as
outputs.

This configuration is usually not needed. Nx comes with reasonable defaults which implement the configuration above.

## Project-Specific Configuration

For a lot of workspaces, where projects are similar, `nx.json` will contain the whole Nx configuration. Sometimes, it's
useful to have a project-specific configuration, which is placed in the project's `package.json` file.

```json title="package.json"
{
  "name": "parent",
  "scripts": {
    "build": "...",
    "test": "..."
  },
  "dependencies": {...},
  "nx": {
    "namedInputs": {
      "prod": [
        "!{projectRoot}/**/*.test.tsx",
        "{workspaceRoot}/configs/webpack.conf.js"
      ]
    },
    "targets": {
      "build": {
        "dependsOn": [
          "^build"
        ],
        "inputs": [
          "prod",
          "^prod"
        ],
        "outputs": [
          "{workspaceRoot}/dist/parent"
        ]
      }
    }
    "implicitDependencies": ["projecta", "!projectb"]
  }
}
```

Note, the `namedInputs` and `targetDefaults` defined in `nx.json` are simply defaults. If you take that configuration
and copy it into every project's `package.json` file, the results will be the same.

In other words, every project has a set of named inputs, and it's defined as: `{...namedInputsFromNxJson, ...namedInputsFromProjectsPackageJson}`. Every target/script's `dependsOn` is defined
as `dependsOnFromProjectsPackageJson || dependsOnFromNxJson`. The same applies to `inputs` and `outputs`.

### inputs & namedInputs

Defining `inputs` for a given target would replace the set of inputs for that target name defined in `nx.json`.
Using pseudocode `inputs = packageJson.targets.build.inputs || nxJson.targetDefaults.build.inputs`.

You can also define and redefine named inputs. This enables one key use case, where your `nx.json` can define things
like this (which applies to every project):

```
"test": {
  "inputs": [
    "default",
    "^prod"
  ]
}
```

And projects can define their prod fileset, without having to redefine the inputs for the `test` target.

```json title="package.json"
{
  "name": "parent",
  "scripts": {
    "build": "...",
    "test": "..."
  },
  "dependencies": {...},
  "nx": {
    "namedInputs": {
      "prod": [
        "!{projectRoot}/**/*.test.js",
        "{workspacRoot}/jest.config.js"
      ]
    }
  }
}
```

In this case Nx will use the right `prod` input for each project.

### dependsOn

Defining `dependsOn` for a given target would replace `dependsOn` for that target name defined in `nx.json`.
Using pseudocode `dependsOn = packageJson.targets.build.dependsOn || nxJson.targetDefaults.build.dependsOn`.

### outputs

Defining `outputs` for a given target would replace `outputs` for that target name defined in `nx.json`.
Using pseudocode `outputs = packageJson.targets.build.outputs || nxJson.targetDefaults.build.outputs`.

### implicitDependencies

The `"implicitDependencies": ["projecta", "!projectb"]` line tells Nx that the parent project depends on `projecta` even
though there is no dependency in its `package.json`. Nx will treat such a dependency in the same way it treats explicit
dependencies. It also tells Nx that even though there is an explicit dependency on `projectb`, it should be ignored.

## Additional Configuration

For additional ways to configure tasks and caching, see the relevant [Nx documentation](https://nx.dev/recipes/running-tasks).

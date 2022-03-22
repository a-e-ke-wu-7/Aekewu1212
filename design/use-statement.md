# The `use` statement

There are currently 9 different [tool or installer](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/go-tool?view=vsts) tasks.
Conceptually they do roughly the same thing (in the user's mind):

- Get a known version of some tool, maybe from the internet
- Set up the CI environment so that when I invoke `tool`, I get that version

## Problems: tool installers

### Semantics of the task

Some tools are hard to install on the fly.
So, these are really "use a version from some limited subset of possible versions".
Others can be used to fetch arbitrary versions from the internet.
We introduced a split between "use version" and "install tool", but this has turned out to be more confusing than helpful in practice.

### Naming

The tasks are not named consistently, and they take inconsistent inputs.
We have 2 `*Tool` tasks, 3 `*Installer` tasks, 2 `*ToolInstaller` tasks, and 2 `Use*Version` tasks.
Some take a `version`, others a `versionSpec`, and one a `versionSelector`.
See the appendix for a list of current tasks.

### YAML syntax

Some of the tasks take complex, interdependent inputs (Visual Studio Test Platform Installer).
This is really hard to work with in YAML.
Also, our `taskName@taskVersion` syntax adds a layer of confusion.

```yaml
- task: UsePythonVersion@0
  inputs:
    versionSpec: 3.x
```

The tasks are all at version 0 right now, so real-world confusion is *probably* limited.

## Problems: authentication

### Authenticated feeds

Builds commonly require access to authenticated package sources (called 'feeds' in Azure
Pipelines and [Azure Artifacts](https://docs.microsoft.com/azure/devops/artifacts)) to restore
and/or publish packages. Each ecosystem has its own tools, each of which handles authentication
slightly differently.

### Task proliferation

Today, to use authenticated npm feeds, you probably need a Tool task, an Authenticate task, and
a script task that actually does the work you want to do.

### Support for scripting

Today, we don't provide Authenticate tasks for all package types. The prior strategy was to build
authentication support directly into full-featured tasks like the `NuGet` and `npm` tasks. However,
this precludes using authenticated feeds from scripting languages and task runners like `gulp` and
is not compatible with the YAML build tenet of using scripts wherever possible and only using tasks
when advanced features/workflows are required.

### Other things we want to include

- Proxy setup: The agent knows about its proxy; the tasks don't always pass that along to client tooling.
- Problem matchers: If the Python compiler emits errors in a known format, we should be able to read that via regex and surface it in our UI. VS Code has a defined format for problem matchers.

## Solution

We'll add new YAML syntax which puts tools on the `PATH`, adds proxy information, adds problem matchers, and optionally configures auth.
This syntax will be powered by tasks, giving a familiar extensibility point for third parties to work with.

### New syntax

YAML will get a new syntax for `use`-ing a tool or ecosystem.

```yaml
- use: {toolName}
```

`{toolName}` may contain characters in [A-Za-z0-9], plus `-` and `_`.

This can be implemented as sugar over the existing syntax, much like the `powershell` and `bash` keywords today.
You'll be bound to the latest major version of the task.

### Version

To select a particular version, pass a `{versionSpec}` to `version`.

```yaml
- use: {toolName}
  version: {versionSpec}
```

`{versionSpec}` is a SemVer or SemVer-like string; see below.

### Proxy setup

By default, `use` will set up the target ecosystem to use the agent's proxy.
If this behavior isn't desired, it can be overridden with:

```yaml
- use: someTool
  proxy: false
```

### Auth

If a tool needs credentials, those can be added like this:

```yaml
steps:
- use: someTool
  authTo: azureArtifactsFeedOrServiceConnection
```

or, for multiple resources:

```yaml
steps:
- use: someTool
  authTo:
  - azureArtifactsFeed
  - artifactoryServiceConnection
  - myGetServiceConnection
```

It is recommended to check in a configuration file showing which feeds are required.
If this file is checked in but not in the root of the repo, a path can be provided.
If no path is provided and no file is checked in, a standard configuration file will
be generated with the feeds/service connections listed. This covers scenarios where
project admins have instead chosen to have each dev manually connect to their private
feed (e.g. by using the NuGet Package Manager settings dialog in Visual Studio).

```yaml
steps:
- use: someTool
  authTo: azureArtifactsFeedOrServiceConnection
  inputs:
    authFile: .nuget/nuget.config
```

Tasks which implement this contract can expect a service connection at runtime.
It's up to the resource provider to convert other forms of auth (such as a raw username + password) into a service connection.
Tasks must accept an array of inputs, though often the array will only contain 1 entry.

### Problem matchers

Problem matchers in general require no input.
See [VS Code's information about problem matchers](https://code.visualstudio.com/docs/editor/tasks#_defining-a-problem-matcher) to learn more.

### Other inputs

Some ecosystems will require optional, additional inputs.
Just like on `task`, you can pass a map of `inputs`.

```yaml
- use: python
  inputs:
    architecture: x64
```

We need to allow tasks the ability to remove fields in major versions.
Therefore, if a task doesn't accept one of the inputs, we won't fail the build but we will inject a warning about the mismatch.
Depending on the scenario, the build could still fail at runtime.
This makes the issue debuggable even if not ideal.

## Schema

```yaml
- use: string               # required tool name
  version: string           # optional version
  proxy: boolean            # whether to install proxy information; defaults to true
  authTo: string | [string] # optional names of feeds or service connections to authenticate
  inputs: {string: any}     # optional additional arguments to pass as task inputs
```

## Example

```yaml
strategy:
  matrix:
    PY35-NODE8:
      py_version: '3.5'
      node_version: '8'
    PY35-NODE10:
      py_version: '3.5'
      node_version: '10'
    PY27-NODE8:
      py_version: '2.7'
      node_version: '8'
    PY27-NODE10:
      py_version: '2.7'
      node_version: '10'

steps:
- use: python
  version: $(py_version)
- use: node
  version: $(node_version)
- script: npm install ...
- script: python ...
```

*Note, as there's been confusion:* nothing changes in the `matrix` section.
It's purely about the `steps`.

### Task changes

The in-box tool installers and Use*Tools will evolve.

Task requirements:

- task.json includes a field indicating what ecosystem the task provides.
Something like `{ 'provides': string }`.
- Accept `version` as an input.
By convention, this should be a SemVer version spec.
It's up to the task to process these (with help from the task lib).
If an ecosystem has additional, non-SemVer "versions", the task may accept those as well.
To support container jobs, `version` is optional.
It's strongly recommended for VM jobs.
- Document the range of acceptable version numbers.
Where it's "install from the internet", say so.
Where it's "from a list in the hosted tools cache", that should be clear in the docs.
Also, we'll make sure all the tooling includes this information - for instance, Intellisense.
  - Future work: some way to acquire the list of available versions at runtime.
- Some tools have multiple architectures available.
By convention, an `architecture` field should use the agent architecture types: `x64`, `x86`, and `arm`.
Others can be added, and of course only the relevant ones should be accepted.

Additional requirements for in-box tasks:
- By convention, in-box tasks will use a consistent name scheme: `Use<Name>`.
This won't be required for third-party tasks.
- Another common parameter is `architecture`.
- Do a simplification pass on each task to make sure each input is necessary, clear, and orthogonal.

## Appendix: List of existing tool and installer tasks

- [GoTool](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/go-tool?view=vsts)
- [NodeTool](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/node-js?view=vsts)
- [HelmInstaller](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/helm-installer?view=vsts)
- [DotNetCoreInstaller](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/dotnet-core-tool-installer?view=vsts)
- [VisualStudioTestPlatformInstaller](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/vstest-platform-tool-installer?view=vsts)
- [JavaToolInstaller](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/java-tool-installer?view=vsts)
- [NuGetToolInstaller](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/nuget?view=vsts)
- [UsePythonVersion](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/use-python-version?view=vsts)
- [UseRubyVersion](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/use-ruby-version?view=vsts)

## Appendix: List of existing authenticate tasks

- [npmAuthenticate](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/package/npm-authenticate?view=vsts)
- [pipAuthenticate](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/package/pip-authenticate?view=vsts)
- [TwineAuthenticate](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/package/twine-authenticate?view=vsts)

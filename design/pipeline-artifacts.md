# Pipeline Artifacts YAML shortcut

**Status: Awaiting final review**

Pipeline Artifacts are the new way to move files between jobs and stages in your pipeline. They are hosted in Azure Artifacts and will eventually entirely replace FCS "Build Artifacts". Because moving files between jobs and stages is a crucial part of most CI/CD workflows, and because Pipeline Artifacts are expected to be the default way to do so, this spec proposes a YAML shortcut syntax for publishing and downloading artifacts.

In this document, `artifact` refers specifically to publishing Pipeline Artifacts from the current pipeline. `downloadArtifact` refers to downloading Pipeline Artifact artifacts from the current pipeline, from other Azure Pipelines, and from other pipeline resources (e.g. Jenkins builds). 

Artifacts are distinct from other `resources` types, including `containers`, `repositories`, and `feeds`.

## Publishing artifacts: `artifact`

`artifact` is a shortcut for the [Publish Pipeline Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/publish-pipeline-artifact.md) task. It will publish files from the current job to be used in subsequent jobs or in other stages or pipelines.

### Schema

```yaml
- artifact: string | [ string ]  # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to publish
  name: string # identifier for this artifact (no spaces allowed), defaults to 'default'
  prependPath: string # a directory path that will be prepended to all published files
  seal: boolean # if true, finalizes the artifact so no more files can be added after this step
```

### Example

```yaml
- artifact:
  - **/bin/*
  - **/obj/*
  name: webapp
  seal: true
```

---

## Downloading artifacts: `downloadArtifact`

`downloadArtifact` is a shortcut for several tasks:

- The [Download Pipeline Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/download-pipeline-artifact) task
- The [Download Fileshare Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/download-fileshare-artifacts) task
- Download tasks for other pipelines systems e.g. Jenkins

It will download artifacts published from a previous job or stage or from another pipeline. Artifacts are downloaded either to `$PIPELINES_RESOURCESDIR` or to the directory specified in `root`.

### Schema

```yaml
- downloadArtifact: string # identifier for the resource from which to download artifacts, optional; defaults to 'self`
  name: string # identifier for the artifact to download; if left blank, downloads all artifacts associated with the resource provided
  patterns: string | [ string ] # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
  root: string # the directory in which to download files, defaults to $PIPELINES_RESOURCESDIR
```

### Examples

```yaml
- downloadArtifact:
  name: webapp
- downloadArtifact: tools-pipeline
```

## More details about Pipeline Artifacts

### Default and named artifacts

Every Pipeline run has a default artifact (named `default`). You can also create multiple artifacts, each with their own name. All artifacts (including `default`) are automatically downloaded to each subsequent job's resources directory (`$(Pipelines.ResourcesDir)`). You can limit the artifacts downloaded by adding a `downloadArtifact` step to the beginning of a job.

### Multi-publish artifacts

You can add files to any artifact multiple times in the same job, in different jobs in the same stage, and in different stages in the same pipeline, or you can `seal` an artifact at any time to prevent further files from being added.

## Examples

### Publish a build artifact and use it in a deployment

This is a simple pipeline that include a build job and a deployment job. The built artifact is provided to the deployment job using the default Pipeline Artifact.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - artifact: bin/*
- job: Deploy
  steps:
  - script: |
      TODO-some-cool-deploy-script-here $(Pipelines.ResourcesDir)/default/bin/
```

### Specify a custom location for a build artifact

You can control the location where artifacts are downloaded using the `downloadRoot` key.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - artifact: bin/*
- job: Deploy
  steps:
  - downloadArtifact:
    root: $(Pipelines.SourcesDir)/from-build/
  - script: |
      TODO-some-cool-deploy-script-here $(Pipelines.SourcesDir)/from-build/bin/
```

### Add to an artifact multiple times

You can add files to both the default artifact and to named artifacts until the end of the pipeline or until a `artifacts` step is run with the `seal` key set to `true`.

```yaml
- job: Build .NET Core
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - artifact: bin/*
    prependPath: netcore/
- job: Build .NET Framework
  steps:
    - task: VSBuild@1
      inputs:
        solution: MySolution.sln
    - artifact: bin/*
      prependPath: netfx/
      seal: true
- job: Deploy .NET Core
  steps:
  - script: |
      TODO-some-cool-deploy-script-here $(Pipelines.ResourcesDir)/default/netcore/bin/
```

### Publish a named artifact

You can give an artifact a name, and you can publish multiple named artifacts. All artifacts are downloaded unless you specify a `downloadArtifact` step to limit the artifacts that are downloaded.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - artifact: bin/WebApp/*
    name: WebApp
  - artifact: bin/MobileApp/*
    name: MobileApp
- job: Deploy
  steps:
  - downloadArtifact: WebApp
  - downloadArtifact: MobileApp
  - script: TODO-some-cool-deploy-script-here $(Pipelines.ResourcesDir)/WebApp/bin/
  - script: TODO-xamarin-magic $(Pipelines.ResourcesDir)/MobileApp/
```
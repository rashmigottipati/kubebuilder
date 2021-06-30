# Extensible CLI and Scaffolding Plugins - Phase 2

## Overview

* Plugin Phase 1.5 was desgined and implemented to achieve chaining of plugins. The main goal of Phase 2 plugins is Kubebuilder discovering and using external plugins, also referred as out-of-tree plugins. So Phase 2 achieves both chaining of plugins and discovery of external plugins/source code which is not intented to be shipped with the binaries. For e.g., Kubebuilder being able to discover the helm plugin and use that without having to build the plugin with the binary. These external plugins can be implemented in any language.

### Related issues and PRs

* [Feature Request: Plugins Phase 2](https://github.com/kubernetes-sigs/kubebuilder/issues/1378)
* [Extensible CLI and Scaffolding Plugins - Phase 1.5](https://github.com/kubernetes-sigs/kubebuilder/blob/master/designs/extensible-cli-and-scaffolding-plugins-phase-1-5.md)
* [Phase 1.5 Implementation PR](https://github.com/kubernetes-sigs/kubebuilder/pull/2060)
* [Plugin Resolution Enhancement Proposal](https://github.com/kubernetes-sigs/kubebuilder/pull/1942)

### Use Cases

* As a CLI user, I would like to be able to provide external source code of my plugins and its boilerplates for the CLI to be able to use them by extending the subcommands provided by Kubebuilder, which will not have the plugin binaries be shipped by default.

* As a Kubebuilder/Operator SDK maintainer, I would like to be able to export an internal plugin and make that an external source, so that the plugin is no longer officially maintained by the project but still available to be used.
  * For e.g., once phase 2 plugin implementation is completed, converting some internal plugins in OpenShift as external ones will remove the necessity to ship those plugins along with the OpenShift binary.

### Goals

* Kubebuilder is able to discover plugin binaries and run plugins using the CLI.

* Kubebuilder can use the external plugins as its own internal ones to do scaffolding and be able to show plugin specific information such as via the --help flag.

* Operator SDK is able to extend Kubebuilder and provide the same functionality for plugins chaining and enabling out-of-tree plugin discovery.

### Non-Goals

* Addition of new arbitrary subcommands other than the subcommands that we already support i.e init, create api, and create webhook.

* Discovering plugin binaries that are not locally present on the machine (i.e. exists in a remote repository).

* Discovering flags that each individual plugin would support
  * Flags would have to be provided by the user as a part of project configuration.

* Providing other options (other than stdin stdout) for inter-process communication between Kubebuilder and the plugins.

### Examples

* `kubebuilder create api --plugins=myexternalplugin/v1`
  * should scaffold files using the source code of the external plugin as defined in its implementation of the create api method.

* `kubebuilder create api --plugins=myexternalplugin/v1,myexternalplugin/v2`
  * should scaffold files using the source code of the external plugin as defined in their implementation of the create api method(by respecting the plugin chaining order, i.e. in the order of create api of v1 and then create api of v2 as specified in the layout field in the configuration).

* `kubebuilder create api --plugins=myexternalplugin/v1 --help`
  * should provide help implementation of the plugin which is not shipped in the binary (source code is present outside of the KB repo).

### How should Kubebuilder discover plugin binaries

Main goal of phase 2 plugins is discovery of out of tree plugins that are external to Kubebuilder and not shipped with the CLI. To be able to run these plugins via the Kubebuilder CLI, it’s necessary for kubebuilder to discover these external plugin binaries and use them as internal ones. I will discuss a few approaches in the order of suitability and preference, with their respective pros and cons.

#### Approach #1

User specified file paths based approach: User will provide a list of file paths for Kubebuilder to discover the plugins in. We will define a variable `KUBEBUILDER_PLUGINS_DIRS` that will take a list of file paths to search for the plugin name. It will also have a default value to search in, in case no file paths are provided. It will search for the plugin name that was provided to the `--plugins` flag in the CLI. Kubebuilder will recursively search for all file paths until the plugin name is found and returns the successful match, and if it doesn’t exist, it returns an error message that the plugin is not found in the provided file paths.

* Pros
  * This provides flexibility for the user to specify the file paths that the plugin would be placed in and Kubebuilder could discover the binaries in those user specified file paths.

  * No constraints on plugin binary naming or directory placements from the Kubebuilder side.

  * Provides a default value for the plugin directory in case user wants to use that to drop their plugins.

* Cons
  * Adds some complexity as we need to have an extra flag to have the user provide those file paths and/or make PROJECT file changes to add an extra field to `plugins` field that is a comma separated list of file paths for every plugin that is defined in the `layout` field.

#### Approach #2

This is how [kustomize](https://kubectl.docs.kubernetes.io/guides/extending_kustomize/) discovers plugins.
The PROJECT file in Kubebuilder has `plugins` field and each plugin will have the resources: Group, Kind and Version, so every plugin defined in the PROJECT file configuration can get its own GVK directory for the executable to be placed in and Kubebuilder will search for a plugin with the name of the plugin in the GVK directory of the plugin. Once KB successfully locates the plugin, it will run the plugin using the CLI.

Following `kustomize`'s way of placement, every plugin gets its own directory such as:

```project
    $XDG_CONFIG_HOME/kubebuilder/plugins
    /${group}/${version}/LOWERCASE(${kind}) 
```

The default value of XDG_CONFIG_HOME is $HOME/.config.

* Pros
  * `kustomize` which is popular and robust tool, follows this approach in which `apiVersion` and `kind` fields are used to locate the plugin.

  * This appoach does not enforce naming of plugins but enforces the directories the plugins have to be in, which is a more common and typical approach.

  * The one-plugin-per-directory requirement eases creation of a plugin bundle for sharing.

* Cons
  * This makes it mandatory for every plugin that is specified in the `layout` field to have additional configuration information such as Group, Version and Kind in the `plugins` field in the PROJECT file for it to be discoverable. Currently, the `plugins` field is more like an optional configuration storage field.

#### Approach #3

Another approach I would like to mention is adding plugin executables with a prefix `kubebuilder-` followed by the plugin name to the PATH variable. This will enable KB to traverse through the PATH looking for the plugin executables starting with the prefix `kubebuilder-` and matching by the plugin name that was provided in the CLI. Furthermore, a check should be added to verify that the match is an executable or not and return an error if it's not an executable. This approach provides a lot of flexibility in terms of plugin discovery as all the user needs to do is to add the plugin executable to the PATH and Kubebuilder will discover it.

* Pros
  * `kubectl` and `git` follow the same approach for discovering plugins, so there’s prior art.

  * There’s a lot of flexibility in just dropping plugin binaries to PATH variable and enabling the discovery without having to enforce any other constraints on the placements of the plugins.

* Cons
  * Enumerating the list of all available plugins might be a bit tough compared to having a single folder with the list of available plugins and having to enumerate those.

  * These plugin binaries cannot be run in a standalone manner outside of Kubebuilder, so may not be very ideal to add them to the PATH var.

#### What Plugin system should we use

Based on the evaluation of plugin libraries such as the [built-in go-plugin library](https://golang.org/pkg/plugin/) and [Hashicorp’s plugin library](https://github.com/hashicorp/go-plugin), it is more suitable to write our own custom plugin library as the built-in plugin library seems to be more suitable for in-tree plugins rather than out-of-tree plugins and it doesn’t offer cross language support, thereby making it a non-starter. Hashicorp’s go plugin system is more suitable than the built-in go-plugin library as it enables cross language/platform support. However, it is more suited for long running plugins as opposed to short lived plugins and the usage of protobuf could be overkill as we will not be handling 10s of 1000s of deserializations.

For the stated reasons, I propose we use our own plugin system that passes JSON blobs back and forth across stdin/stdout and make stdin/stdout the only default option for now as it’s the most language agnostic way and it makes it easy to work with most languages.

In the future, if a need arises (for e.g. users are hitting performance issues), we can then explore the possibility of using the Hashicorp’s go plugin library. From a design standpoint, to leave it architecturally open, I propose using a “type” field in the PROJECT file to potentially allow other plugin libraries in the future and make this a seperate field in the PROJECT file per plugin; and this field determines how the injector will be passed for a given plugin. However, for the sake of simplicity in initial design and not to introduce any breaking changes as Project version 3 would suffice for our needs, this option is out of scope in this proposal.

#### Project configuration

Currently, the project configuration has two fields to store plugin specific information.

* `Layout` field (of type string) is used for plugin chain resolution on initialized projects. This will be the default if no plugins are specified for a subcommand.
* `Plugins` field (of type map[string]interface{}) is used for option plugin configuration that stores configuration information of any plugin.

* So now, where should external plugins be defined in the configuration?

  * I propose that the external plugin should get encoded in the project configuration as a part of the `layout` field.
    * For e.g., external plugin `helm.sdk.operatorframework.io/v2-alpha` can be specified through the `--plugins` flag for every subcommand and also be defined in the project configuration in the `layout` field for plugin resolution.

Example `PROJECT` file:

```yaml
version: "3"
domain: testproject.org
layout: 
- go.kubebuilder.io/v3
- helm.sdk.operatorframework.io/v2-alpha
plugins:
  helm.sdk.operatorframework.io/v2-alpha:
    resources:
    - domain: testproject.org
      group: crew
      kind: Captain
      version: v2-alpha
  declarative.go.kubebuilder.io/v1:
    resources:
    - domain: testproject.org
      group: crew
      kind: FirstMate
      version: v1
repo: github.com/test-inc/testproject
resources:
- group: crew
  kind: Captain
  version: v1
```

#### What to pass between Kubebuilder and plugins

<!--- TODO(rashmigottipati): This section needs more details on what structure to pass to the plugins. --->

Currently, the `injector` type is defined as below:

```go
// injector is used to inject certain fields to file templates.
type injector struct {
    // config stores the project configuration.
    config config.Config

    // boilerplate is the copyright comment added at the top of scaffolded files.
    boilerplate string

    // resource contains the information of the API that is being scaffolded.
    resource *resource.Resource
}
```

One of the plugin hooks allows plugins to set up their own flags. Commong flags (like `--group`, `--version`, `--kind`) are defined by the plugin system itself, and the resource created with those values populated will be passed to the plugins. `create api` and `create webhook` commands would require the GVK being passed as well to the plugins. Currently, `InjectResource` is a required hook for `create api` and `create webhook` commands and the entire resource (which contains the group, version and kind) is being passed to the plugins for these commands.

All the raw flags of the plugins would have to be passed down to the plugins through injector. The plugin specific flags would have to be specified in the project configuration by the user. And the current implmentation is such that, every subcommand can optionally implement the `InjectConfig` hook which passes the project configuration to the plugins and also enable the plugins to modify it if required.

Along with flags, boilerplate with empty copyright or an apache license and in go-style comments is also being passed down to the plugins through injector. If other kind of comments were to be supported, this logic would have to be added (scaffold markers do support YAML and Go-style comments).

Additionally, we would pass the entire CLI environment to the plugin, as anything executing in subshell environment would be able to read the environment variables even if it's being restricted in terms of what Kubebuilder passes to it.

For displaying help text information of the plugins, `UpdateMetadata` hook is the one that allows to modify the description and examples for the `--help` message of plugins. Currently, there is no support for help implementaion of plugins so logic would have to be added to the `UpdateMetadata` hook to display plugin specific help.

#### Handling plugin failures across the chain

If any plugin in the chain fails, we will halt the chain execution and report errors as one plugin may be dependent on the success of another. We can use `stderr` for the plugin to write back warnings/errors in a structured format that Kubebuilder understands and return a process exit code 1. The successful return type with exit code 0 contains the list of files that were scaffolded and the file contents will be given back to Kubebuilder.

In the case of plugin failures even if one plugin across the chain fails, all the files that were scaffolded already until that point will not be written to disk to prevent a half committed state.

#### Open questions

* Do we want to support the addition of new arbitrary subcommands other than the subcommands (init, create api, create webhook) that we already support?

* Do we need to discover flags by calling the plugin binary or should we have users define them in the project configuration?

* What alternatives to stdin/stdout exist and why shouldn't we use them?
  * Other alternatives exist such as named pipe and sockets, but stdin/stdout seems to be more suitable for our needs.

* What happens when two plugins bind the same flag name? Will there be any conflicts?

* Should plugins depend on environment variables?
  * The same plugin would behave differently based on the environment.

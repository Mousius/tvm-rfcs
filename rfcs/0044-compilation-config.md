- Feature Name: Compiler Configuration Representation
- Start Date: 2021-10-26
- RFC PR: [apache/tvm-rfcs#44](https://github.com/apache/tvm-rfcs/pull/0000)
- GitHub Issue: [apache/tvm#0000](https://github.com/apache/tvm/issues/0000)

# Summary
[summary]: #summary

This RFC supersedes [Migrating IRModules to Attributes](https://github.com/apache/tvm-rfcs/blob/main/rfcs/0029-migrating-to-irmodule-attributes.md) by replacing the various attributes associated with the `Runtime` and `Executor` with a single `CompilationConfig` object that encapsulates the configuration of the compiler for a given `IRModule`. By collecting this together, it introduces on object which can be coupled to the `IRModule` and used through-out compilation with guarantees about the properties of the configuration.

# Motivation
[motivation]: #motivation

## Argument Duplication
When implementing [Migrating IRModules to Attributes](https://github.com/apache/tvm-rfcs/blob/main/rfcs/0029-migrating-to-irmodule-attributes.md), it became clear that the arguments were getting duplicated in many places across the codebase and never collated. Such as the following `tvmc` CLI:

```
tvmc --target=c --executor=aot --runtime=crt
```

Which populates the arguments `executor`, `runtime` and `target` all the way into the compilation flow, rather than collating them and passing the pre-processed representation. This introduces places we can make errors due to missing one of the three and always having to replicate the signature.

## Single Point of Setup
Futher to this, many areas of the code required access to the compiler configurations non-optionally, this introduces uncertainty in later passes which should be able to guarantee the compiler has been configured - thus making the dynamic attributes less ideal for this use-case:

```cpp
// Hopefully this is set!
ir_mod->GetAttr<Executor>(kExecutor).value()->name;
```

As can be seen by [adding SEScope](https://github.com/apache/tvm/pull/9313), there's also need for a single place to collect and define configuration rather than cause duplication of effort, with multiple calls to setup various parts of the configuration, such as `Target` environment which should be known when the compilation flow begins:
```cpp
CheckAndUpdateHostConsistency(&target, &target_host);
```

By moving these to a single property of the `IRModule` which is required to exist before entry into the compiler flow, this reduces both the cognitive overhead of accessing the configuration and provides a place to encapsulate the logic of setup early in the compilation flow.

This is a fixed set of configuration alongside the existing `PassContext::Global()` to solidify the configuration requirements of the TVM Compiler.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## User API
A user creates and passes a configuration to `relay.build(mod, config)`, leading to something similar to:
```python
ir_mod = IRModule.from_expr(function)
config = CompilationConfig(
    target_host=target,
    targets=[target],
    executor=Executor("aot", {"interface-api": "c", "unpacked-api": True}),
    runtime=Runtime("crt", {"system-lib": True})
)
relay.build(ir_mod, config)
```

To easily create a default `CompilationConfig` from a single `Target`, the `from_target` option is provided - this can be used to add syntactic ease in `relay.build`:
```python
def relay.build(ir_mod, config, ...):
  if isinstance(config, Target):
    config = CompilationConfig.from_target(config)
```

The C++ API is then amended to require configuration:
```cpp
/*!
   * \brief Build relay IRModule
   *
   * \param mod Relay IRModule
   * \param config Compilation Config for this compiler run
   * \param mod_name Name of the module
   */
  void Build(IRModule mod, const CompilationConfig& config, const String mod_name)
```

This means a user can continue to easily create and pass an `IRModule` to `relay.build` as an opaque object whilst providing the configuration alongside.

## Developer API
The developer facing API for the `IRModule` changes for internal passes to easily access the configuration:
```cpp
ir_mod->GetConfig()->GetExecutor()
```

And functionality related to inspecting this configuration can be added to `CompilationConfig` which can see all available configuration properties:
```cpp
ir_mod->GetConfig()->ShouldLinkParams()
```

Importantly, for ad-hoc information passed alongside with the `IRModule`, attributes continue to be available:
```cpp
ir_mod->SetAttr<String>("woof", "woof");
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## IRModule Configuration
To incorporate this into the `IRModule`, a property of type `CompilationConfig` will be added and exposed via a function `GetConfig()` in line with [the Google C++ guidelines](https://google.github.io/styleguide/cppguide.html) except for being a public property for TVMs internal node structures:

```cpp
class IRModuleNode {
  CompilationConfig config;

  const CompilationConfig& GetConfig() {
    return config;
  }
}
```

The actual `CompilationConfig` class will represent the current and future shape of configuration, such as `Executor` and `Runtime`, alongside `Target` and `Target` host (example illustrative not carved into stone):
```cpp
class CompilationConfigNode {
  Target target_host;
  Array<Target> targets;
  Runtime runtime;
  Executor executor;
}
```

From Python the module will be passed into C++ with the `CompilationConfig` to start the compilation flow:
```python
def relay.build(ir_mod, config: Union[CompilationConfig, Target], ...):
    if isinstance(config, Target):
        config = CompilationConfig.from_target(config)
    mod["build"](ir_mod, config)
```

Which is then connected within C++:
```cpp
void Build(IRModule mod, const CompilationConfig& config, const String mod_name) {
    mod->config = config;
    ...
}
```

When creating `IRModule` -> `IRModule` passes within the compiler this property should now be guaranteed to exist in the main Relay flow.

## Non-Relay Flows
There are a number of ways of accessing the internals of TVM which have been exposed to the user, for these to continue to function the `CompilationConfig` will be settable from Python such that:

```python
ir_mod = ir_mod.with_configuration(config)
```

The above will attach the configuration within the alternative pathways, it is the opinion of the author that these should be eventually deprecated in favour of a single standard compilation path.

## Compilation Configuration Creators
Within `python/tvm/target/target.py` there are a number of functions which returned `Target`s with configuration built in, such as:

https://github.com/apache/tvm/blob/e8077439efbe021d73cf27018cd83653f964255f/python/tvm/target/target.py#L305-L328

These are user-facing configurations, and will be ported to `tvm/target/config.py` to match the `include/tvm/target/compilation_config.h` and provide the same logic but returning a `CompilationConfig`:

```python
def micro(model="unknown", options=None, executor="graph"):
  opts = _merge_opts(
      MICRO_SUPPORTED_MODELS[model] + ["-runtime=c", f"-model={model}"],
      options,
  )
  target = Target(" ".join(["c"] + opts))
  return CompilationConfig(
    target_host=[target],
    targets=[target],
    executor=Executor(executor),
    runtime=Runtime("crt", {
      "system-lib": executor == "graph"
    })
  )
```

# Drawbacks
[drawbacks]: #drawbacks

- This is more information on the `IRModule` alongside existing attributes and pass information
- The entry contract for `relay.build` and the TVM Compiler changes to require configuration of the initial `IRModule`

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Continue to use `PassContext::Global()` and general purpose attributes for fixed compilation configuration, using the general purpose options here hides the details of the compilation configuration and makes it harder for us to provide a robust representation of none-`Optional` configuration. If all things are considered `Optional` then they have to be considered as none-configured and requiring multiple passes of reconfiguration to provide safety to the users.

# Prior art
[prior-art]: #prior-art

- [Migrating IRModules to Attributes](https://github.com/apache/tvm-rfcs/blob/main/rfcs/0029-migrating-to-irmodule-attributes.md) split the settings from `Target` into `Executor` and `Runtime` attributes
- `CompilationConfig` was introduced in [Add SEScope PR #9313](https://github.com/apache/tvm/pull/9313#issuecomment-955732036)
- `PassContext` already exists in TVM as a way of passing general configuration

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Do we want to fully deprecate passing a `target` to `relay.build`? The assumption is made that we don't want to fully deprecate it and a sugar has been added above for populating a `CompilationConfig` straight from a `Target`
- Similarly, do we want to provide a similar helper for `target` and `target_host`:
```python
def relay.build(ir_mod, config, target_host=None):
  if isinstance(config, Target):
    config = CompilationConfig.from_target(config, target_host=target_host)
```
- What is the future for `IRModule` attributes and `PassContext` - this RFC aims to provide one piece of concrete configuration which we know can be fixed at the start of compilation but there should be a default choice for ad-hoc configuration within the compilation flow.

# Future possibilities
[future-possibilities]: #future-possibilities

By codifying the properties of the compilation flow in `CompilationConfig` and providing `IRModule` attributes we may be able to deprecate `PassContext::Global()` and only use the passed state in TVM.
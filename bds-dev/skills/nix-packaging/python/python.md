# Package a python module with Nix

Packaging a python module (`buildPythonPackage`) is mainly about figuring out the source, the dependencies and the build-system, usually from the `pyproject.toml` file. Ensure dependencies are already packaged before referencing them.

More complicated situation might need special treatments. Below are the examples for different situations.

## Normal Python Module

Just follow the simple example: [tyro](./tyro/package.nix)

## Customize Build System and Disabling Tests

Sometimes the target python module may need access to more tools when build, and sometimes we want to disable specific tests due to for example the tests cannot run in sandbox. Follow this example [Swanboard](./swanboard/package.nix) when need.

## Customize Source Root and Optionalk Dependences

This example [agno](./agno/package.nix) demonstrates that 

- If you find that the python module's root is not the project root of the software, use `sourceRoot` to specify the actual relative path with respect to the project root.
- When there are optional dependencies, it is encouraged to put them in `optional-dependencies` if we can; it is fine to ignore them if we do not have the dependencies packaged already.

## Rust Backed Python Module

If the python package is actually implemented in rust and then wrapped, we can use `rustPlatform` as a helper. Use [tantivy](./tantivy/package.nix) as an example if needed.

## Relax or Remove Dependencies

Sometimes the dependencies requirement cannot be satisfied. Do not give up but investigate further

1. If the dependency is packaged but the version requirement is not satisfied, determine whether the version requirement is necessary. If not, add it to `pythonRelaxDeps` to bypass the check.
2. If the dependency is not packaged, determine whether this dependency is absolutely necessary based on the use case of the target python module. It may well be acceptable to use the target python module without this particular dependency, and in this case, add it to `pythonRemoveDeps`.

Use [darts](./darts/package.nix) as an example if needed.

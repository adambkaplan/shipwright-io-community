<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: project-look-and-feel
authors:
  - "@adambkaplan"
reviewers:
  - TBD
approvers:
  - "@oscardoe"
creation-date: 2022-04-20
last-updated: 2022-04-20
status: provisional
see-also: []
replaces: []
superseded-by: []
---

# Uniform Project Look And Feel


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

TBD

## Summary

This SHIP establishes a standard project structure for Shipwright backend projects.
This structure is based on [Kubebuilder](https://book.kubebuilder.io/), with enhancements using
tools that the community has found beneficial.

## Motivation

As the Shipwright community has grown, additional projects have emerged from the original "Build"
repository.
This evolution has been organic, with each contributor bringing their own ideas on tools to use and
best practices to follow.
Unfortunately, this has resulted in a codebase that can feel inconsistent and chaotic, making it
difficult for contributors to provide enhancements and fixes across repositories.
Agreeing to a standard structure will help community members make contributions between
repositories and help new community members navigate the codebase.

### Goals

1. Outline a standard structure for Shipwright controllers.
2. Define the toolchain that will be used to build and deploy container images.

### Non-Goals

1. Define a standard structure for the shp cli. The cli is built on different libraries and
   should not be beholden to the KubeBuilder structure.
2. Define a standard structure for the website. The website is built with [Hugo](https://gohugo.io/)
   and should conform to this framework's structure.
3. Centralize tooling for testing with GitHub actions.
4. Centralize release tooling.
5. Prescribe a standard structure or format for documentation.

## Proposal

### Project Structure

Controller repositories should standardize on the following form, based on the KubeBuilder v3
project structure:

```
.github/
api/
client/
cmd/
config/
controllers/
docs/
hack/
pkg/
test/
tools/
vendor/
.gitignore
CONTRIBUTING.md
LICENSE
go.mod
go.sum
main.go
Makefile
PROJECT
README.md
```

The following directories and files are derived from the Kubebuilder v3 format:

- `api/`: Expected when generating APIs and webhooks
- `config/`: Always expected
- `controllers/`: Expected when generating controllers
- `hack/`: Expected when generating deep copy functions
- `.gitignore`
- `LICENSE`
- `go.mod`
- `go.sum`
- `main.go` : Required by kubebuilder
- `Makefile`
- `PROJECT`

Keeping this structure ensures that Kubebuilder's command line utilites can be used to simplify
future development (for example - adding new APIs, controllers, and webhooks).

The following directories are extensions based on our own experiences with Shipwright:

- `.github/`: Configuration for GitHub and GitHub Actions.
- `client/`: Generated client code. This is optional, and will hold generated go client code.
- `docs/`: In-tree documentation.
- `pkg/`: Business logic that can be shared between controllers and/or tested outside of a
  reconciliation loop.
- `test/`: Test specifications for integration and end-to-end testing.
- `tools/`: Additional tooling that is imported with go modules. This an informal [project layout standard](https://github.com/golang-standards/project-layout#tools).
- `vendor/`: Vendor tree for go code.

### Tooling

Most of the tooling controllers use will be based on the KubeBuilder v3 toolkit.
These are the baseline tools used to generate code and run integration tests, and should be
installed locally with Makefile targets:

- Kustomize
- Controller-gen
- Envtest

Shipwright controller projects should also standardize on [ko](https://github.com/google/ko) for
building golang-based container images.
Ko provides critical features such as reproducible builds and automatic generation of Software
Bills of Material (SBOMs).
Ko also simplifies the process of deploying golang containers on Kubernetes, which improves
contributor experience for testing.

Makefiles should also support a core set of commands, based on what is provided by Kubebuilder:

- `help` - print help for the make commands. This can be generated from comments in the Makefile.
- `manifests` - generate CRD manifests
- `generate` - generate go code (DeepCopy, etc.)
- `fmt` - run `go fmt`
- `vet` - run `go vet`
- `test` - run unit tests
- `test-integration` - run integration tests with Envtest
- `test-e2e` - run end to end tests against a full Kubernetes cluster
- `container-build` - build image with ko
- `container-push` - build and push the image
- `install` - install CRDs
- `uninstall` - uninstal CRDs
- `deploy` - deploy images with Ko
- `undeploy` - delete deployment

Make targets should also be provided to install build and test tooling:

- `kustomize`
- `controller-gen`
- `envtest`
- `ko`

Finally, every project should have a `release` target that automates the release process.

Projects should feel free to add additional make targets if necessary, but should take care
to avoid clutter.

### Standardize on ko

Projects should use ko to build container images and render image references into YAML manifests.
This can be done automatically with the `ko resolve` and `ko apply` commands.
The `deploy` and `release` targets should ensure that ko is used to render image manifests.

### Implementation Notes

In general, projects should implement restructuring in phases.
The following staging is recommended:

1. Update Makefile targets
2. Move API code to `api/`
3. Move controller code to `controller/`
4. Add KubeBuilder project configuration.

#### Operator

The operator repository utilizes operator-sdk, and therefore is already based on KubeBuilder.
Ko is already utilized to build the operator image.
Some changes are required to use ko-rendered image references in the operator bundle.

#### Image

Much of this repository would need to be rebuilt with KubBuilder libraries.
The most significant impact will be on the boilerplate/setup code.
The migration could be done in phases, on a per-controller basis.

Refactoring to Kubebuilder should be done sooner, rather than later, as this project is still in
its infancy.

#### Build

This new structure will have significant impact, since this repository is very active and has a lot
of infrastructure built for testing and development.
Build was initialized with an older version of operator-sdk, and much of that legacy format is
present in the structure today.
Thankfully, the codebase is built on top of the controller-runtime library and therefore is largely
compatible with Kubebuilder.
Any refactoring effort should be coordinated to occur at the start of a new release cycle, to
minimize the impact of other feature development.

Below is a proposed staging of changes:

1. Standardize Makefile targets. There are currently multiple targets for testing - these should be
   unified, and any variations should be enabled with variables/arguments.
   In addition, a significant portion of the release scripting is inlined with GitHub Actions,
   making it difficult to debug or lint.
   These should be moved to scripts in `hack/` and invoked with a `make release` target.
2. Move API code to `api/`, and generated client code to `client/`. We should ensure that CRDs and
   client code continue to be generated as expected.
   *Note: this will be a breaking change for golang consumers, like the cli*.
3. Move controller code to the `controller/` format. This should be limited to actual controller
   reconciliation functions. Existing business logic in the `pkg/` should be kept in place.
4. Move the controller-manager's entrypoint to `main.go` at the root of the project. This will
   change the container image name generated by `ko`.
5. Add the `PROJECT` file at the repository root, to declare the kubebuilder configuration.


### Test Plan

At each phase of a given repository's refactoring, existing CI tests in GitHub Actions should pass.
To the furthest extent possible, existing tests should not be disabled.

### Release Criteria

Code should always be in a "release ready" state during any project restructuring.

### Risks and Mitigations

Refactoring and restructuring code carries the risk that we introduce changes that break critical
functionality.
This is best mitigated by ensuring CI is always functional and the project is in a "release-ready"
state.
Adopting nightly releases in all projects can help ensure that releases can indeed happen.

## Drawbacks

Restructuring code is an inherently disruptive process.
Contributors working on code in parallel may find themselves needing to rebase and lose work in the
process of resolving merge conflicts.
It is therefore advised that any major restructure be closely coordinated with the release cycle.

## Alternatives

The alternative is to maintain the status quo - each controller project continues to adopt code
organically.
This inhibits contributor productivity, as each project will have its own structure and
idiosyncrasies.
Independent project structures also prevent us from creating common tooling for code generation,
deployment, and releases.

## Infrastructure Needed [optional]

None.

## Implementation History

- 2022-04-22: Initial draft

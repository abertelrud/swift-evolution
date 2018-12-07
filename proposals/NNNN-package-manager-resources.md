# Package Resources

* Proposal: [SE-NNNN](NNNN-package-manager-resources.md)
* Authors: [Anders Bertelrud](https://github.com/abertelrud)
* Review Manager: [TBD](https://github.com/)
* Status: **WIP**


## Introduction

Packages should be able to contain images, data files, or other resources that are needed at runtime.  This draft proposal describes SwiftPM support for specifying package resources, and introduces a consistent way of referring to them from the source code in the package.


## Motivation

Packages consist primarily of source code to be compiled and linked into executables.  Sometimes, however, the code needs additional resources that are expected to be available at runtime.  Such resources could include images, sounds, user interface specifications, and other typical runtime resources.  During the build, package resources might be copied verbatim into product, or might need to be processed in some way.

Resources are not always used by clients of the package;  one use of resources might include test fixtures needed only by unit tests in the package.  Such resources would not be incorporated into clients of the package, but would only be used when the package tests are run.

One of the fundamental principles behind SwiftPM is that packages should be as portable and client-agnostic as possible:  in particular, packages should make as few assumptions as possible about the details of how they will be incorporated into a particular client on a particular platform.

For example, a package might in one case be built as a dynamic library or framework that is embedded into an application bundle, and might in another case be statically linked into the client executable.  These differences might be due to the platform for which the package is being built, or might be a result of deployment packaging choices made by the client.

Packages should therefore be able to specify the resources that will be needed at runtime, and SwiftPM should provide a way for the code to access those resources at runtime without requiring the code to bake in assumptions about exactly where the resources will be.  This allows the code to be written in a way that is independent of how the resources will be made available at runtime by a particular type of build artifact on a particular platform.


## Scoping of resources

Resources are most commonly referenced by the code that needs them, so it seems natural to consider the code and resources to be part of the same target.  Any client that depends on the target's code also needs the associated resources to be available at runtime (though it usually doesn't access them directly, but rather through the code that needs them).

Furthermore, the processing of some types of resources might result in generated code, which will need to be linked into the same module as the other code in the target; it is therefore natural to consider the resources (and the generated code) as conceptually part of the same target.

Scoping resources by target also aligns naturally with how targets are currently built in other development environments that support resources:  in Xcode, for example, each target is built into a CFBundle (usually a framework or an application), and any resources associated with the target are copied into that bundle.  Since a bundle provides a namespace for the resources in it, scoping resources by target is a natural fit.


## Location of resources in a package

The Swift Package Manager uses file system conventions to determine which source files belong to which targets and tests, and it seems logical to do the same for resources.  Just as for sources, it is likely that there will eventually be a need for more precise control on a file-by-file basis in the package manifest.

If resources are considered to be additional target content, then the most natural location for them is inside the individual target subdirectories under `Sources`.  This could take the form of a special `Resources` subdirectory under each target directory, but it seems more natural to allow the resource files to be located alongside the source files with which they are most closely associated.  This would allow grouping of package resources in the same way as sources are grouped, by just creating a directory structure under the individual target subdirectories.  This approach is, for example, common in Xcode projects today.

One can view this as a broadening of the notion of what constitutes a "source file" beyond just source code that is compiled and linked into the executable part of the product;  the executable code is, after all, just one aspect of the built artifact.  This can in the future be extended to a variety of file-type-specific processing.

This raises the question of how to determine which files to treat as compilable source code and which ones to treat as resources.  One approach that seems fairly straightforward is to base this choice on the file's type, as determined either by its suffix or by some more sophisticated means (e.g. content analysis or other metadata).

SwiftPM already does this in a limited way, by treating files that have a `.c` suffix as C, those that have a `.swift` suffix as Swift, etc.  This proposal extends the set of default rules, so that common resource file types are treated as resources by default.  We will also want to extend the package manifest API to allow packages to override the processing rule for any given file type or file in the target.

This can be thought of as determining the "role" that each file plays in the target.  While it applies to resources vs compilable source code here, this could in the future extend to public vs private headers etc.

```swift
Package
 ├ Sources
 │  ├ MyInfoPanel
 │  │  ├ InfoPanelController.swift
 │  │  ├ Background.jpg
 │  │  ├ LicenseAgreement.html
 │  │  └ FileThatWouldUsuallyNotBeConsideredResource.dat
 │  └ IconWidget
 │     ├ IconDataSource.swift
 │     ├ IconView.swift
 │     └ Border.png
 └ Tests
    ├ MyInfoPanelTests
    │  ├ InfoPanelViewTests.swift
    │  └ TestInput.json
    └ MyTestFixtures
       ├ Something.xml
       └ SomethingElse.xml
```

In this case, the JPG, PNG, JSON, XML, and HTML files are treated as resources because that's the default role for those file types (the default mapping of file suffixes to roles would need to be defined).

This is also an example of a codeless module, `MyTestFixtures`.

Package authors will need some way of controlling how resource files are processed.  Adding a way to define custom rules is beyond the scope of this proposal, but we should consider at least allowing the package manifest to explicitly override the default role for a file or set of files.

One approach would be to allow the package manifest to provide a mapping from file type patterns to roles:

```
let package = Package(
    name: "MyInfoPanel",
    targets: [
        .target(
            name: "MyInfoPanel",
            dependencies: ["IconWidget"],
            roles: [
              "*.dat": .resource
            ]
        ),
        .target(
            name: "IconWidget",
            exclude: ["Border.png"]
        ),
        .testTarget(
            name: "MyInfoPanelTests",
            dependencies: ["MyInfoPanel", "MyTextFixtures"]
        ),
        .testTarget(
            name: "MyTestFixtures",
        ),
    ]
)
```

There are a lot of things left open in this example, but the idea would be to allow a mapping of subpaths (which could contain glob-patterns) to an enum value that tells the Package Manager what to do with files that match the pattern.  The specific processing done for each role would depend on the platform and possibly other factors.

There is a question of what to do if multiple patterns match, and there is also the question of what exactly the set of possible enums should be.  A further open question would be how to handle files for which no role can be determined.

One observation about this approach is that the existing `exclude:` parameter can be considered to be a specialized case of mapping certain filepath patterns to a hypothetical `.ignore` role.


## Referencing resources from code

Historically, macOS and iOS resources have been accessed primarily by name, using untyped Bundle APIs such as `path(forResource:ofType:)` that have assumed that each resource is stored as a separate file.  Missing resources or mismatched types (e.g. trying to load a font as an image) have historically resulted in runtime errors.

In the long run, we would like to do better.  Because SwiftPM knows the names and types of resource files, it should be able provide type-safe access to those resources by, for example, generating the right declarations that the package code could then reference.  Missing or mismatched resources would produce build-time errors, leading to earlier detection of problems.

This would also serve to separate the reference to a resource from the details about how that resource is packaged; instead of getting the path to an image, for example, and then loading that image using a separate API (which assumes that the image is stored in a separate file on disk), the image accessor could do whatever is needed to load the image based on the platform and client packaging choices that were made at build time, so that the package code doesn't have to be aware of such details.

In the short term, however, we want to keep things simple and allow existing code to work without major modifications.  Therefore, the short-term approach suggested by this draft proposal is to stay with Bundle APIs for the actual resource access, and to provide a very simple way for code in a package to access the bundle associated with that code.  A follow-on proposal would introduce typed resource references as additional functionality.

Since resources are scoped to modules, this could be done by having SwiftPM generate an internal static extension on Bundle for each module it compiles:

```
extension Bundle {
    internal static var moduleResources: Bundle { get }
}
```

Because this is an internal static property, it would be visible only to code within the same module, and the implementations for each module would not interfere with each other.  The implementation generated by SwiftPM would use information about the layout of the built product to instantiate and the cache the bundle containing the resources.

The source code of the module could reference it like this:

```
let path = Bundle.moduleResources.path(forResource:"MyInfo", ofType:"plist")
```

The first access to the `moduleResources` property would cause the bundle to be instantiated.  Modules without any resources would not have resource bundles, and for such modules, no declaration would be created.

Because the name of the accessor would always be `moduleResources`, code could be moved between modules (as long as the resources were moved as well) without requiring any source code changes.

This would be an improvement over the status quo in Cocoa code, which involves either `Bundle.init(for:)` or `Bundle.init(identifier:)` --- the former is problematic because it assumes that the resources will necessarily be in the same bundle as the code (which won't be true for staticly linked code), and the latter is problematic because it hardcodes the identifier name (and also doesn't automatically cause the bundle to be loaded if it hasn't been loaded already).


## Accessing resources at runtime

The previous section discussed how the resource access conventions make the code immune to how exactly the resources are packaged into bundles.  It then becomes the job of the build system that's producing the artifacts to provide the necessary glue that lets the resources be found at runtime.

In a simplistic implementation that always produced a dynamic framework for each module, the resources could be put into the framework and the implementation of the accessor would simply return the module having the identifier of the framework built from the module (the build system is in charge of compiling the module code as well as copying the resources, so it has all the necessary information).

A more sophisticated implementation that statically links library code into clients could choose to instead produce a codeless "asset pack" bundle for each module that has resources, and would in that case have to generate code to load the bundle the first time it is requested.  This code would be platform-dependent, but build systems are already making many platform-specific decisions;  it's reasonable for the generated implementation of the `moduleResources` property to be different for different platforms and artifact layouts as long as the source code of the package can remain unchanged across the various platforms.


## Alternatives considersd

#### Resource scoping

Rather than broadening the notion of "source file" to include resources, an alternate approach would be to introduce a new kind of "resource target".  Such a target would contain only resources, and no compilable source code.

While this might seem natural for a certain kind of resource-only target (e.g. a set of assets in a game), it would make the general case of a mixture of source code and resources be more complicated to work with, and would interfere with use of directory structure as functional grouping.


## Other considerations

#### Impact on existing packages

The proposed changes should not cause any existing packages to fail to build, but there may be subtle changes in semantics.

In particular, the Package Manager currently ignores any files with unknown types in the source directories.  With the proposed changes, those files would start to be included in the built product, which may have unexpected consequences.

Given this, it may make sense to tie the new semantics to a new version of the package manifest, since semantics are a part of the API contract as much as explicit functions and properties are.  Once we decide on how source file processing rules can be overridden, there will likely be explicit package manifest API changes as well, so the package manifest version may be a fairly natural way to opt into resource processing.

#### Localization

Basic support for localization should at least be discussed in this proposal, and should preferably be convention-based, like the rest of SwiftPM.  Other workflows surrounding localization, such as whether additional localizations can be incorporated from outside the package during a build, are beyond the scope of the present proposal.

---

This draft proposal is a still being written.  Due to the large number of open questions, it is not yet ready for review.

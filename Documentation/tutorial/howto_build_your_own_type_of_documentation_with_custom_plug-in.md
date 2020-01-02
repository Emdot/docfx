How-to: Build your own type of documentation with a custom plug-in
====================================

In this topic we will create a plug-in to convert some simple [Rich Text Format](https://en.wikipedia.org/wiki/Rich_Text_Format) files to HTML documents.

Goal and limitation
-------------------
1.  In scope:
    1.  Our input will be a set of RTF files with `.rtf` as the file extension name.
    2.  The RTF files will be built as HTML document.
2.  Out of scope:
    1.  Pictures or other objects in RTF files.
    2.  Hyperlinks in RTF files. (See the [advanced tutorial](advanced_support_hyperlink.md) for how to support hyperlinks in a custom plugin.)
    3.  Metadata and title.

Preparation
-----------
1.  Create a new C# class library project in Visual Studio, targetting .NET Framework 4.7.2.

2.  Add nuget packages:  
    * `System.Collections.Immutable` with version 1.3.1
    * `Microsoft.Composition` with version 1.0.31

3.  Add `Microsoft.DocAsCode.Plugins` and `Microsoft.DocAsCode.Common`
    If you're building DocFX from source code, add a reference to the project;
    otherwise, add the nuget packages that have the same version as DocFX.

4.  Add framework assembly references:
    `PresentationCore`, `PresentationFramework`, `WindowsBase`. (This step is optional in Visual Studio 2017 or above)

5.  Add a project for converting RTF to HTML:  
    Clone project [MarkupConverter](https://github.com/mmanela/MarkupConverter), and reference it.

6.  Copy the code file `{C#,C++,F#,VB}\ParallelExtensionsExtras\TaskSchedulers\StaTaskScheduler.cs` from [ParExtSamples](https://code.msdn.microsoft.com/ParExtSamples)

Create a document processor
---------------------------

### Responsibility of the document processor

* Declare which files can be handled.
* Load from a file to the object model.
* Provide build steps.
* Report the document's document type, file links, and xref links.
* Update references.

### Create our RtfDocumentProcessor

1. Create a new class (RtfDocumentProcessor.cs) with the following code:
   ```csharp
   [Export(typeof(IDocumentProcessor))]
   public class RtfDocumentProcessor : IDocumentProcessor
   {
       // todo : implements IDocumentProcessor.
   }
   ```

2. Declare that we can handle the `.rtf` file:

   [!Code-csharp[GetProcessingPriority](../codesnippet/Rtf/RtfDocumentProcessor.cs?name=GetProcessingPriority)]

   Here we declare this processor can handle any `.rtf` file in the article category with normal priority.
   When two or more processors compete for the same file, DocFX will give it to the higher priority one.
   *Unexpected*: two or more processor declare for the same file with same priority.

3. Load our RTF file by reading all its text:
   [!Code-csharp[Load](../codesnippet/Rtf/RtfDocumentProcessor.cs?name=Load)]

   We use `Dictionary<string, object>` as the data model, similar to how [ConceptualDocumentProcessor](https://github.com/dotnet/docfx/blob/dev/src/Microsoft.DocAsCode.EntityModel/Plugins/ConceptualDocumentProcessor.cs) stores the content of markdown files.

4. Implement a `Save` method as follows:
   [!Code-csharp[Save](../codesnippet/Rtf/RtfDocumentProcessor.cs?name=Save)]

5. The `BuildSteps` property can provide several build steps for the model. We suggest implementing this in the following manner:
   [!Code-csharp[BuildSteps](../codesnippet/Rtf/RtfDocumentProcessor.cs?name=BuildSteps)]

6. The `Name` property is used for display in the log, so give it any constant string you like.  
   e.g.:  
   [!Code-csharp[Name](../codesnippet/Rtf/RtfDocumentProcessor.cs?name=Name)]

7. Since we aren't supporting hyperlinks, leave the `UpdateHref` method empty.
   [!Code-csharp[UpdateHref](../codesnippet/Rtf/RtfDocumentProcessor.cs?name=UpdateHref)]

View the final [RtfDocumentProcessor.cs](../codesnippet/Rtf/RtfDocumentProcessor.cs)


Create a document build step
----------------------------

### Understanding the processor's build process ###

A processor has three phases that are relevant here: pre-build, build, and post-build. Each phase begins when all previous phases have ended.
* The pre-build phase is intended for:
  * modifying the list of documents that will be built; e.g., to exclude documents that match some rule,
  * generating artifacts for use by later phases; e.g., to let a `Build` call use data that came from a document other than the one it's processing.
* The build phase is intended for:
  * transforming the in-memory representation of individual documents; e.g., from RTF to HTML,
  * generating artifacts to be consumed by later pharses; e.g., building index entries for an index-building step in the post-build phase.
* The post-build phase is intended for:
  * generating artifacts to be used by later processing --- especially artifacts that aggregate over the files

A processor contains a set of build steps. Whenever the sequence of build steps matters, steps with lower `BuildOrder` values run before steps with higher `BuildOrder` values. Every build step implements methods `Prebuild`, `Build`, and `Postbuild`, which correlate to the build phases.

The pre-build phase is a pipeline of calls to the steps's `Prebuild` methods: the output of one step's `Prebuild` is provided as an input to the next step's `Prebuild`.

The build phase begins after the pre-build phase completes. In the build phase, each document is processed in a pipeline of `Build` calls. Since each pipeline is independent of the others, pipelines run in parallel with each other.

The post-build phase begins when all of the build phase's pipelines have completed. In this phase, each step's `Postbuild' is invoked in sequence.

Example: Document processor *X* has two steps: Foo (with `BuildOrder`=1), Bar (with `BuildOrder`=2). When *X* is called for documents [A, B, C], the invoke order is as follows:

    sequence:
    1. Foo.Prebuild([A, B, C]) returns [A1, B1, C1]
    2. Bar.Prebuild([A1, B1, C1]) returns [A2, B2, C2]
    3. parallel:
       * sequence: 
          1. Foo.Build(A2) returns A3
          2. Bar.Build(A3) returns A4
       * sequence: 
          1. Foo.Build(B2) returns B3
          2. Bar.Build(B3) returns B4
       * sequence: 
          1. Foo.Build(C2) returns C3
          2. Bar.Build(C3) returns C4
    4. Foo.Postbuild([A4, B4, C4])
    5. Bar.Postbuild([A4, B4, C4])

### Create our RtfBuildStep:

1. Create a new class (RtfBuildStep.cs), and declare it is a build step for `RtfDocumentProcessor`:
   ```csharp
   [Export(nameof(RtfDocumentProcessor), typeof(IDocumentBuildStep))]
   public class RtfBuildStep : IDocumentBuildStep
   {
       // todo : implements IDocumentBuildStep.
   }
   ```

2. In the `Build` method, convert RTF to HTML:
   [!Code-csharp[Build](../codesnippet/Rtf/RtfBuildStep.cs?name=build)]

3. Implement other methods:
   [!Code-csharp[Others](../codesnippet/Rtf/RtfBuildStep.cs?name=Others)]

View the final [RtfBuildStep.cs](../codesnippet/Rtf/RtfBuildStep.cs)


Enable the plug-in
------------------
1. Build the project.
2. Install the result:
    * To install globally, copy the output DLLs to the DocFx Plugins folder, which is located in the same folder as DocFX.exe.
    * To install in a particular *DocFx* template, create a folder named `Plugins` inside your template's folder, copy the output DLLs there, and run `DocFX build -t {yourtemplate}`.
3. Connect it to docfx.json:
   * If you installed the plugin globally, add its name to docfx.json >> `build` >> `postProcessors: []`.
   * If you installed the plugin in a template, add its name to docfx.json >> `build` >> `template: []`.

Build the documents
--------------
1. Run command `DocFX init` and set the source article to `**.rtf`.
2. Run command `DocFX build`.

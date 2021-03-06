Shtirlitz
=========

Framework for gathering support data on the client machine for .NET applications.

What it produces
----------------

`Shtirlitz` creates one archive file that contains the data generated by multiple _reporters_ and then supplies its path to one or more _senders_ for processing.

Each reporter (implementation of the `IReporter` interface) is free to create as many files as it needs using arbitrary formats, but the common convention is as follows:

* All reporters should produce information that can be read using standard programs, such as text and HTML files, images in widely adopted formats such as PNG or JPG and so on. Proprietary binary formats that need special software to view them are discouraged.
* If a reporter produces one file, it should add this file into the root of the archive. The name of the added file should be connected to the name of the reporter, or otherwise state the nature of the information in that file.
* If a reporter produces many files (which can happen if either one reporter object writes many files, or the addition of multiple reporter objects into one `Shtirlitz` is allowed), the reporter should put the files into the directory which name is also connected to the reporter as in the previous item. To implement this kind of a reporter, you can use the `MultiFileReporter` as a parent class, which will create the sub-directory for you.

Usage
-----

To collect and send support data in background:

```csharp
// specify reporters and senders
List<IReporter> reporters = new List<IReporter> { ... };
List<ISender> senders = new List<ISender> { ..., new ArchiveRemover() /* removes archive after the senders are done with it */ };

// create Shtirlitz object to run the report
IShtirlitz shtirlitz = new Shtirlitz(reporters, new DotNetZipArchiver(), senders);

// start the report generation (asynchronous)
shtirlitz.Start(CancellationToken.None);
```

If you need to run report generation synchronously, get the `Task` object from the `Start` method and wait for it to finish:

```csharp
Task reportTask = shtirlitz.Start(CancellationToken.None);

// wait for the operation to finish
reportTask.Wait();
```

Should you need to display the progress information, just subscribe to the `GenerationProgress` event and you'll receive both numeric progress which is between `0.0` and `1.0` and the name of the stage that's currently running:

```csharp
shtirlitz.GenerationProgress += (sender, args) => Console.Write("\r{0:0}% finished, Current step: {1}", args.Progress * 100, args.StageName);
// there's also an args.StageType property that contains the type of the stage
```

If you need to manually process the file that was generated or do something once the operation is finished (close the report UI dialog for example), subscribe to events `ReportGenerated` and `ReportCanceled`. The former happens when the report generation has finished successfully and provides the name of the archive (`args.Filename`, if you want to process the file, be sure to remove `ArchiveRemover` sender from the sender list so that archive stays on disk), while the latter happens when the report generation was canceled or has crashed (use `args.HasFailed` to distinguish between the two).

```csharp
shtirlitz.ReportGenerated += (sender, args) => Console.WriteLine("Report was successfully generated, see the file {0}", args.Filename);
shtirlitz.ReportCanceled += (sender, args) => Console.WriteLine(args.HasFailed ? "Report generation has failed" : "Report generation was canceled");
```

To allow for the cancellation of the report generation, supply the valid `CancellationToken` to `Shtirlitz` when calling the `Start` method:

```csharp
CancellationTokenSource cancellationSource = new CancellationTokenSource();

// start the report generation (asynchronous)
shtirlitz.Start(cancellationSource.Token);
```

and later you can abort the report generation by simply calling `cancellationSource.Cancel()`.

Extensibility
-------------

To change or extend the functionality of the library, you can implement your custom `IReporter`s, `IArchiver`s and `ISender`s and then supply them to the `Shtirlitz` when it is created.

Typical `Shtirlitz` workflow looks like this:

* First, the temporary directory for the report is created.
* Then all the `IReporter`s are executed with the path to the directory supplied to them. They're supposed to write all the info they gather into this directory or its sub-directories.
* Once that is done, `IArchiver.Archive` is called which combines all the data from the temporary directory into one file, the name of which is pre-determined.
* Then the report temporary directory is deleted, because there is no need in those raw files. (Unless, of course, you specified to skip this intermediate cleanup by setting the `cleanup` parameter in the `IShtirlitz.Start` method to `false`.)
* Then the file name of the created archive is passed to a series of `ISender`s, each of which is supposed to send this file to the developers or otherwise process the file. There is one special sender called `ArchiveRemover` which removes the archive, you should put it at the end of the senders list to avoid leaving the report archive in the user's temporary directory.

If at any stage of the report generation, an exception occurs, `Shtirlitz` removes all traces of the info (both the temporary directory and the archive file) and raises its `ReportCanceled` event, so you need not to do any cleanup in this case.

An interface of any stage has one main method which receives, among other things, the `CancellationToken` and the `SimpleProgressCallback`. Stage implementation should

* Periodically execute `cancellationToken.ThrowIfCancellationRequested()` to allow for the cancellation of the report generation in the middle of the stage. `Shtirlitz` will, again, handle the thrown exception and perform the needed cleanup/event raising.
* Report its progress by executing the supplied progress callback specifying the progress as a number between `0.0` and `1.0`. To handle the `null`-ness of the progress callback, the `ProgressCallbackUtil.TryInvoke` extension method is implemented: just write `progressCallback.TryInvoke(/* progress calculation formula */);` and it'll complete successfully even if `progressCallback` is null.

License
-------

Copyright 2013, 2014 by Slava Kolobaev. This work is made available under the terms of the Creative Commons Attribution 3.0 license, http://creativecommons.org/licenses/by/3.0/

---
title: "What's the Target Model? (And Why?)"
date: 2022-10-03T09:54:07-07:00
draft: false
---

How do you teach an old dog new tricks? That's the topic of today's post. "WinDbg" is short for "Windows Debugger", but lately that name seems a bit odd since the WinDbg of today knows about a lot more than just Windows. WinDbg now supports Linux and MacOS crash dump targets, as well as few things that are a bit of a hybrid, like [Open Enclave debugging](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/open-enclave-debugging).

The core "debug engine" behind WinDbg is called "DbgEng", and it's been a Windows-centric debugging engine for decades. Adapting it to work on other platforms was no small task. You might be wondering why we even adapted DbgEng to other platforms rather than taking an existing debugger that worked for those platforms, like GDB. There were a few reasons, but the most compelling reason was that it let us take advantage of the infrastructure Microsoft has built around the DbgEng platform. This has also been built up over many years and includes things such as symbol servers, source indexing, debugging extensions, scripting, debugging UI, and automated crash analysis. It also let us take advantage of the existing Windows debugging capabilities that DbgEng had for hybrid debugging scenarios.

In adapting DbgEng to other platforms, we knew that we needed to have a flexible system that wouldn't just be to support Linux, but also be able to support any new system that needed to be debugged, whether that meant an entirely new hardware platform, operating system, symbol format, or crash dump format. To do that, we leaned into the strength of DbgEng, which is extensibility. While the existing extensibility in the debugger focused on creating new debugger commands, scripts, or visualizations using the [IDebugClient interfaces](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nn-dbgeng-idebugclient) or the [debugger data model](https://www.timdbg.com/posts/whats-the-data-model/), this new extensibility mechanism focuses on enabling new types of debugging targets and enhancing core debugging concepts. We call it the "Target Composition Model" (sometimes just abbreviated as Target Model or Target Composition). The "Composition" part of the name is important, because it speaks to the core idea we wanted of having an independent set of components that can be composed in different ways to support debugging a target. For instance, we could write a component for [DWARF debug info](https://en.wikipedia.org/wiki/DWARF) to support Linux debugging, and use that same component to debug a Windows target that uses DWARF debug information. 

The Target Model is available for anyone to use and extend. To get started, you can use the [Microsoft.Debugging.TargetModel.SDK](https://www.nuget.org/packages/Microsoft.Debugging.TargetModel.SDK) NuGet package, which provides the core headers and static libs needed to implement a target model component, including DbgServices.h that has the most important interface definitions. To see an in depth explanation of how to create a target model extension, take a look at the [TextDump sample on our GitHub repo](https://github.com/microsoft/WinDbg-Samples/tree/master/TargetComposition/TextDump). If you're looking for an overview of what the Target Model is and what it can do, read on!

# Overview

A Target Composition is a collection of components that each provide one or more services. Each service has an associated GUID, such as DEBUG_SERVICE_PHYSICAL_MEMORY, and an interface or set of interfaces that can be implemented by that service, such as ISvcMemoryAccess. While interfaces and service IDs are closely related, there is not a 1:1 relationship. For instance, ISvcMemoryAccess can also be implemented by the DEBUG_SERVICE_VIRTUAL_MEMORY service. These components are managed by the Service Manager, which is accessed through the IDebugServiceManager interface. This interface is used for both locating and registering services. These services give access to every aspect of a target such as reading memory, enumerating modules, or unwinding a stack.

# Creating new targets

When talking about target model extensions, there are two basic types. The first type implements all of the core services needed for a target and starts as a "blank slate", implementing services such as the virtual memory service. In an extension that implements a new dump format, the source of this data is a file and the extension is responsible for providing a virtual memory service that interprets the data in the file and presents it to the other components through the ISvcMemoryAccess interface. The extension is also responsible for presenting any other information in the file through the appropriate target model services. The set of services implemented will determine the capabilities of a debugging session for that target. Any commands that rely on physical memory to be available will not work for a target that does not provide a DEBUG_SERVICE_PHYSICAL_MEMORY service, for instance.

The [TextDump example on GitHub](https://github.com/microsoft/WinDbg-Samples/tree/master/TargetComposition/TextDump) is this first type of extension. It provides services for process enumeration, thread enumeration, module enumeration, virtual memory, and stack unwind. The data backing these services come from a text file representation. A target model extension doesn't have to provide every service from scratch, and can be built on top of existing components. The TextDump sample uses the DEBUG_COMPONENTSVC_PEIMAGE_IMAGEPROVIDER component, which knows how to "map" the memory of an image into a view of virtual memory. These components can also come as "component aggregates" where a set of related services can be brought in together. For windows kernel debugging, you can use DEBUG_COMPONENTAGGREGATE_OS_KERNEL_WINDOWS to bring in the relevant components. These can be added through their respective GUIDs (defined in DbgServices.h) through the IDebugTargetComposition interface (which can be retrieved from IDebugTargetCompositionBridge::GetCompositionManager).

(Hint: To see all of the existing components available, search for DEBUG_COMPONENT in DbgServices.h)

For an extension like this, the debugger needs to know when it should be used. This is done through an XML manifest file that describes the conditions for which an extension should be loaded. The TextDump sample uses a [manifest file](https://github.com/microsoft/WinDbg-Samples/blob/master/TargetComposition/TextDump/TextDump_GalleryManifest.xml) that describes the type of target that is supported, which in this case is a "text dump file" registered for the ".txt" extension:

{{< highlight xml >}}
<?xml version="1.0" encoding="utf-8"?>
<ExtensionPackages Version="1.0.0.0" Compression="none">
  <ExtensionPackage>
    <Name>TextDumpCompositionPackage</Name>
    <Version>1.0.0.0</Version>
    <MinDebuggerSupported>10.0.17074.1002</MinDebuggerSupported>
    <Components>
        <BinaryComponent Name="TextDumpComposition" Type="Engine">
            <LoadTriggers>
                <TriggerSet>
                    <IdentifyTargetTrigger FileExtension="txt" />
                </TriggerSet>
            </LoadTriggers>
            <Files>
                <File Architecture="amd64" Module="amd64\TextDumpComposition.dll" FilePathKind="RepositoryRelative"/>
            </Files>
        </BinaryComponent>
    </Components>
  </ExtensionPackage>
</ExtensionPackages>
{{< / highlight >}}

There are a number of other "load triggers" that can be used in a manifest. They are described in more depth in the [documentation for the TextDump sample](https://github.com/microsoft/WinDbg-Samples/blob/master/TargetComposition/TextDump/Readme.txt), but a few that might be interesting are the "AlwaysTrigger" which causes the extension to always be loaded, the "ModuleTrigger" which causes the extension to be loaded when a specified module is loaded in a target, and the "ExceptionTrigger" which can load an extension in response to an exception.

# Extending an existing target

While the target extensions I've talked about so far are "big" components that provide the basic services like the memory of a target, there is a second type of extension that do not start from a "blank slate" but instead provides functionality on top of an existing target. For this type of extension, you could use some of the triggers previous mentioned such as using the ModuleTrigger to load a target model extension that provides additional information about a specific module (like a scripting engine). It's also possible to interact with the target composition from an IDebugClient-based extension, so you could write an extension command called "!loadMyTarget" that loads and registers your target model extension.

The target model interfaces are designed to be completely independent from the IDebugClient interfaces so that they could be used outside of DbgEng in some other debugger. To "bridge" from the IDebugClient style extensions to the target model extensions, we have the IDebugTargetCompositionBridge interface, which can be queried (with [QueryInterface](https://learn.microsoft.com/en-us/windows/win32/api/unknwn/nf-unknwn-iunknown-queryinterface(refiid_void))) from an IDebugClient interface. From this interface, you can register for a specific file extension:

{{< highlight cpp >}}
STDMETHOD(RegisterFileActivatorForExtension)(
    THIS_
    _In_ PCWSTR pwszFileExtension,
    _In_ IDebugTargetCompositionFileActivator *pFileActivator
    ) PURE;
{{< / highlight >}}

You can also directly interact with the "Service Manager", which can be found by calling IDebugTargetCompositionBridge::GetServiceManager:

{{< highlight cpp >}}
// GetServiceManager():
//
// Gets the service manager for a particular target as given by its system id.
//
STDMETHOD(GetServiceManager)(
    THIS_
    _In_ ULONG systemId,
    _Out_ IDebugServiceManager **ppServiceManager
    ) PURE;
{{< / highlight >}}

Using the service manager, you can query for existing services as well as register new services. While it is possible to completely replace a specific service, it's often more useful to register a service that provides additional information beyond what the existing service provides. For instance, you can implement a stack walk service that understands the stack frames of a scripting language, and integrate those frames into a larger stack walk. To cooperate with an existing native stack walk service, you can use a concept called "service aggregations". A service aggregation allows multiple components to participate as a single logical service. How a service aggregation works depends on the service. A module enumeration service aggregation (DEBUG_SERVICE_MODULE_ENUMERATOR/ISvcModuleEnumeration) will simply return the union of all of the modules enumerated by the child services. To see how this works, see the documentation on IDebugServiceManager5::AggregateService.

There are lots of different ways that you can change and extend targets through these interfaces. Some of the things you can do include:

* Support new symbol file formats (like DWARF and PDB)
* Support new image file formats (like ELF and PE)
* Extend a type system (like that of a script environment)
* Support a new type of CPU architecture
* Provide information about modules loaded outside of the OS loader 
* Provide custom name demangling

These interfaces are already being used by components of the debugger to support scenarios we'd never been able to support before. Our plan is to continue refactoring existing functionality from the monolithic debug engine into components of the target composition model. As we do this, we are going to continue to enable new scenarios, as well as making the platform more extensible and flexible.

This post is clearly not exhaustive, but hopefully it gives you the big picture of what is possible. You'll find more in-depth information in the readme for the [TextDump example on GitHub](https://github.com/microsoft/WinDbg-Samples/blob/master/TargetComposition/TextDump/Readme.txt) and in DbgServices.h in the [Microsoft.Debugging.TargetModel.SDK nuget package](https://www.nuget.org/packages/Microsoft.Debugging.TargetModel.SDK).

My work on this has been minimal, so I can't take credit for any of the amazing things that you can do now with the Target Composition Model (and there could be lots of mistakes in this post). The real expert on this who has implemented most of the functionality is [Bill Messmer](https://twitter.com/wmessmer). Did you find this interesting, or have any questions? Let us know on [Twitter](https://twitter.com/timmisiak/status/1576986941685264384)!

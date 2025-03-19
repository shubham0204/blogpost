---
title: The Apple Computing Stack
tags:
  - "#programming"
---
I recently bought a M4 Mac Mini (16GB/512GB) which is basically my first Apple product coming from the Windows/Android world. Fascinated by the machine's capabilities, I started touching iOS app development with XCode and thought about researching how Apple operating systems work under the hood. 

Spending 7 years in Android development, the 'Android' stack right from the hardware to Jetpack Compose was clear to me. This article is a compilation of my reading and research on how Apple operating systems work and some other core components that helped me understand the ecosystem better.

By definition, I should be starting from Apple's computer hardware, but I will skip it as hardware details remain scarce and largely hidden from an application programmer's perspective.

> [!WARNING]
> I am not an experienced Apple developer, hence the article should be read as a 'glossary' and not as a descriptive ground source of truth. Although, text derived in this article is validated from the linked sources (mostly Wikipedia articles and old Apple docs)

---
## Darwin and XNU
![Darwin Architecture](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f2/Diagram_of_Mac_OS_X_architecture.svg/1024px-Diagram_of_Mac_OS_X_architecture.svg.png)
[Darwin](https://en.wikipedia.org/wiki/Darwin_(operating_system)) is the core operating system behind all of Apple's OSes (iOS, macOS, watchOS, iPadOS etc.) and based on the [XNU](https://en.wikipedia.org/wiki/XNU) kernel.

The kernel forms the core of any operating system handling all major functions and interactions with the hardware. The XNU kernel is a micro-kernel, meaning different services are maintained as different components of the kernel. It is based on NeXTSTEP, FreeBSD and other BSD based applications.

Darwin also has its root in the Unix operating system through its BSD lineage, analogous to how Android has its lineage through the Linux kernel. Thus, macOS, does not seem too different for programmers coming from Linux-based OSes working in the terminal. 

> [!INFO]
> The Android OS is also open-source under the AOSP and uses a modified version of the Linux kernel.
> 
## Changes in ISA
Apple computers have witnessed three ISA migrations since their inception, starting from,
1. Motorola 68000 to PowerPC in 1994
2. [PowerPC to Intel's x86](https://en.wikipedia.org/wiki/Mac_transition_to_Intel_processors) in 2006 with Mac OS X
3. [Intel's x86 to ARM 64](https://en.wikipedia.org/wiki/Mac_transition_to_Apple_silicon) in 2020 with macOS Big Sur
## Universal and Universal 2 binaries
[Universal](https://en.wikipedia.org/wiki/Universal_binary) is a multi-architecture binary format for macOS that contains compiled application code for PowerPC and Intel's x86 ISA to ease the migration from the former to the latter. Universal binaries can execute on pre Mac OS X (PowerPC-based) computers and on the latter x86-based computers without having users to struggle with applications designed for both ISA(s).

Compiled object code for two ISA(s) within the Universal binary do share common resources, thus reducing the size of the overall binary (lesser than the sum of two individual binaries).

Universal 2, similar to Universal, contains compiled object code for Intel's x86 architecture and the new Apple Arm architecture Mac computers.

> [!INFO]
> Android uses the [APK](https://en.wikipedia.org/wiki/Apk_(file_format)) format to distribute applications and their required resources
## Mach-O object file format
[Mach-O](https://en.wikipedia.org/wiki/Mach-O) is a binary executable format for programs running on Apple computers. It is comparable to [ELFs](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format#:~:text=In%20computing%2C%20the%20Executable%20and,shared%20libraries%2C%20and%20core%20dumps.) for Linux and [PEs](https://en.wikipedia.org/wiki/Portable_Executable) for Windows. As it is only a format, a Mach-O object file can contain ARM, x86 or PowerPC code.

The format is described extensively in [this](https://alexdremov.me/mystery-of-mach-o-object-file-builders/) blog.

> [!INFO]
> Being a Linux descendant, Android uses the ELF format for its object-code
## Rosetta and Rosetta2

![](https://cdn0.tnwcdn.com/wp-content/blogs.dir/1/files/2011/10/apple_rosetta.jpg)

By definition, [Rosetta](https://en.wikipedia.org/wiki/Rosetta_(software)) is a dynamic binary translator that acts as an interoperability layer for an executable binary written in a non-native ISA on a Mac computer. Simply put, Rosetta is a translator that transforms executables compiled for the [PowerPC ISA to the Intel's x86 ISA](https://en.wikipedia.org/wiki/Mac_transition_to_Intel_processors) for execution. It allowed Mac OS X, Apple's first Intel based Mac computer, to execute applications compiled for PowerPC ISA which was found on all Macs prior to OS X.

Rosetta 2 is a similar layer that assisted [migration from Intel's x86 ISA to Apple Silicon](https://en.wikipedia.org/wiki/Mac_transition_to_Apple_silicon) based on the Arm architecture starting from macOS Big Sur in 2020.

> **From [Apple StackExchange](https://apple.stackexchange.com/questions/407731/how-does-rosetta-2-work),**
> 
> Rosetta 2 works by doing an ahead-of-time (AOT) translation of the Intel code to corresponding ARM code. It is able to do this efficiently and easily mainly because the M1 CPU contains a special instruction that switches the memory-ordering model observed by the CPU for that thread into a model equivalent to the Intel x86 model (TSO - total store order). This has to do with how programs can expect memory consistency to work when having multiple processors (i.e. cores in this case).
## Cocoa
As from the [docs](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CocoaFundamentals/WhatIsCocoa/WhatIsCocoa.html#//apple_ref/doc/uid/TP40002974-CH3-SW16),
> Cocoa is a set of object-oriented frameworks that provides a runtime environment for applications running in OS X and iOS.

Cocoa, apart from providing 'runtime' components in OS X and iOS, it also includes 'development' frameworks. These frameworks are written in Objective-C and ease application development.

For OS X, Cocoa mainly packages its classes into two core frameworks,
1. [AppKit](https://developer.apple.com/documentation/appkit): Defines UI, interaction and drawing
2. [Foundation](https://developer.apple.com/documentation/foundation): Provides collections, OS services and other primitives

For iOS, similar to OS X, the two core frameworks are,
1. [UIKit](https://developer.apple.com/documentation/coredata): Defines UI, interaction and drawing for iOS apps
2. [Foundation](https://developer.apple.com/documentation/foundation)

> [!INFO]
> Cocoa can be compared with the Android app development SDK and Jetpack Compose. Just like UIKit, Android's UI system based on XMLs/Views is also imperative in nature

The [Core Data](https://developer.apple.com/documentation/coredata) package that manages persistent storage for both iOS and OS X is common in Cocoa for both platforms.

```objective-c
#include <Cocoa/Cocoa.h>

@interface Window : NSWindow {
  NSTextField* label;
}
- (instancetype)init;
- (BOOL)windowShouldClose:(id)sender;
@end

@implementation Window
- (instancetype)init {
  label = [[[NSTextField alloc] initWithFrame:NSMakeRect(5, 100, 290, 100)] autorelease];
  [label setStringValue:@"Hello, World!"];
  [label setBezeled:NO];
  [label setDrawsBackground:NO];
  [label setEditable:NO];
  [label setSelectable:NO];
  [label setTextColor:[NSColor colorWithSRGBRed:0.0 green:0.5 blue:0.0 alpha:1.0]];
  [label setFont:[[NSFontManager sharedFontManager] convertFont:[[NSFontManager sharedFontManager] convertFont:[NSFont fontWithName:[[label font] fontName] size:45] toHaveTrait:NSFontBoldTrait] toHaveTrait:NSFontItalicTrait]];

  [super initWithContentRect:NSMakeRect(0, 0, 300, 300) styleMask:NSWindowStyleMaskTitled | NSWindowStyleMaskClosable | NSWindowStyleMaskMiniaturizable | NSWindowStyleMaskResizable backing:NSBackingStoreBuffered defer:NO];
  [self setTitle:@"Hello world (label)"];
  [[self contentView] addSubview:label];
  [self center];
  [self setIsVisible:YES];
  return self;
}

- (BOOL)windowShouldClose:(id)sender {
  [NSApp terminate:sender];
  return YES;
}
@end

int main(int argc, char* argv[]) {
  [NSApplication sharedApplication];
  [[[[Window alloc] init] autorelease] makeMainWindow];
  [NSApp run];
}
```
> *A simple 'Hello World' GUI app in Objective-C using AppKit ([source](https://github.com/gammasoft71/Examples_Cocoa/blob/master/src/HelloWorlds/HelloWorld/HelloWorld.m))*

## Swift and Objective-C
Objective-C is a superset of the C language with added features like objects, dynamic typing making it a full-fledged OOP language. 

[Swift](https://developer.apple.com/swift/) is a programming language introduced by Apple in 2014, as a successor to its existing Objective-C APIs for modernization. With Swift becoming a modern language for app development on iOS and macOS, [Swift UI](https://developer.apple.com/xcode/swiftui/), a declarative UI framework was introduced in 2019.

> [!INFO]
> Swift is comparable to Kotlin, and Swift UI to Jetpack Compose

```swift
import SwiftUI

@main
struct AnApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

struct ContentView: View {
    var body: some View {
        Text("Hello, World!")
    }
}
```
> *A simple 'Hello World' GUI application using Swift UI*

Compilers for Swift (`swiftc`) and Objective-C (`clang`) both generate the LLVM IR which is then converted to machine code by the LLVM compiler backend. Check [this](https://www.swift.org/documentation/package-manager/) answer on Stack Overflow for diagrams explaining the compilation of Swift and Objective-C.
## Package Managers

- [CocoaPods](https://en.wikipedia.org/wiki/CocoaPods) is a package/dependency manager for Cocoa based Objective-C and Swift projects and provides a standard format for packaging and distributing code. The code to be distributed contains a `Podfile` that defines the structure and dependencies of the library.
- [Swift Package Manager (SPM)](https://www.swift.org/documentation/package-manager/) is a modern package management solution from Apple. The package format is defined with a `Package.swift` file.

> [!INFO]
> Android libraries (generalization a Java library) are packaged in the AAR format and distributed mainly on Maven Central

## Xcode
[Xcode](https://developer.apple.com/xcode/) is an IDE developed by Apple to aid developers in building applications for Apple platforms. It uses its own build system to manage and compile source files and artifacts.

> [!INFO]
> Android Studio is the official IDE from Google

## XCFramework
It is used to distribute precompiled binary code for different platforms Apple supports (macOS, iOS, tvOS etc.).

From the [docs](https://developer.apple.com/documentation/xcode/creating-a-multi-platform-binary-framework-bundle),
> An XCFramework bundle, or _artifact_, is a binary package created by Xcode that includes the frameworks and libraries necessary to build for multiple platforms (iOS, iPadOS, macOS, tvOS, visionOS, watchOS, and DriverKit), including Simulator builds. The frameworks can be static or dynamic and also include headers.

---
Knowing the Apple computing stack, will help me understand how the project is structured and compiled thoroughly. Listing the Android development counter parts against each component is also beneficial for having a quick understanding.

I hope the article was informative and helped you understand different tools/frameworks present in the Apple's computing stack.
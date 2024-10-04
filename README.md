# How cpp fixes react-native

## Introduction

What is a native module?

Sometimes a React Native app needs to access a native platform API that is not available by default in JavaScript, for example the native APIs to access Apple or Google Pay. Maybe you want to reuse some existing Objective-C, Swift, Java or C++ libraries without having to reimplement it in JavaScript, or write some high performance, multi-threaded code for things like image processing.

The NativeModule system exposes instances of Java/Objective-C/C++ (native) classes to JavaScript (JS) as JS objects, thereby allowing you to execute arbitrary native code from within JS.

Native modules allowed JavaScript and platform-native code to communicate over the React Native "bridge", which handles cross-platform serialization via JSON.

In a nutshell,

1. JavaScript serialized the state of the UI or whatever native function you wanted to call in JSON that is then passed to the "Bridge"
   - You can imagine it as an object that handles this communication
2. and then proceeds to call the native functions itself

This serialization until now has been a performance bottleneck to any React Native application and library because it's slow, especially when there are a lot of things happening in the app between UI updates, native modules calls or simply there are is a lot of data

So the idea of the React Native Core Team was to replace this Bridge with JSI (which stands for JavaScript Interface), that instead of being an JS object like the "bridge", it's a collection of headers and functions that allow the native functions calling, so this is the reason why the new architecture is called "bridgeless mode"

This change is not gonna be instant, as you know (hopefully), we cannot kill already existing app and libraries, and to start the process of migration from the bridge, it has not been totally removed in favour of JSI, but right now they're working together thanks to the "interop layer" which allows the old syntax and ways to develop native modules to work in "bridgless mode"

This layer, in the future, will be removed in favour of JSI.

For example we created this little library to demonstrate and show you the capabilities of these new native modules

## get-random-values
We created a little but yet useful React Native library that allows you to generate random numbers securely and fill a bytearray using a very popular library written in C called `libsodium`.

The library exposes a single method `getRandomValues`. The purpouse of this method is to polyfill `crypto.getRandomValues` in the global scope that is missing in the React Native runtime. This is useful to make React Native compatible with other libraries that are designed to run on browser or node runtime, for example `uuid` or `ethers.js`

The implementation of the library is done using a very new framework called `react-native-nitro-modules` that adds some abstractions that allow you to write native modules in Swift/Objective-C, Java/Kotlin or C++ faster.

In this case we decided to write the Native Module using C++ only. The reason is that we wanted to use a single codebase and on the `libsodium`github  repo it was very easy to get the precompiled binaries for Android and iOS.

The Native Module is built on the iOS side using CocoaPods and on the Android side using a combination of gradle and CMake.

### Folder structure of the Native Module
The scaffolding of the library was done using `create-react-native-library` and then we modified it with the files needed to implement the Native Module in C++.

```
react-native-get-random-values
├── ios
│   └── Sodium.mm
├── RNSodium.podspec
├── android
│   ├── CMakeLists.txt
│   ├── cpp-adapter.cpp
│   └── src/main/java/com
│       └── sodium
│           └── SodiumPackage.java
├── src
│   └── index.ts
└── cpp
    ├── Sodium.cpp
    └── Sodium.hpp

```

### TypeScript implementation

```ts
import { NitroModules, type HybridObject } from "react-native-nitro-modules"

export type CompatibleArray =
  | Uint8Array
  | Int8Array
  | Uint16Array
  | Int16Array
  | Int32Array
  | Uint32Array

export interface Sodium extends HybridObject {
  getRandomValues(buffer: ArrayBuffer): void;
}

// Create the native module instance
// A Hybrid Object is a native object that can be used from JS like any other object. They can have natively implemented methods, as well as properties (get + set).
const sodium = NitroModules.createHybridObject<Sodium>("Sodium")

const getRandomValues = <TypedArray extends CompatibleArray>(array: TypedArray): TypedArray => {
  // Call the native method getRandomValues registered in the Sodium Hybrid Object
  sodium.getRandomValues(array.buffer)
  // The original byte array is modified in place, so we return it
  return array
}

```
That was all for Typescript.

### C++ implementation
To implement the HybridObject in C++ we need the following files:c
- `Sodium.hpp`
- `Sodium.cpp`

`Sodium.hpp` contains the definition of our `Sodium` class which extends `HybridObject` class defined by the NitroModules framework.

```cpp

#pragma once
using namespace margelo::nitro;

class Sodium: public HybridObject {
    public:
      explicit Sodium();
      virtual ~Sodium() {};

    public:
      // This is out exported method that will be accessible from JavaScript
      void getRandomValues(const std::shared_ptr<ArrayBuffer>& buffer); 

    protected:
      // Loads the other base methods and properties of the HybridObject like `toString` and `name`
      void loadHybridMethods() override;
};

```

`Sodium.cpp` contains the implementation of our `Sodium` Hybrid Object.

```cpp
#include "Sodium.hpp"
#include "sodium.h" // <- libsodium libraries

Sodium::Sodium(): HybridObject() {
  // Initialize libsodium
  sodium_init();
}

void Sodium::loadHybridMethods() {
    // load base methods/properties
    HybridObject::loadHybridMethods();
    // load custom methods/properties defined by ourselves
    registerHybrids(this, [](Prototype& prototype) {
      // This method will be callable from JavaScript
      prototype.registerHybridMethod("getRandomValues", &Sodium::getRandomValues);
    });
  }

void Sodium::getRandomValues(const std::shared_ptr<ArrayBuffer>& buffer) {
  // Obtain a byteArray from the ArrayBuffer object
  uint8_t* data = buffer.get()->data();
  // Get the bytelength of the ArrayBuffer
  size_t arraySize = buffer->size();
  // Fill the byteArray with random values
  randombytes_buf(data, arraySize);
}

```
The core of the native module is now written. The only remaining thing is to tell to XCode and AndroidStudio how to compile the cpp files together with the libsodium library and the rest of the target app. These are scaffolding files that once written don't need to be modified again.

### iOS side settings

The important files are:
- `RNSodium.podspec`
- `Sodium.mm`

`RNSodium.podspec` è il file di configurazione del modulo per CocoaPods, in questo caso abbiamo aggiunto la dipendenza per la libreria di libsodium e abbiamo definito il codice sorgente del modulo.

`RNSodium.podspec` is the configuration file for CocoaPods, in this case we added the dependency for the libsodium library and we defined the source code of the module.

```ruby
  # RNSodium.podspec

  # ...

Pod::Spec.new do |s|
  # ...

  s.source_files = "ios/**/*.{h,m,mm}", "cpp/**/*.{hpp,cpp,c,h}" # our Native module source code

  s.vendored_framework = "Clibsodium.xcframework" # <- libsodium precompiled for every Apple arch

  s.dependency "NitroModules" # <- Dependency of NitroModules framework

  install_modules_dependencies(s) # <- Install NitroModules dependencies
end

```
`Sodium.mm` is the file that contains a function that is called automatically by the NitroModules framework to register the native module and make it accessible on the react native side.

```objc

#import <NitroModules/HybridObjectRegistry.hpp>
#import "Sodium.hpp"

@interface SodiumOnLoad: NSObject
@end

@implementation SodiumOnLoad

  using namespace margelo::nitro;
+ (void)load {
  HybridObjectRegistry::registerHybridObjectConstructor(
    "Sodium", []() -> std::shared_ptr<HybridObject> { return std::make_shared<Sodium>();
  });
}

@end

```

### Android side settings
- `CMakelists.txt`
- `SodiumPackage.java`
- `cpp-adapter.cpp`

Java calls the C++ code through JNI (Java Native Interface), so we need to write a C++ adapter that will be called by Java to register the native module.


`CMakeLists` contains the configurations for the compilation of the native module on the Android side. And it is the equivalent of the `RNSodium.podspec` file for iOS.

```cmake

cmake_minimum_required(VERSION 3.9.0)

project(RNSodium)
set(PACKAGE_NAME RNSodium)

set(CMAKE_VERBOSE_MAKEFILE ON)
set(CMAKE_CXX_STANDARD 20)

# Define C++ library and add all sources
add_library(
  ${PACKAGE_NAME} SHARED
  src/main/cpp-adapter.cpp
  ../cpp/Sodium.cpp
)

# Add into the include directories the path to the libsodium headers
include_directories(
  ../cpp
  "${CMAKE_SOURCE_DIR}/../libsodium-android/${CMAKE_ANDROID_ARCH_ABI}/include"
)

include_directories(
  ../cpp
)
find_package(fbjni REQUIRED)
find_package(ReactAndroid REQUIRED)
find_package(react-native-nitro-modules REQUIRED)

find_library(LOG_LIB log)

# linkiamo libsodium
add_library(sodiumcpp SHARED IMPORTED)

set_target_properties(
  sodiumcpp
  PROPERTIES
  IMPORTED_LOCATION
  ${CMAKE_SOURCE_DIR}/../libsodium-android/${CMAKE_ANDROID_ARCH_ABI}/lib/libsodium.so
)

# Link all libraries together
target_link_libraries(
  ${PACKAGE_NAME} # <- Our Native Module
  ${LOG_LIB}           
  android
  sodiumcpp # <- Link libsodium
  fbjni::fbjni
  ReactAndroid::jsi
  ReactAndroid::turbomodulejsijni
  ReactAndroid::react_nativemodule_core
  ReactAndroid::react_render_core
  ReactAndroid::runtimeexecutor
  ReactAndroid::fabricjni
  ReactAndroid::react_debug
  ReactAndroid::react_render_core
  ReactAndroid::react_render_componentregistry
  ReactAndroid::rrc_view
  ReactAndroid::folly_runtime
  react-native-nitro-modules::NitroModules # <- Link NitroModules Framework
  )

```

Il file `cpp-adapter.cpp` contiene la funzione che viene chiamata in automatico dal framework di NitroModules per registrare il modulo nativo, è l'equivalente del file `Sodium.mm` per iOS.

`cpp-adpater.cpp` contains the function that is called automatically by the NitroModules framework to register the native module and make it accessible on the react native side. It's the equivalent of the `Sodium.mm` file for iOS.

```cpp
#include <jni.h>

#include "Sodium.hpp"
#include <NitroModules/HybridObjectRegistry.hpp>

JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void*) {

  using namespace margelo::nitro;
  HybridObjectRegistry::registerHybridObjectConstructor(
    "Sodium", []() -> std::shared_ptr<HybridObject> { return std::make_shared<Sodium>();
  });

  return JNI_VERSION_1_2;
}
```

`SodiumPackage.java` contains the class that will be called by React Native to register the native module.

```java
package com.sodium;

import androidx.annotation.NonNull;
import android.util.Log;
import androidx.annotation.Nullable;

import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.TurboReactPackage;

import com.facebook.react.bridge.NativeModule;
import com.facebook.react.module.model.ReactModuleInfoProvider;

import java.util.HashMap;

public class SodiumPackage extends TurboReactPackage {
  public static final String TAG = "RNSodium";

  static {
    try {
      Log.i(TAG, "Loading C++ library...");
      System.loadLibrary("RNSodium"); // <- Load the C++ built with CMakeLists
      Log.i(TAG, "Successfully loaded C++ library!");
    } catch (Throwable e) {
      Log.e(TAG, "Failed to load C++ library! Is it properly installed and linked?", e);
      throw e;
    }
  }

  // ...
}

```

### References

https://reactnative.dev/docs/native-modules-intro

https://github.com/reactwg/react-native-new-architecture/blob/main/docs/turbo-modules.md

https://github.com/reactwg/react-native-new-architecture/discussions/154

https://reactnative.dev/blog/2018/06/14/state-of-react-native-2018#architecture

https://github.com/LinusU/react-native-get-random-values

https://www.npmjs.com/package/@korekoi/react-native-get-random-values

https://mrousavy.github.io/nitro/docs/what-is-nitro

https://github.com/jedisct1/libsodium

https://www.npmjs.com/package/create-react-native-library

<img src="./assets/repoQR.png" />
<img src="./assets/issueQR.png" />

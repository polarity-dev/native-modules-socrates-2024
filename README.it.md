# How cpp fixes react-native

## Introduzione

Cos’è un native module?

A volte un'applicazione React Native deve accedere a un API della piattaforma nativa che non è disponibile di default in JavaScript, ad esempio le API native per accedere a Apple o Google Pay. Forse si desidera riutilizzare alcune librerie Objective-C, Swift, Java o C++ esistenti senza doverle re-implementare in JavaScript, oppure scrivere codice multi-thread ad alte prestazioni per cose come l'elaborazione delle immagini.

Il sistema NativeModule espone istanze di classi Java/Objective-C/C++ (native) a JavaScript (JS) come oggetti JS, permettendo così di eseguire codice nativo arbitrario da JS.

I moduli nativi fino ad ora hanno consentito a JavaScript e al codice nativo di comunicare attraverso il “Bridge” di React Native, che gestisce la serializzazione cross-platform tramite JSON.

In poche parole JavaScript serializza lo stato della UI, o qualsiasi funzione che si volesse chiamare dall’ambiente nativo, in JSON che poi passato al Bridge (che si può immaginare come un oggetto che gestisce questa comunicazione), procede a chiamare le funzioni native.

Questa serializzazione tramite JSON fino ad ora è stata un bottleneck per le performance di qualsiasi app e libreria realizzata con React Native, poiché lenta.
JSON.stringifizzare, soprattutto nel caso in cui ci fossero tante cose che accadono all’interno dell’app, tra UI updates, chiamate a native modules o dove ci sono molti dati.

Quindi l’idea del team core di React Native è stato quello di sostituire questo “JSON Bridge” con JSI (JavaScript Interface) che al contrario del Bridge non è un oggetto, ma un insieme di headers e funzioni che permettono di chiamare codice nativo, da questo il nome della nuova architettura “bridgeless mode”

Questo cambiamento non è stato immediato, in modo da mantenere funzionanti le app già sviluppate e per inizializzare il processo di migrazione dal Bridge, il Bridge non è stato completamente rimosso in favore di JSI, ma funzionavano in parallelo, questo grazie ad uno strato aggiuntivo, l’interop layer che permette alla sintassi con cui erano sviluppati i vecchi moduli di funzionare in “bridgeless mode”.
Questo layer, in futuro, verrà poi rimosso totalmente in favore di JSI

Per esempio abbiamo creato questa piccola libreria per dimostrare le capacità dei nuovi native modules.

## get-random-values
In questo caso abbiamo creato un modulo nativo che permette di generare numeri casuali in modo sicuro e riempire un array di bytes utilizzando una libreria molto popolare scritta in C chiamata `libsodium`.

La libreria espone un solo metodo `getRandomValues` che serve per integrare questa funzione nello scope globale dell'applicazione di react-native. Questo serve per poter essere compatibile con altre librerie pensate per la runtime browser o per node, per esempio `uuid` oppure `ethers.js`

L'implementazione è stata fatta utilizzando un framework uscito da pochissimo chiamato `react-native-nitro-modules` che aggiunge delle astrazioni che permettono di scrivere più velocemente dei moduli nativi in Swift/Objective-C, Java/Kotlin o C++

In questo caso abbiamo scelto di scrivere il modulo di C++ per poter utilizzare una sola codebase e sulla repo di libsodium era molto facile ottenere i precompilati per Android e iOS.

### Struttura della libreria
La libreria di base è stata creata usando `create-react-native-library` e poi sono stati aggiunti i file necessari per utilizzare i nitro modules
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

### Implementazione lato TypeScript

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

// Crea un istancza del native module
const sodium = NitroModules.createHybridObject<Sodium>("Sodium")

const getRandomValues = <TypedArray extends CompatibleArray>(array: TypedArray): TypedArray => {
  // Chiamata al metodo del native module definito nella interface
  sodium.getRandomValues(array.buffer)
  // L'array originale viene modificato direttamente dal modulo nativo
  return array
}

```
Questo era tutto quello che ci serviva lato JavaScript.

### Implementazione lato C++
L'implementazione lato C++ contiene solamente due files
- `Sodium.hpp`
- `Sodium.cpp`

Il file `Sodium.hpp` contiene la definizione della classe `Sodium` che estende la classe `HybridObject` definita dal framework di NitroModules.

```cpp

#pragma once
using namespace margelo::nitro;

class Sodium: public HybridObject {
    public:
      // Constructor
      explicit Sodium();

      // Destructor
      virtual ~Sodium() {};

    public:
      // Methods
      void getRandomValues(const std::shared_ptr<ArrayBuffer>& buffer); // <- Metodo che verrà chiamato dal modulo JavaScript

    protected:
      // Hybrid Setup
      void loadHybridMethods() override;

    protected:
      // Tag for logging
      static constexpr auto TAG = "Sodium";
};

```

Il file `Sodium.cpp` contiene l'implementazione dei metodi della classe `Sodium`.

```cpp
#include "Sodium.hpp"
#include "sodium.h" // <- Libreria di libsodium

Sodium::Sodium(): HybridObject(Sodium::TAG) { // <- Costruttore
  // Inizializza la libreria di libsodium
  sodium_init();
}

void Sodium::loadHybridMethods() {
    // load base methods/properties
    HybridObject::loadHybridMethods();
    // load custom methods/properties
    registerHybrids(this, [](Prototype& prototype) {
      prototype.registerHybridMethod("getRandomValues", &Sodium::getRandomValues); // <- Registra il metodo getRandomValues per essere chiamato dal modulo JavaScript
    });
  }

void Sodium::getRandomValues(const std::shared_ptr<ArrayBuffer>& buffer) {
  // Otteniamo la lista di bytes dall'ArrayBuffer
  uint8_t* data = buffer.get()->data();
  // Otteniamo la lunghezza dell'ArrayBuffer
  size_t arraySize = buffer->size();
  // Generiamo numeri casuali sicuri e li inseriamo nell'ArrayBuffer
  randombytes_buf(data, arraySize);
}

```

Il core della libreria è stato scritto, ora va aggiunta alla configurazione della nostra libreria il modo con cui vengono compilati di file di cpp insieme alla libreria di libsodium e al resto dell'app.
Una volta che questi file sono impostati non c'è più bisogno di toccarli.

### Implementazione lato iOS

Ci sono due file importanti per l'implementazione del modulo nativo lato iOS:
- `RNSodium.podspec`
- `Sodium.mm`

Il file `RNSodium.podspec` è il file di configurazione del modulo per CocoaPods, in questo caso abbiamo aggiunto la dipendenza per la libreria di libsodium e abbiamo definito il codice sorgente del modulo.

```ruby
  # RNSodium.podspec

  # ...

Pod::Spec.new do |s|
  # ...

  s.source_files = "ios/**/*.{h,m,mm}", "cpp/**/*.{hpp,cpp,c,h}" # <- Codice sorgente del modulo che va compilato

  s.vendored_framework = "Clibsodium.xcframework" # <- Libreria di libsodium compilata per vari target iOS

  s.dependency "NitroModules" # <- Dipendenza per il framework di NitroModules

  install_modules_dependencies(s) # <- Funzione che installa le dipendenze del modulo definita da NitroModules
end

```
Ora passiamo al file `Sodium.mm` che è il file che contiene una funzione che viene chiamata in automatico dal framework di NitroModules per registrare il modulo nativo.

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

### Implementazione lato Android
- `CMakelists.txt`
- `SodiumPackage.java`
- `cpp-adapter.cpp`

Il codice di java chiama il codice nativo scritto in C++ tramite la libreria JNI.

Il file CmakeLists contiene le configurazioni per la compilazione del modulo nativo lato Android. Ed è l'equivalente del file `RNSodium.podspec` per iOS.

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

# Aggiungiamo nelle include directories il path dei precompilati di libsodium della target arch di Android
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
  ${PACKAGE_NAME} # <- Nome del modulo nativo
  ${LOG_LIB}           
  android
  sodiumcpp # <- Linkiamo la libreria di libsodium
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
  react-native-nitro-modules::NitroModules # <- Linkiamo il framework di NitroModules
  )

```

Il file `cpp-adapter.cpp` contiene la funzione che viene chiamata in automatico dal framework di NitroModules per registrare il modulo nativo, è l'equivalente del file `Sodium.mm` per iOS.
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

E infine il file `SodiumPackage.java` che contiene la classe che esporta il modulo nativo per essere utilizzato in JavaScript.

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
      System.loadLibrary("RNSodium"); // <- Carica la libreria nativa creata usando CMake
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

---
title: Building JNI bindings For clip.cpp
tags:
  - android
  - programming
  - cpp
  - machine-learning
---
# Building an Android app for `clip.cpp`
`clip.cpp` is a runtime based on GGML and `llama.cpp` that performs inference on the popular CLIP model. It includes examples written in C/C++ and bindings that can be used in a Python package. I found `clip.cpp` while searching for an efficient method to infer CLIP in an Android app, and also discovered an issue on the repository created by the author suggesting the inclusion of JNI bindings along with an Android sample. 
For the Android app to use the C++ routines of `clip.cpp`, JNI bindings are necessary as they allow JVM to bind Java/Kotlin routines with their native (C/C++) counterparts present in the object-code of the shared libraries. Here are the three main contributions I made to the `clip.cpp` repository in response to the issue:

1. JNI bindings for `clip.h` 
2. An Android app built on top of the JNI bindings that allows users to select an image and enter a textual description to compute their semantic similarity using the CLIP model (stored on the mobile device)

## Implementation

### 1. Setup
First, we clone the `clip.cpp` repository and the dependent sub-modules,
```commandline
git clone https://github.com/monatis/clip.cpp
cd clip.cpp
git submodule update --init --recursive
```
Next, create a new directory, `clip.android` in the `examples` directory which will the house the Android project (demo app). Using Android Studio, create a new `Empty Project (Compose)` in the `examples/clip.android` directory.

The default project which gets created will only contain the `app` module. By navigating to `File > New > New Module > Android Library Module` in Android Studio, we create another module named `clip` in the project.

The `clip` module will contain the JNI bindings and a Java wrapper class that encapsulates the native interfaces we're going to write. To start with the JNI bindings, create a new directory named `cpp` in `clip/src/main`. Add two files in the `cpp` directory,

- `clip_android.cpp`: JNI bindings
- `CMakeLists.txt`: CMake script that describes the build process of `clip_android.cpp`

We also need to make sure that Gradle compiles `clip_android.cpp` when building the module/project. To do this, we modify `clip/build.gradle.kts` and add the path to the `CMakeLists.txt` we just created,
```kotlin
plugins {  
    alias(libs.plugins.android.library)  
    alias(libs.plugins.jetbrains.kotlin.android)  
}  
  
android {  
    namespace = "android.clip.cpp"  
    compileSdk = 34  
    defaultConfig {  
        // ...
        externalNativeBuild {
            cmake {
                arguments += listOf(
                    "-DCMAKE_BUILD_TYPE=Release",
                    "-DCLIP_NATIVE=OFF"
                )
            }
        }
    }    
    // ...
    externalNativeBuild {  
        cmake {  
            path("src/main/cpp/CMakeLists.txt")  
            version = "3.22.1"  
        }  
    }
}  
  
dependencies {  
    // ...
}
```
Setting the `CLIP_NATIVE` to `Off` ensures that the `-march=native` is not passed to `clang` (from Android NDK) as it is not supported. 

Before we build the `clip` module, we need to configure the build process by modifying `CMakeLists.txt`,
```cmake
# clip/src/main/cpp/CMakeLists.txt
cmake_minimum_required(VERSION 3.22.1)  
  
project("clip-android")  
  
add_subdirectory(../../../../../../ build-clip)  
  
add_library(  
        ${CMAKE_PROJECT_NAME}  
        SHARED        
        clip_android.cpp)  
  
target_link_libraries(${CMAKE_PROJECT_NAME}  
        clip  
        ggml  
        android  
        log)
```
In the script above, `add_subdirectory()` will add the *main* CMake project i.e. the `CMakeLists.txt` present in the `clip.cpp` directory in the build process.  This is important as the libraries on which `clip_android.cpp` will dependent i.e. `clip` and `ggml` come from the main `clip.cpp` project, their compilation being defined in `clip.cpp/CMakeLists.txt` and `ggml/CMakeLists.txt`. `android` and `log` libraries are made available by linker provided by Android NDK. 

Also, we load the `clip` module as a dependency in the `app` module, to access the wrapper class we'll be writing in the `clip` module. To do so, in `app/build.gradle.kts`, add the following in `dependencies` block,

```kotlin
dependencies {
    // android dependencies
    
    implementation(project(":clip"))

	// android-test dependencies
}
```

> [!NOTE]
> In a good production setting, the `clip` module can be packed as a JAR/AAR and distributed through Maven Central

Click `Build > Make Project` to build the `clip` module now. 

## 2. Writing JNI bindings
We'll try to create bindings for declarations (function prototypes) present in `clip.h`. We also created a Java class in `clip/src/main/java/android/example/clip` named `CLIPAndroid.java`.

For the following prototype in `clip.h`,
```c
struct clip_ctx * clip_model_load(const char * fname, const int verbosity);
```
We create a method in `CLIPAndroid.java`,
```java
private native long clipModelLoad(String filePath, int verbosity);
```
and the corresponding JNI binding in `clip_android.cpp`,
```cpp
#include <jni.h>
#include <android/log.h>
#include "clip.h"

#define TAG "clip-android.cpp"
#define LOGi(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)
#define LOGe(...) __android_log_print(ANDROID_LOG_ERROR, TAG, __VA_ARGS__)

extern "C" JNIEXPORT jlong JNICALL
Java_android_clip_cpp_CLIPAndroid_clipModelLoad__Ljava_lang_String_2I(
    JNIEnv *env,
    jobject,
    jstring file_path,
    jint verbosity
) {
    const char* file_path_chars = env -> GetStringUTFChars(file_path, nullptr);
    LOGi("Loading the model from %s", file_path_chars);
    const clip_ctx* ctx = clip_model_load(file_path_chars, verbosity);

    if (!ctx) {
        LOGe("Failed to load the model from %s", file_path_chars);
        return 0;
    }

    env -> ReleaseStringUTFChars(file_path, file_path_chars);
    return reinterpret_cast<jlong>(ctx);
}
```
Similarly, we go on to write bindings for the following functions in `clip.h`,
* `clip_free`
* `clip_image_encode`
* `clip_image_batch_encode`
* `clip_text_encode`
* `clip_text_batch_encode`
* `clip_get_vision_hparams`
* `clip_get_text_hparams`
For the last two functions, `clip_get_vision_hparams` and `clip_get_text_hparams`, we return Java objects instantiated at their bindings in `clip_android.cpp`,
```cpp
extern "C" JNIEXPORT jobject JNICALL
Java_android_clip_cpp_CLIPAndroid_clipGetVisionHyperParameters__J(
        JNIEnv *env,
        jobject,
        jlong ctx_ptr
) {
    auto* ctx = reinterpret_cast<clip_ctx*>(ctx_ptr);
    clip_vision_hparams* vision_params = clip_get_vision_hparams(ctx);

    jclass cls = env -> FindClass("android/clip/cpp/CLIPAndroid$CLIPVisionHyperParameters");
    jmethodID constructor = env -> GetMethodID(cls, "<init>", "(IIIIIII)V");
    jvalue args[7];
    args[0].i = vision_params -> image_size;
    args[1].i = vision_params -> patch_size;
    args[2].i = vision_params -> hidden_size;
    args[3].i = vision_params -> projection_dim;
    args[4].i = vision_params -> n_intermediate;
    args[5].i = vision_params -> n_head;
    args[6].i = vision_params -> n_layer;
    jobject object = env -> NewObjectA(cls, constructor, args);

    return object;
}

extern "C" JNIEXPORT jobject JNICALL
Java_android_clip_cpp_CLIPAndroid_clipGetTextHyperParameters__J(
        JNIEnv *env,
        jobject,
        jlong ctx_ptr
) {
    auto* ctx = reinterpret_cast<clip_ctx*>(ctx_ptr);
    clip_text_hparams* text_params = clip_get_text_hparams(ctx);

    jclass cls = env -> FindClass("android/clip/cpp/CLIPAndroid$CLIPTextHyperParameters");
    jmethodID constructor = env -> GetMethodID(cls, "<init>", "(IIIIIII)V");
    jvalue args[7];
    args[0].i = text_params -> n_vocab;
    args[1].i = text_params -> num_positions;
    args[2].i = text_params -> hidden_size;
    args[3].i = text_params -> projection_dim;
    args[4].i = text_params -> n_intermediate;
    args[5].i = text_params -> n_head;
    args[6].i = text_params -> n_layer;
    jobject object = env -> NewObjectA(cls, constructor, args);

    return object;
}
```
The Java classes in `CLIPAndroid.java` are,
```java
public static class CLIPVisionHyperParameters {

    public final int imageSize;
    public final int patchSize;
    public final int hiddenSize;
    public final int projectionDim;
    public final int nIntermediate;
    public final int nHead;
    public final int nLayer;
    
    public CLIPVisionHyperParameters(int imageSize, int patchSize, int hiddenSize, int projectionDim, int nIntermediate, int nHead, int nLayer) {
        this.imageSize = imageSize;
        this.patchSize = patchSize;
        this.hiddenSize = hiddenSize;
        this.projectionDim = projectionDim;
        this.nIntermediate = nIntermediate;
        this.nHead = nHead;
        this.nLayer = nLayer;
    }
}

public static class CLIPTextHyperParameters {

    public final int nVocab;
    public final int numPositions;
    public final int hiddenSize;
    public final int projectionDim;
    public final int nIntermediate;
    public final int nHead;
    public final int nLayer;
    
    public CLIPTextHyperParameters(int nVocab, int numPositions, int hiddenSize, int projectionDim, int nIntermediate, int nHead, int nLayer) {
        this.nVocab = nVocab;
        this.numPositions = numPositions;
        this.hiddenSize = hiddenSize;
        this.projectionDim = projectionDim;
        this.nIntermediate = nIntermediate;
        this.nHead = nHead;
        this.nLayer = nLayer;
    }
    
}
```

To pass an image from Java to the binding, we use a `java.nio.ByteBuffer` pass it to the corresponding `native` method along with the `width` and `height` as `jint`. Then in `clip_android.cpp`, we use `env -> GetDirectBufferAddress(img_buffer)` to get a pointer to the buffer's data.
```cpp
extern "C" JNIEXPORT jfloatArray JNICALL
Java_android_clip_cpp_CLIPAndroid_clipImageEncode__JLjava_nio_ByteBuffer_2IIIIZ(
        JNIEnv *env,
        jobject,
        jlong ctx_ptr,
        jobject img_buffer,
        jint width,
        jint height,
        jint n_threads,
        jint vector_dims,
        jboolean normalize
) {
    auto* ctx = reinterpret_cast<clip_ctx*>(ctx_ptr);
    auto* img = clip_image_u8_make();
    img -> nx = width;
    img -> ny = height;
    img -> data = reinterpret_cast<uint8_t*>(env -> GetDirectBufferAddress(img_buffer));
    img -> size = width * height * 3;

    auto* img_f32 = clip_image_f32_make();
    img_f32 -> nx = width;
    img_f32 -> ny = height;
    img_f32 -> data = new float[width * height * 3];
    img_f32 -> size = width * height * 3;
    clip_image_preprocess(ctx, img, img_f32);

    float image_embedding[vector_dims];
    clip_image_encode(ctx, n_threads, img_f32, image_embedding, normalize);
    jfloatArray result = env -> NewFloatArray(vector_dims);
    env -> SetFloatArrayRegion(result, 0, vector_dims, image_embedding);

    return result;
}
```
In the snippet above, first, we create an instance of `clip_image_u8` using the provided buffer's data. Next, we preprocess the instance using `clip_image_preprocess` which yield a `clip_image_f32` instance. This instance of `clip_image_f32` is then passed to `clip_image_encode` to get the embedding. 

### 3. Writing the Java wrapper class
`CLIPAndroid.java` is our wrapper class which (till now) has contained the `native` methods connecting to our JNI binding. But these methods work with `clip_ctx*` represented as a `long` in Java, which isn't good with regards to abstraction. To abstract the inner details and to provide an intuitive API to the user, we write some helper functions, completing `CLIPAndroid.java`,
```java
public class CLIPAndroid {

    private long contextPtr; // holds clip_ctx* pointer

    static {
        System.loadLibrary("clip-android");
    }

    public void load(String filePath, int verbosity) {
        if (!Paths.get(filePath).toFile().exists()) {
            throw new IllegalArgumentException("File not found: " + filePath);
        }
        long ptr = clipModelLoad(filePath, verbosity);
        if (ptr == 0) {
            throw new RuntimeException("Failed to load the model from " + filePath);
        } else {
            contextPtr = ptr;
        }
    }
    
    public CLIPVisionHyperParameters getVisionHyperParameters() {
        return clipGetVisionHyperParameters(contextPtr);
    }

    public CLIPTextHyperParameters getTextHyperParameters() {
        return clipGetTextHyperParameters(contextPtr);
    }

    public float[] encodeImage(ByteBuffer image, int width, int height, int numThreads, int vectorDims, boolean normalize) {
        return clipImageEncode(contextPtr, image, width, height, numThreads, vectorDims, normalize);
    }

    public List<float[]> encodeImage(ByteBuffer[] images, int[] widths, int[] heights, int numThreads, int vectorDims, boolean normalize) {
        if (images.length != widths.length || images.length != heights.length) {
            throw new IllegalArgumentException("images, widths, and heights must have the same length. Got "
                    + images.length + ", " + widths.length + ", " + heights.length);
        }
        float[] vectors = clipBatchImageEncode(contextPtr, images, widths, heights, numThreads, vectorDims, normalize);
        ArrayList<float[]> vectorsList = new ArrayList<>();
        for (int i = 0; i < vectors.length / vectorDims; i++) {
            float[] vec = new float[vectorDims];
            System.arraycopy(vectors, i * vectorDims, vec, 0, vectorDims);
            vectorsList.add(vec);
        }
        return vectorsList;
    }

    public float[] encodeText(String text, int numThreads, int vectorDims, boolean normalize) {
        return clipTextEncode(contextPtr, text, numThreads, vectorDims, normalize);
    }

    public float getSimilarityScore(float[] vec1, float[] vec2) {
        if (vec1.length != vec2.length) {
            throw new IllegalArgumentException("Vectors must have the same length. Got " + vec1.length + ", " + vec2.length);
        }
        return clipSimilarityScore(vec1, vec2);
    }

    public void close() {
        clipModelRelease(contextPtr);
    }

	// native methods here ...

}
```
With some more Compose code in `app` module, specifically in `MainActivity` and `MainActivityViewModel`, we should be ready with the demo app.
## Demo App

| ![[clip-cpp (4).png\|300]] | ![[clip-cpp (3).png\|300]] |
| -------------------------- | -------------------------- |
| ![[clip-cpp (6).png\|300]] | ![[clip-cpp (2).png\|300]] |
| ![[clip-cpp (5).png\|300]] | ![[clip-cpp (1).png\|300]] |







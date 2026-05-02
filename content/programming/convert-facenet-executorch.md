---
title: Running FaceNet on Android with ExecuTorch
tags:
  - programming
  - android
  - machine-learning
---
We follow these steps to run FaceNet in an Android app with ExecuTorch:
1. Load the FaceNet mode using `facenet-pytorch` as a `torch.nn.Module`
2. After fixing some issues, we lower the module and export it to the EXIR format
3. Use the `executorch` package to partition the model graph for the XNN delegate
4. Load the partitioned model (`.pte` file) in an Android app

An example can be found here in the `feature/use-executorch-api` branch:
https://github.com/shubham0204/OnDevice-Face-Recognition-Android/tree/feature/use-executorch-api
(check `FaceNet.kt` from above branch to get the HF URL of the model file)

## Exporting the FaceNet model to the ExecuTorch format
Install the following dependencies in a Python virtual environment:
```bash
pip3 install facenet-pytorch executorch
```

The following code is used to load the model, export it to the EXIR format and subsequently to the ExecuTorch format (using XNN/CPU as a backend):
```python
# main.py
import torch  
from executorch.backends.xnnpack.partition.xnnpack_partitioner import XnnpackPartitioner  
from executorch.exir import to_edge_transform_and_lower  
from facenet_pytorch import InceptionResnetV1  
from torch.export import export  
  
model = InceptionResnetV1(pretrained='vggface2').eval()  
sample_inputs = (  
    torch.zeros(1, 160, 160, 3, dtype=torch.float32),  
)  
  
exported_program = export(model, sample_inputs)  
executorch_program = to_edge_transform_and_lower(  
    exported_program,  
    partitioner = [XnnpackPartitioner()],  
).to_executorch()  
  
with open("pte/facenet-512-vggface2/model.pte", "wb") as file:  
    file.write(executorch_program.buffer)
```

As `executorch` expects `torch==2.11.0` and `facenet-pytorch` expects `torch==2.4.0`, we face the following error when trying to import the `InceptionResnetV1` model:
```
Traceback (most recent call last):
  File "/Users/shubhampanchal/PythonProjects/embedding-model-executorch/facenet.py", line 4, in <module>
    from facenet_pytorch import InceptionResnetV1
  File "/Users/shubhampanchal/PythonProjects/embedding-model-executorch/.venv/lib/python3.12/site-packages/facenet_pytorch/__init__.py", line 2, in <module>
    from .models.mtcnn import MTCNN, PNet, RNet, ONet, prewhiten, fixed_image_standardization
  File "/Users/shubhampanchal/PythonProjects/embedding-model-executorch/.venv/lib/python3.12/site-packages/facenet_pytorch/models/mtcnn.py", line 6, in <module>
    from .utils.detect_face import detect_face, extract_face
  File "/Users/shubhampanchal/PythonProjects/embedding-model-executorch/.venv/lib/python3.12/site-packages/facenet_pytorch/models/utils/detect_face.py", line 3, in <module>
    from torchvision.transforms import functional as F
  File "/Users/shubhampanchal/PythonProjects/embedding-model-executorch/.venv/lib/python3.12/site-packages/torchvision/__init__.py", line 6, in <module>
    from torchvision import _meta_registrations, datasets, io, models, ops, transforms, utils
  File "/Users/shubhampanchal/PythonProjects/embedding-model-executorch/.venv/lib/python3.12/site-packages/torchvision/_meta_registrations.py", line 163, in <module>
    @torch._custom_ops.impl_abstract("torchvision::nms")
     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/shubhampanchal/PythonProjects/embedding-model-executorch/.venv/lib/python3.12/site-packages/torch/library.py", line 1087, in register
    use_lib._register_fake(
  File "/Users/shubhampanchal/PythonProjects/embedding-model-executorch/.venv/lib/python3.12/site-packages/torch/library.py", line 204, in _register_fake
    handle = entry.fake_impl.register(
             ^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/shubhampanchal/PythonProjects/embedding-model-executorch/.venv/lib/python3.12/site-packages/torch/_library/fake_impl.py", line 50, in register
    if torch._C._dispatch_has_kernel_for_dispatch_key(self.qualname, "Meta"):
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
RuntimeError: operator torchvision::nms does not exist

```

A simple solution is to remove the `_meta_registrations` import from `torchvision/__init__.py" line 6`:
```python
import torch  
# from torchvision import _meta_registrations, datasets, io, models, ops, transforms, utils  
from torchvision import datasets, io, models, ops, transforms, utils
```

We also modify the `forward` method of the `InceptionResnetV1` module in `inception_resnet_v1.py` to support the channels-last format for the input tensor (input image) and perform input standardization (expected by the FaceNet model):
```python
def forward(self, x):  
    """Calculate embeddings or logits given a batch of input image tensors.  
  
    Arguments:        x {torch.tensor} -- Batch of image tensors representing faces.  
    Returns:        torch.tensor -- Batch of embedding vectors or multinomial logits.    """    # channels-last to channels-first format  
    x = torch.permute(x, (0, 3, 1, 2))  
  
    # fixed standardization  
    x = (x - 127.5) / 128.0
    x = self.conv2d_1a(x)  
	x = self.conv2d_2a(x)  
	x = self.conv2d_2b(x)
	# rest of the impl ...
```

When we now run `main.py`, the script executes without any errors leaving us with a `model.pte` file. Move the `model.pte` file to the `assets` directory of the Android app.

## Using the ExecuTorch model in an Android app
Add the ExecuTorch Android library to `app/build.gradle.kts`:
```kotlin
implementation("org.pytorch:executorch-android:1.2.0")
```

Use the following `FaceNet` class to infer the model:
```kotlin
import android.content.Context  
import android.graphics.Bitmap  
import androidx.core.graphics.scale  
import kotlinx.coroutines.Dispatchers  
import kotlinx.coroutines.withContext  
import org.pytorch.executorch.EValue  
import org.pytorch.executorch.Module  
import org.pytorch.executorch.Tensor  
import java.io.File  
import java.io.FileOutputStream  
import java.nio.IntBuffer  

class FaceNet(val context: Context) {  
    private val imgSize = 160L  
    private var module: Module  
  
    init {  
        module = Module.load(copyAndReturnPath("model.pte"))  
    }  
  
    // Gets an face embedding using FaceNet  
    suspend fun getFaceEmbedding(image: Bitmap) =  
        withContext(Dispatchers.Default) {  
            return@withContext runFaceNet(convertBitmapToBuffer(image))[0]  
        }  
  
    // Run the FaceNet model  
    private fun runFaceNet(inputs: FloatArray): Array<FloatArray> {  
        val imageTensor = Tensor.fromBlob(  
            inputs,  
            longArrayOf(1, imgSize, imgSize, 3)  
        )  
        val outputTensor = module.forward(EValue.from(imageTensor))[0].toTensor()  
        val embedding = outputTensor.dataAsFloatArray  
        return arrayOf(embedding)  
    }  
  
    // Resize the given bitmap and convert it to a ByteBuffer  
    private fun convertBitmapToBuffer(image: Bitmap): FloatArray {  
        val resizedBitmap = image.scale(160, 160, false)  
        val intBuffer = IntBuffer.allocate(1 * 160 * 160 * 3)  
        resizedBitmap.copyPixelsToBuffer(intBuffer)  
        return intBuffer.array().map { it.toFloat() }.toFloatArray()  
    }  
  
    // Copy the file from the assets to the app's internal/private storage  
    // and return its absolute path    
    private fun copyAndReturnPath(assetsFilepath: String): String {  
        val storageFile = File(context.filesDir, assetsFilepath)  
        if (!storageFile.exists()) {  
            storageFile.parentFile?.mkdir()  
            FileOutputStream(storageFile).use { outputStream ->  
                context.assets.open(assetsFilepath).use { inputStream ->  
                    inputStream.copyTo(outputStream)  
                }  
            }        }  
        return storageFile.absolutePath  
    }  
}
```

We initialize a `Module` instance providing the path of the model. Reading models from the `assets` folder using a file-path is not possible, hence, we copy the model from the `assets` folder to the app's internal directory. Note, this operation is performed only once and subsequently initializations will re-use the model file from the app's internal directory.

The `getFaceEmbedding` method will return a `FloatArray` (the face embedding) for the given `Bitmap` (containing a face image).






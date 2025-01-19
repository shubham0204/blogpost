---
title: Model2Vec - Faster Sentence Transformers In Rust
tags:
  - programming
  - machine-learning
  - rust
---

[Model2Vec](https://github.com/MinishLab/model2vec) is a technique from [MinishLab](https://github.com/MinishLab) wherein static-embeddings are generated from a sentence-transformer model, thus reducing the process of encoding a sentence to a tokenize + lookup + pool operation instead of performing a forward-pass on the sentence-transformer model.
### What is Model2Vec?

- The vocabulary of a sentence-transformer model is about 32K tokens. We can store the embeddings for each of these tokens in a lookup table. 
- To encode a sentence, we tokenize it, fetch the token embeddings from the lookup table and pool them to get a sentence embedding. 
- As you might have noticed, these **embeddings are not contextual** i.e. the embedding of a token does not depend on the tokens preceding it in the input sequence. 
- These embeddings are also compressed with PCA to reduce their dimensions.
- The authors claim that the use of PCA increases performance as it normalizes the resulting space.

### Motivation
#### On-Device RAG

- A fast sentence-embedding model can be useful in RAG applications where semantic similarity between two pieces of text has to be computed. 
- The size of the lookup table for 32K tokens with each token having an embedding of length 256 sums to $32000 \times 256 \times sizeof(float32) = 32768000 \ \text{bytes} = 32.7 \ \text{MB}$
- This is ideal for on-device applications, for e.g. on mobile devices, where there are space constraints.

#### Conversion to a more portable format

- The codebase for Model2Vec is in Python, which is not a good choice to run it in a native Android or iOS application.
- The Python package `model2vec` essentially loads the embeddings from a `safetensors` file and the tokenizer from `tokenizer.json` (HuggingFace tokenizers) and computes the sentence embeddings.
- The good news, both, the HuggingFace [`tokenizers`](https://github.com/huggingface/tokenizers) and HuggingFace [`safetensors`](https://github.com/huggingface/safetensors) are written in Rust. The authors of Model2Vec provide a pre-distilled version of  [baai/bge-base-en-v1.5](https://huggingface.co/baai/bge-base-en-v1.5) sentence transformer model.
- We can use the Rust packages of `tokenizers` and `safetensors` to write a library which produces sentence embeddings and then expose C-like interface for interaction with other languages (like Kotlin/Java for Android).

## Rust Implementation
### Setup
We create a new Rust project (currently an executable binary) with `cargo new model2vec` and the required packages to the `Cargo.toml` found in the resulting directory,
```toml
[package]
name = "model2vec"
version = "0.1.0"
edition = "2021"

[dependencies]
tokenizers = "0.21.0"
safetensors = "0.5.2"
memmap2 = "0.9.5"
anyhow = "1.0.95"
rayon = "1.10.0"
```
The dependencies are:
1. `tokenizers` for loading HuggingFace's `tokenizer.json` and to encode textual sequences.
2. `safetensors` for loading `embeddings.safetensors` which gives the access to token embeddings for all 32 tokens present in the model's vocabulary.
3. `memmap2` to create a memory-mapped file for `embeddings.safetensors`.
4. `anyhow`, from [Comprehensive Rust](https://google.github.io/comprehensive-rust/error-handling/anyhow.html), *The [`anyhow`](https://docs.rs/anyhow/) crate provides a rich error type with support for carrying additional contextual information, which can be used to provide a semantic trace of what the program was doing leading up to the error.*
5. `rayon` for data-parallelism and multi-threading.

### The `StaticModel` struct
We create a module named `static_model.rs` that will contain a `struct` `StaticModel` providing functions like `encode` to produce sentence embeddings for a given text.
```rust
use anyhow::Ok;
use anyhow::Result;
use memmap2::MmapOptions;
use rayon::iter::IntoParallelRefIterator;
use rayon::iter::ParallelIterator;
use safetensors::SafeTensors;
use std::fs::File;
use std::sync::Arc;
use std::sync::Mutex;
use tokenizers::Tokenizer;

pub struct StaticModel {
    tokenizer: Tokenizer,
    embedding_dims: usize,
    embeddings_u8: Vec<u8>,
}
```
The structure the binary data for the embeddings and the dimensions for each embedding. To instantiate this structure, we define a `StaticModel::new` function, 
```rust
impl StaticModel {
    pub fn new(embeddings_filepath: &str, tokenizer_filepath: &str) -> Result<Self> {
        // load the tokenizer
        let tokenizer: Tokenizer = Tokenizer::from_file(tokenizer_filepath).expect("");

        // load the embeddings
        let tensor_file: File = File::open(embeddings_filepath)?;
        let buffer = unsafe { MmapOptions::new().map(&tensor_file)? };
        let tensors = SafeTensors::deserialize(&buffer)?;
        let embeddings_tensor_view = tensors.tensor("embeddings")?;
        let embeddings_u8 = embeddings_tensor_view.data().to_vec();
        let embedding_dims = embeddings_tensor_view.shape()[1];
        Ok(StaticModel {
            tokenizer,
            embedding_dims,
            embeddings_u8,
        })
    }
}
```
Now comes the interesting part, implementing the `StaticModel::encode` function which given `sequences: Vec<String>` returns embeddings of type `Vec<Vec<f32>>`.
```rust
impl StaticModel {
    // StaticModel::new()

    pub fn encode(&self, sequences: &Vec<String>, num_threads: usize) -> Result<Vec<Vec<f32>>> {
        // set num threads for Rayon
        rayon::ThreadPoolBuilder::new()
            .num_threads(num_threads)
            .build_global()?;

        // tokenize the input sequences
        let tokenized_sequences = self
            .tokenizer
            .encode_batch(sequences.to_vec(), false)
            .expect("tokenizer.encode_batch failed");

        // use a mutex to ensure atomic access to the embeddings Vec.
        let embeddings_mutex: Arc<Mutex<Vec<Vec<f32>>>> = Arc::new(Mutex::new(Vec::new()));

        tokenized_sequences.par_iter().for_each(|sequence| {
            let ids: &[u32] = sequence.get_ids();

            // the sentence embeddings, obtained after pooling/averaging
            // all `token_embedding`
            let mut sentence_embedding: Vec<f32> = vec![0.0; self.embedding_dims];
            for id in ids { // for each id in the tokenized sequence
                let token_embedding: &[f32] = self.get_embedding(*id as usize);
                for di in 0..self.embedding_dims {
                    sentence_embedding[di] += token_embedding[di];
                }
            }
            for di in 0..self.embedding_dims {
                sentence_embedding[di] /= ids.len() as f32;
            }
            embeddings_mutex.lock().unwrap().push(sentence_embedding);
        });

        let embeddings = embeddings_mutex.lock().unwrap().clone();
        Ok(embeddings)
    }

    fn get_embedding(&self, index: usize) -> &[f32] {
        // slice the raw embeddings data
        let embedding_raw: &[u8] = &self.embeddings_u8
            [index * self.embedding_dims * 4..(index + 1) * self.embedding_dims * 4];

        // cast embedding_raw to a *const f32 to parse as a float-array
        // instead of u8 array
        let len: usize = embedding_raw.len();
        let ptr: *const f32 = embedding_raw.as_ptr() as *const f32;
        unsafe { std::slice::from_raw_parts(ptr, len / 4) }
    }
}

```

The `index` of the embedding in the raw data is its token ID. We obtain the slice from the raw data and cast it to a `*const f32` to get a `f32` array.

Let's write some code to get embeddings in `main.rs`,
```rust
mod static_model;

use anyhow::{Ok, Result};
use static_model::StaticModel;

fn main() -> Result<()> {
    let static_model = StaticModel::new("embeddings.safetensors", "tokenizer.json")?;
    let inputs = vec![
        String::from("It's dangerous to go alone!"),
        String::from("It's a secret to everybody."),
    ];
    let embeddings = static_model.encode(&inputs, 4)?;
    for embedding in embeddings {
        println!("{:?}", embedding);
    }
    Ok(())
}
```
These are the same sentences included by the authors of Model2Vec in the [`README.md`](https://github.com/MinishLab/model2vec/blob/main/README.md#quickstart) of the project. On running the executable with `cargo run`,
```
[-6.062381, -1.5545515, -5.0984917, 2.099059, 0.6878451, 1.9142252, -2.805557, 0.9798312, 0.5798943, 1.8842146, -2.9659731, 1.3618052, 1.1411309, -0.66664845, 2.61916, -1.3473017, 3.8661003, 1.0561283, -0.11547451, 2.4597473, 0.86268514, 0.57152665, -1.8901768, -1.4163082, -0.6279257, -0.65686285, 0.88550055, 1.6368501, 0.5986386, -0.12917002, -0.16199933, 1.7613158, 1.359037, -2.609987, 0.45865658, 1.1723642, 3.0539, -1.6516464, -0.06350985, 0.8703271, 0.27223834, 0.5241103, -0.63625324, 1.1554337, -0.5713408, 0.80762815, 2.5786963, 0.08609018, -0.5585427, -0.15418807, -0.5366878, -1.0179839, 1.6285973, -1.3136624, 0.2147216, 1.2835652, -1.698041, -1.5122787, 1.5071486, -1.5159504, 0.07437308, 0.9576184, -1.1952554, -1.0150384, 0.07655884, -2.256091, 1.0543764, 0.95232487, 2.8377798, 0.0918544, 0.53704554, 0.7783982, 1.4082336, -0.038419887, -0.18429558, -0.7649706, 1.0153078, 0.3931616, -0.59539235, 2.5317721, -0.6503701, 0.052558437, -0.26835978, 1.7136263, -2.3584037, 1.4950824, -1.6488646, -1.5430261, 1.4526038, -3.1716123, -1.8124771, 1.3286778, 2.150063, 0.38093346, -0.68572515, -0.36874622, -1.3646579, 1.723564, -1.2197155, 1.6349839, -0.09755337, 0.30339906, -1.3862239, -0.17300001, -0.68491364, -1.3000839, 0.54946804, -1.1279598, 2.8317895, -0.33907315, 1.3647276, 0.02645266, 0.253199, 0.31234154, 0.67832196, -0.6554457, 0.16064125, -2.6008806, 1.1940062, -0.73480797, -1.0374694, 2.5018816, 0.29112464, 0.44804728, 0.2946486, 0.32670557, -1.874754, 0.93468606, 0.3702876, -0.8231077, 0.0025781132, 0.76615685, 0.27371526, -0.1791095, -1.0268447, 0.26974455, 0.53118414, 1.8019879, 2.4131038, 1.0147654, 1.2114263, -1.442251, -0.6819745, 2.5206609, 0.3976662, 0.97252184, 1.7872707, -0.88014424, 0.652184, -0.11817723, 0.021888867, -0.98354065, 0.22938779, 3.7347405, 1.6754321, -3.508242, -2.303398, 0.7339891, 2.2544053, -0.36084908, 0.15955457, -1.3541389, -2.8540068, -1.1680337, 0.11558508, 0.0011304617, -1.3428507, 0.41419068, -0.18025596, 0.6059449, -0.088956565, 0.942282, 0.728949, -0.11537789, 0.13964233, 0.77633667, -0.14567141, -0.07622194, 1.185357, -0.9122021, 0.1589686, -0.16714656, 0.6794307, -0.8326399, -0.6716671, 1.1819272, 0.7107702, -1.5916209, 1.9055943, -0.59011525, 0.3618587, 2.6446857, -0.9702283, 1.6232251, 1.9724056, -0.21293975, 1.0337882, 2.6286082, -2.5753727, -0.9975584, -0.5151176, 0.28342497, 0.88990104, 0.582082, 0.18967497, -1.2473122, 0.8467652, -0.7425349, 0.012134321, 0.37099764, 1.4211557, 0.09102595, -0.3127473, -1.3207991, 1.1240747, -0.7805079, 0.37666726, -1.9092792, 0.8678582, 0.005088061, -2.165389, -0.5171934, 0.29700917, -0.51977855, 0.8541367, -1.2363023, -0.30734563, -0.85690427, 0.39203992, 0.3321757, -0.83660173, 1.1959959, -0.57259166, 0.75185806, -0.26735374, 0.2545839, -0.0029066354, 1.0669718, 0.04697463, -0.7108304, -0.9279256, 0.6714646, 0.048637867, -0.8352596, 0.19012898, 0.8764433, 0.1408397, -0.20475186, 0.6068979, -0.89105505, 1.1341738, -0.54343957, 1.8856088, 1.0558753, 0.1704104, 0.0724985]

[-6.7709017, -1.7427853, -0.6825185, -1.2152479, 1.9106612, 1.309958, -0.71968544, -1.2013556, 1.9377314, 5.085854, -2.5336845, 1.5957739, -1.3834323, 0.31336504, 4.094859, 1.1188592, 3.1228871, 0.9327519, 0.3729151, 2.4226913, 3.7890875, -0.7222133, -1.8862313, -0.47458547, -0.2135025, 0.2872669, -0.17230195, 0.37359363, -3.0385203, -1.0574137, 1.5955627, -1.3514857, 2.0800862, 0.19144303, 0.03821115, 2.5536313, 1.587122, -2.6337285, -2.4472027, 0.69117314, 0.15786918, 1.021182, -1.5274317, -1.2304488, -0.9320693, 0.78526, 1.9650898, 0.14898002, -0.31987607, -2.8111129, -0.27600247, -2.482934, 0.31992894, 0.9268683, 0.60779124, 0.6769286, -1.1252868, -0.10893522, 1.583445, -1.684909, 2.5377135, -0.34598076, -0.035452366, -2.2760117, -1.3306732, -1.9392102, -0.4559238, -0.4427015, 1.91725, 0.09187974, 1.2816904, 1.5855871, -0.67267936, 0.19437376, -0.37785286, -0.6873005, 1.2155861, 0.7973621, 0.34811163, 2.0692267, 0.5726788, 0.24627843, 0.39943683, -2.2146764, -0.47445166, -1.8044326, -2.4458656, -0.9955376, 1.9761384, -2.1275027, -1.5075816, 1.4590108, 3.2647738, -1.5955731, -1.2645248, 0.8024656, -1.15804, -0.41567, -1.5341094, 1.9647448, -1.6284541, 2.480859, -0.5656401, 0.6234371, -2.8440108, -0.85672176, -0.1775991, -0.28636375, 3.3037713, -1.0519713, 2.5383265, 1.0486845, -0.8530821, -0.026071567, -0.06542225, -0.70431876, -0.90636855, -1.1598448, 0.5338694, 1.1182172, -2.588242, 2.1212413, -1.0706013, 0.28512943, 1.4060966, 0.030680805, -0.8223305, 0.14657708, -0.45788208, 0.8524926, -1.4439392, 1.6334343, 1.5376487, 1.0941782, 0.208169, -0.71627706, 0.7416445, -1.0906687, 0.15202078, 0.82874846, 1.9824876, -1.6938905, -0.3673666, 0.44611537, -0.12016955, 1.1073523, -0.07328738, -1.2971697, 1.888833, -1.1386307, 0.5085306, -1.6601198, -1.6068884, 2.363585, 2.1150498, -2.906581, -1.6668851, -0.03533417, 2.3731391, -1.4015625, -0.64529705, -1.189023, -1.6940054, 0.9298807, 0.47311282, -0.91809916, -0.96908075, -0.4989406, -0.223361, -0.81780994, -1.8786337, -0.48931906, 1.6521071, 1.1645827, 0.4577713, -0.6927124, -1.279851, -0.69219416, -0.19899127, -1.190292, 1.1515251, 0.40580997, 1.2623837, 0.72085184, 0.71631914, 1.8757875, 0.8096778, 1.5118188, 0.7785634, -1.2907723, 1.0201141, 1.6203743, -2.019534, -0.36370814, 0.80062735, -0.6610102, 0.7638037, 1.0461193, -3.1233761, -1.5993708, 0.56475323, 0.9712727, 1.3132973, 0.8402577, 1.9719452, 0.8027752, -0.07514961, -1.2904321, -0.6096111, 0.15057074, 1.2230872, 0.38019362, 0.0116705, -1.3957986, 1.5922043, 0.66761446, 0.43974298, -0.7907469, 0.681283, 0.8896071, -0.8054414, 0.2600738, 1.1196597, -0.41986942, -0.416831, -1.2588446, 0.35978705, -0.23960058, -0.03143947, 0.10429024, 1.1214565, 0.7776127, 0.5022755, 1.1390584, -0.3844561, 0.080986775, -0.43193513, -0.6183974, 0.12137005, -0.12838833, -1.6376299, 0.23406458, -0.25467387, -1.8697625, 0.6247519, 2.0806153, 0.67795646, 0.5341332, -1.0374103, -0.4080578, 0.61277854, 0.935323, 0.8713538, 0.79788315, 0.20516403, -0.8103115]
```
The output is identical to the Python snippet used in the `README.md`,
```python
from model2vec import StaticModel

# Load a model from the HuggingFace hub (in this case the potion-base-8M model)
model = StaticModel.from_pretrained("minishlab/potion-base-8M")

# Make embeddings
embeddings = model.encode(["It's dangerous to go alone!", "It's a secret to everybody."])
print(embeddings)
```





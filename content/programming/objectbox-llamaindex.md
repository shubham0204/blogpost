---
title: Building RAG Apps with the ObjectBox LlamaIndex Integration
draft: false
tags:
  - programming
  - vector_databases
  - objectbox
  - rag
---
## Introduction
### What s RAG and why is it useful?

In the rapidly evolving landscape of AI and machine learning, Retrieval-Augmented Generation (RAG) has emerged as a groundbreaking approach that combines the best of two worlds: retrieval-based models and generative AI. It allows AI models to augment their responses by dynamically retrieving relevant information from external sources—ensuring responses are both contextually rich and grounded in facts. The input data is represented as vectors or embeddings that capture the semantic meaning of the data. These vectors can also be compared with other embeddings, thus quantifying the similarity between the data instances. Vector databases provide specialized indexes that enable faster retrieval of vectors similar to a given query vector, thus finding information that is most relevant to the given query. 

Read more about vector databases and semantic search here: [https://objectbox.io/vector-search-making-sense-of-search-queries/](https://objectbox.io/vector-search-making-sense-of-search-queries/)

### LlamaIndex and ObjectBox
LlamaIndex (formerly known as GPT Index) is a key tool that supercharges RAG implementations by making it easy to connect language models with your own data. Whether you're dealing with massive databases, niche domains, or proprietary knowledge repositories, LlamaIndex simplifies the process of building a custom retrieval system tailored to your needs. It acts as a bridge between your data and the generative model, enabling highly customized, domain-specific responses. 

ObjectBox is a complete on-device NoSQL database with vector indexing capabilities built-in, available in all popular languages like Java, Python, C++, Go etc. It offers low-latency, offline availability, and efficient resource usage, making it ideal for mobile AI systems. Integrating ObjectBox with LlamaIndex allows developers to inherit all of ObjectBox's merits, particularly its cross-platform APIs and high-performance capabilities.

Continue reading on how on-device vector databases could be useful for local AI applications here: [https://objectbox.io/python-on-device-vector-and-object-database-for-local-ai/](https://objectbox.io/python-on-device-vector-and-object-database-for-local-ai/)

The [LlamaIndex](https://www.llamaindex.ai/) Python package now supports adding [ObjectBox](https://objectbox.io/) as a vector-database in your existing RAG pipelines. The [llama-index-vector-stores-objectbox](https://pypi.org/project/llama-index-vector-stores-objectbox/)  package will allow AI developers to include a fast, efficient on-device vector-store directly in their LlamaIndex RAG applications with minimal changes required. 

Check the LlamaIndex integration package here: [https://github.com/run-llama/llama_index/tree/main/llama-index-integrations/vector_stores/llama-index-vector-stores-objectbox](https://github.com/run-llama/llama_index/tree/main/llama-index-integrations/vector_stores/llama-index-vector-stores-objectbox)

Check the example Jupyter notebook here: [https://github.com/run-llama/llama_index/blob/main/docs/docs/examples/vector_stores/ObjectBoxIndexDemo.ipynb](https://github.com/run-llama/llama_index/blob/main/docs/docs/examples/vector_stores/ObjectBoxIndexDemo.ipynb)

## Getting Started
Install the package from PyPI via `pip`, 
```
pip install llama-index-vector-stores-objectbox
```
The package contains the `ObjectBoxVectorStore` class which is a LlamaIndex-supported vector-store, along with the `objectbox` module that provides low-level control over the ObjectBox database.

You can import the ObjectBox vector-store with from `llama_index.vector_stores.objectbox` import `ObjectBoxVectorStore` and start using it,
```python
from llama_index.vector_stores.objectbox import ObjectBoxVectorStore
from objectbox import VectorDistanceType

embedding_dim = 384  # size of the embeddings to be stored

vector_store = ObjectBoxVectorStore(
    embedding_dim,
    distance_type=VectorDistanceType.COSINE,
    db_directory="obx_data",
    clear_db=False,
    do_log=True,
)
```

- `embedding_dim` (required): The dimensions of the embeddings that the vector DB will hold
- `distance_type`: Choose from `COSINE`, `DOT_PRODUCT`, `DOT_PRODUCT_NON_NORMALIZED` and `EUCLIDEAN` 
- `db_directory`: The path of the directory where the `.mdb` ObjectBox database file should be created
- `clear_db`: Deletes the existing database file if it exists on `db_directory`
- `do_log`: Enables logging from the ObjectBox integration

The ObjectBox `vector_store` can be used as a LlamaIndex `VectorIndex` using `StorageContext`,
```python
from llama_index.core import StorageContext, VectorStoreIndex

storage_context = StorageContext.from_defaults(vector_store=vector_store)

index = VectorStoreIndex(nodes=nodes, storage_context=storage_context)
```
or, as a vector retriever,
```python
retriever = index.as_retriever()
# Get the text chunks that are similar to `<query>`
chunks = retriever.retrieve("<query>")
```
## A Complete RAG example
We implement a simple RAG pipeline that will be used for answering user questions on a set of given documents. The document readers, text splitters will be used from LlamaIndex’s core package whereas [Google’s Gemini](https://gemini.google.com/) API is used as a LLM to generate natural-language answers. To find context relevant to the user’s query from the documents provided, we use an embedding model from [HuggingFace](https://huggingface.co/) and [ObjectBox’s vector search](https://docs.objectbox.io/on-device-vector-search) capabilities to search for the most similar text chunks (context).

Along the `llama-index-vector-stores-objectbox`, install the following packages,
```
pip install llama-index --quiet
pip install llama-index-embeddings-huggingface --quiet
pip install llama-index-llms-gemini --quiet
```
Then, download a sample text document,
```
mkdir -p 'data/paul_graham/'
wget 'https://raw.githubusercontent.com/run-llama/llama_index/main/docs/docs/examples/data/paul_graham/paul_graham_essay.txt' -O 'data/paul_graham/paul_graham_essay.txt'
```
To implement an embedding model for RAG, we can leverage one of HuggingFace's extensive collection of pre-trained embedding models. HuggingFace provides a diverse selection of models that are ranked based on performance metrics and can be found on the [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard). For this example, we will use the [bge-base-en-v1.5](https://huggingface.co/BAAI/bge-base-en-v1.5) embedding model from HF,
```python
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

hf_embedding = HuggingFaceEmbedding(model_name="BAAI/bge-base-en-v1.5")
embedding_dim = 384
```
We use the [SimpleDirectoryReader](https://docs.llamaindex.ai/en/stable/module_guides/loading/simpledirectoryreader/) that selects the best file reader by checking the file extension from the directory.

Next, we produce chunks (text subsequences) from the contents read by the `SimpleDirectoryReader` from the documents. A [SentenceSplitter](https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/modules/#sentencesplitter) is a text-splitter that preserves sentence boundaries while splitting the text into chunks of size `chunk_size`,
```python
from llama_index.core import SimpleDirectoryReader
from llama_index.core.node_parser import SentenceSplitter

reader = SimpleDirectoryReader("./data/paul_graham")
documents = reader.load_data()

node_parser = SentenceSplitter(chunk_size=512, chunk_overlap=20)
nodes = node_parser.get_nodes_from_documents(documents)
```
Next, we configure the ObjectBoxVectorStore as the vector database, and set Gemini as the LLM for the RAG pipeline. You can get an API-key for Gemini from the [console](https://aistudio.google.com/app/apikey),
```python
from llama_index.vector_stores.objectbox import ObjectBoxVectorStore
from llama_index.core import StorageContext, VectorStoreIndex, Settings
from objectbox import VectorDistanceType
from llama_index.llms.gemini import Gemini
import getpass

gemini_key_api = getpass.getpass("Gemini API Key: ")
gemini_llm = Gemini(api_key=gemini_key_api)

vector_store = ObjectBoxVectorStore(
    embedding_dim,
    distance_type=VectorDistanceType.COSINE,
    db_directory="obx_data",
    clear_db=False,
    do_log=True
)

storage_context = StorageContext.from_defaults(vector_store=vector_store)

Settings.llm = gemini_llm
Settings.embed_model = hf_embedding

index = VectorStoreIndex(nodes=nodes, storage_context=storage_context)**
```
Finally, we can ask questions and the RAG pipeline will retrieve the relevant chunks from the ObjectBox database using vector search and provide them to the LLM in the prompt.
```python
query_engine = index.as_query_engine()
response = query_engine.query("Who is Paul Graham?")
print(response)
```

The output is a natural language response:
```
ObjectBox retrieved 2 vectors in 0.0011882740000146441 seconds
INFO:llama_index.vector_stores.objectbox.base:ObjectBox retrieved 2 vectors in 0.0011882740000146441 seconds
Paul Graham is a computer scientist who wrote a book about Lisp hacking called "On Lisp". 
```
## Conclusion
The integration of ObjectBox with LlamaIndex offers a powerful solution for developers looking to enhance their RAG pipelines with on-device, efficient vector storage. By using the `llama-index-vector-stores-objectbox` package, AI developers can seamlessly incorporate ObjectBox as a vector database into their applications, enabling faster and more efficient retrieval of contextual information.

With minimal setup and changes required, developers can combine the speed of ObjectBox's vector search capabilities with the flexibility and power of LlamaIndex's document processing and retrieval functionalities. This setup, paired with HuggingFace’s advanced embedding models and LLMs like Google’s Gemini, allows for the creation of highly effective, on-device, retrieval-augmented generation pipelines. The ObjectBox vector store ensures optimal performance, even in resource-constrained environments, making this integration a robust choice for a wide range of AI applications.

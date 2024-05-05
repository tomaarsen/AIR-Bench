<h1 align="center"> AIR-Bench: Automated Heterogeneous Information Retrieval Benchmark </h1>


- [Background & Motivation](#background--motivation)
- [Features Of AIR-Bench](#features-of-air-bench)
- [Data Generation](#data-generation)
  - [Task Type & Domain & Language](#task-type--domain--language)
  - [Pipeline](#pipeline)
    - [Corpus Preparation](#corpus-preparation)
    - [Query Generation](#query-generation)
    - [Hard Negative Generation](#hard-negative-generation)
    - [Quality Control](#quality-control)
- [Current Dataset](#current-dataset)
- [Evaluation Result](#evaluation-result)
- [Future Work](#future-work)
- [Community Contribution](#community-contribution)


## Background & Motivation

Evaluation is crucial for the development of information retrieval models. In recent years, a series of milestone works have been introduced to the community, such as MSMARCO, Natural Question, (open-domain QA), MIRACL (Milti-lingual retrieval), BEIR and MTEB (general-domain zero-shot retrieval). However, the existing benchmarks are severely limited in the following perspectives.

- **Incapability of dealing with new domains**. All of the existing benchmarks are static, which means they are established for the pre-defined domains based on human labeled data. Therefore, they are incapable of dealing with new domains which are interested by the users. 
- **Potential risk of over-fitting and data leakage**. The existing retrievers are intensively fine-tuned in order to achieve strong performances on popular benchmarks, like BEIR and MTEB. Despite that these benchmarks are initially designed for zero-shot evaluation of O.O.D. Evaluation, the in-domain training data is widely used during the fine-tuning process. What is worse, given the public availability of the existing evaluation datasets, the testing data could be falsely mixed into the retrievers' training set by mistake. 

## Features Of AIR-Bench

The new benchmark is highlighted for the following new features. 

- **Automated**. The testing data is automatically generated by large language models without human intervention. Therefore, it is able to instantly support the evaluation of new domains at a very small cost. Besides, the new testing data is almost impossible to be covered by the training sets of any existing retrievers.
- **Heterogeneous** **and Dynamic**: The testing data is generated w.r.t. diverse and constantly augmented domains and languages (i.e. Multi-domain, Multi-lingual). As a result, it is able to provide an increasingly comprehensive evaluation benchmark for the community developers.  
- **Retrieval and RAG-oriented**. The new benchmark is dedicated to the evaluation of retrieval performance. In addition to the typical evaluation scenarios, like open-domain question answering or paraphrase retrieval, the new benchmark also incorporates a new setting called inner-document retrieval which is closely related with today's LLM and RAG applications. In this new setting, the model is expected to retrieve the relevant chunks of a very long documents, which containts the critical infomration to answer the input question. 

## Data Generation

### Task Type & Domain & Language

Task Types: QA, Long-Doc

Domains: Wiki, Web, Healthcare, Law, Arxiv, News, Finance, Book

Languages: 

- Current: English, Chinese

- Future: Other languages

### Pipeline

In the whole pipeline, we use `gpt-4-1106-preview` as the LLM.

```python
def generate_dataset(corpus, num_queries, task_type):
    # Generate Triplets
    triplets = []
    documents = sample(corpus, n=num_queries)
    for d in documents:
        # Generate Query
        q = generate_query(d)
        # Generate Hard Negative
        if task_type == 'QA':
            hn = generate_hard_negative(q, d)
        else:
            hn = None
        # Add New Triplet
        triplets.append((q, d, hn))

    # Build Dataset
    dataset = build_dataset(corpus, triplets)

    # Quality Control
    new_dataset = quality_control(dataset)
    return new_dataset

def generate_query(document):
    pass

def generate_hard_negative(query, document):
    pass

def quality_control(dataset):
    pass
```

#### Corpus Preparation

For QA task, we use the real-world datasets as the corpus, such as Wikipedia, mC4, etc.

For Long-Doc task, we firstly select the long documents, such as ArXiv papers, books, etc. Then we use the `SimpleNodeParser` from the [LlamaIndex](https://github.com/run-llama/llama_index/tree/main) to split the long document to fixed-size chunks (`chunk_size=200`, `chunk_overlap=50`) as the corpus.

#### Query Generation

Given a document, the pipeline generates a query as follows:

1. Let the LLM generate the characters who will find the document useful.
2. Let the LLM generate the scenarios in which the character may find the document useful.
3. Let the LLM generate the query based on the specific character and scenario.
4. Let the LLM rewrite the query 1~3 times to generate the final query.

#### Hard Negative Generation

Given a query and the positive document, the pipeline generates the hard negatives as follows:

1. Let the LLM generate 3~7 hard negatives for the given query and positive document.

#### Quality Control

Given a dataset, the pipeline will control the quality as follows:

1. Use the embedding model to search top-1000 relevant documents from the corpus for each query.
2. Use multiple rerankers to rerank the top-1000 relevant documents for each query. Then set the rank threshold to filter possible false negatives for each query.
3. Use the LLM as labeler to label the positive and possible false negatives for each query. Filter the queries with false positives, discard false hard negatives. and label the other false negatives as positives. Then the final new dataset comes out.

## Current Dataset

The following table gives you an overview of the tasks in AIR-Bench.

| Task Type | Domain     | Language | Source                                                       | #Query | Avg Char Len of Query | #Pos   | #Corpus   | #Hard Neg |
| --------- | ---------- | -------- | ------------------------------------------------------------ | ------ | --------------------- | ------ | --------- | --------- |
| QA        | Wiki       | English  | [Wikipedia](https://huggingface.co/datasets/NeuML/wikipedia-20240101) (20240101) | 1,727  | 79                    | 4,260  | 6,738,498 | 7,882     |
| QA        | Wiki       | Chinese  | [Wikipedia](https://huggingface.co/datasets/wikipedia) (20240401) | 1,679  | 24                    | 4,745  | 1,161,226 | 6,963     |
| QA        | Web        | English  | [mC4](https://huggingface.co/datasets/mc4)                   | 1,707  | 81                    | 5,543  | 2,459,587 | 7,439     |
| QA        | Web        | Chinese  | [mC4](https://huggingface.co/datasets/mc4)                   | 1,683  | 24                    | 6,250  | 956,699   | 6,721     |
| QA        | News       | English  | [CC-News](https://huggingface.co/datasets/cc_news)           | 1,614  | 83                    | 5,798  | 574,417   | 6,784     |
| QA        | News       | Chinese  | [CC-News](https://huggingface.co/datasets/intfloat/multilingual_cc_news) | 1,697  | 25                    | 7,381  | 935,162   | 6,618     |
| QA        | Healthcare | English  | [PubMedQA](https://huggingface.co/datasets/qiaojin/PubMedQA) | 1,707  | 96                    | 5,052  | 847,395   | 7,023     |
| QA        | Healthcare | Chinese  | [Huatuo-26M](https://huggingface.co/datasets/FreedomIntelligence/huatuo_encyclopedia_qa) (Encyclopedia QA) | 1,874  | 22                    | 10,029 | 360,218   | 7,336     |
| QA        | Law        | English  | [pile-of-law](https://huggingface.co/datasets/pile-of-law/pile-of-law) (EUR-Lex) | 1,801  | 91                    | 5,372  | 141,678   | 6,574     |
| QA        | Law        | Chinese  | [npc.gov.cn](https://flk.npc.gov.cn/index.html)              | 1,816  | 27                    | 6,118  | 25,857    | 4,725     |
| QA        | Finance    | English  | [Reuters-Financial](https://huggingface.co/datasets/reuters21578) | 1,585  | 87                    | 3,357  | 26,266    | 5,595     |
| QA        | Finance    | Chinese  | [FinCorpus](https://huggingface.co/datasets/Duxiaoman-DI/FinCorpus) (fin_articles) | 1,805  | 24                    | 7,836  | 2,398,095 | 7,211     |
| QA        | Arxiv      | English  | [Arxiv](https://github.com/armancohan/long-summarization) (abstract) | 1,731  | 98                    | 5,340  | 222,877   | 6,288     |
| QA        | Web        | English  | [MS MARCO](https://huggingface.co/datasets/intfloat/simlm-msmarco) (dev) | 6,319  | 78                    | 31,447 | 8,872,840 | 26,828    |
| Long-Doc  | Arxiv      | English  | [gpt3](https://arxiv.org/pdf/2005.14165.pdf)                 | 337    | 81                    | 496    | 515       | -         |
| Long-Doc  | Arxiv      | English  | [llama2](https://arxiv.org/pdf/2307.09288.pdf)               | 326    | 93                    | 635    | 566       | -         |
| Long-Doc  | Arxiv      | English  | [llm-survey](https://arxiv.org/pdf/2303.18223.pdf)           | 357    | 94                    | 924    | 1,144     | -         |
| Long-Doc  | Arxiv      | English  | [gemini](https://arxiv.org/pdf/2312.11805.pdf)               | 249    | 95                    | 249    | 276       | -         |
| Long-Doc  | Book       | English  | [origin-of-species_darwin](https://www.vliz.be/docs/Zeecijfers/Origin_of_Species.pdf) | 366    | 89                    | 1,145  | 1,758     | -         |
| Long-Doc  | Book       | English  | [a-brief-history-of-time_stephen-hawking](https://www.docdroid.net/GCLN82v/stephen-hawking-a-brief-history-of-time-pdf) | 370    | 87                    | 876    | 778       | -         |
| Long-Doc  | Healthcare | English  | [pubmed_100K-200K_1](https://github.com/armancohan/long-summarization) | 372    | 94                    | 1,008  | 899       | -         |
| Long-Doc  | Healthcare | English  | [pubmed_100K-200K_2](https://github.com/armancohan/long-summarization) | 355    | 92                    | 980    | 872       | -         |
| Long-Doc  | Healthcare | English  | [pubmed_100K-200K_3](https://github.com/armancohan/long-summarization) | 357    | 93                    | 978    | 873       | -         |
| Long-Doc  | Healthcare | English  | [pubmed_40K-50K_5-merged](https://github.com/armancohan/long-summarization) | 336    | 95                    | 1,046  | 1,731     | -         |
| Long-Doc  | Healthcare | English  | [pubmed_30K-40K_10-merged](https://github.com/armancohan/long-summarization) | 368    | 94                    | 1,485  | 2,154     | -         |
| Long-Doc  | Law        | English  | [lex_files_300K-400K](https://huggingface.co/datasets/lexlms/lex_files) | 339    | 86                    | 1,307  | 2,797     | -         |
| Long-Doc  | Law        | English  | [lex_files_400K-500K](https://huggingface.co/datasets/lexlms/lex_files) | 333    | 87                    | 1,427  | 3,320     | -         |
| Long-Doc  | Law        | English  | [lex_files_500K-600K](https://huggingface.co/datasets/lexlms/lex_files) | 346    | 93                    | 1,324  | 4,087     | -         |
| Long-Doc  | Law        | English  | [lex_files_600K-700K](https://huggingface.co/datasets/lexlms/lex_files) | 338    | 96                    | 1,442  | 5,049     | -         |

## Evaluation Result

Detailed Results are avaliable [here](./avaliable_evaluation_results.md).

Analysis about the results:

- **AIR-Bench performance scales with model size**. For example, `multilingual-e5-large` is better than `multilingual-e5-base` and `multilingual-e5-base` is better than `multilingual-e5-small`. This can also be observed in `bge-large-en-v1.5`, `bge-base-en-v1.5` and `bge-small-en-v1.5`.
- **The generated dataset maintains good consistency with the original dataset**. The Spearman correlation between the rankings on original MSMARCO dataset and the generated MSMARCO dataset is 0.7133.
- **The performance of the model varies across different domains**. For example, `e5-mistral-7b-instruct` is better than `bge-m3` in the healthcare domain, but `e5-mistral-7b-instruct` is worse than `bge-m3` in the law domain.

## Future Work

- More data
- A public evaluation platform for community: TODO

## Community Contribution

We welcome contributions such as corpus in more domains and more languages to AIR-Bench! You can upload the corpus you provide to Huggingface Hub and then open an issue in our repo. In the issue, you should provide:

- The source of the new corpus, such as link or citing.
- The domain and language of the new corpus.
- The huggingface link to download the new corpus.

The file format of the new corpus should be a jsonl file, where each line is a dict like this:

```python
{"text": str}
```


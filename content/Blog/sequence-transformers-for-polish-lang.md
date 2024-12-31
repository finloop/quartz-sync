---
title: "Sequence Transformers for Polish Lang"
date: 2022-08-06T00:00:02+02:00
publish: true
---

In this tutorial, I'll show you how to generate embeddings for sequences in polish using Sequence Transformers. I won't explain how they work, there are many great articles:

- [Measuring Text Similarity Using BERT
  ](https://www.analyticsvidhya.com/blog/2021/05/measuring-text-similarity-using-bert/)
- [Bert 101](https://huggingface.co/blog/bert-101)

What we'll need is a Sequence Transformers library from Huggingface:

```sh
pip install sequence_transformers
```

The code is simple, we import library, create model and ask it for embeddings.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('Voicelab/sbert-base-cased-pl')
embeddings = model.encode(["Ten tekst zostanie zakodowany"])
print(embeddings)
```

And that's it. This is the output of the model:

```
[[ 7.74895132e-01  7.00104088e-02 -5.02209544e-01 -2.06187874e-01
  -1.28363922e-01  1.18705399e-01 -1.88303709e-01 -9.09971595e-02
...
```

If you wanted to change the model change `Voicelab/sbert-base-cased-pl` to a model from [this](https://huggingface.co/models?language=pl&pipeline_tag=sentence-similarity&sort=downloads) list, it's pre-filtered for Polish language.

Those embeddings can be pretty useful, as we could use them for classification, similarity search etc.

## Example of usage

I have a list of sentences. I want to know which ones are the most similar. How could I do that? As you can guess – with embeddings. We'll calculate a distance matrix for each sentence and look which are the most similar.

```python
sentences = [
"Pożar w mieście. Zgnięło 10 osób."
,"Wypadek pod wiaduktem kolejowym."
,"W Poniedziałek odbędzie się konferencja naukowa"
,"Magia potrafi wzniecać pożary"]

embeddings = model.encode(sentences)
```

I'll use cosine distance as measure of similarity.

```
from sklearn.metrics import pairwise

sns.heatmap(pairwise.cosine_similarity(embeddings, embeddings))
```

![Heatmap of distance matrix](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m8rdn3mhl8b2reftklsj.png)

From this heatmap we can deduce that our model works, it found similarity between sentences with `pożar` and `wypadek` which both refer to an accident.

_Pracę przygotowano w ramach realizacji projektu pt.: „Hackathon Open Gov Data oraz stworzenie innowacyjnych aplikacji, z wykorzystaniem technologii GPU”, dofinansowanego przez Ministra Edukacji i Nauki ze środków z budżetu państwa
w ramach programu „Studenckie koła naukowe tworzą innowacje”._

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/89b9olhqzqty9moz6kzg.png)

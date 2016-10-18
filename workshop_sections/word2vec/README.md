# Building a Custom Estimator for word2vec

This walkthrough assumes that your working directory is that of this `README` file. 

## Getting set up

First make sure you have installed TensorFlow, numpy and nltk in your python environment.

Previous workshop sections have instructions on how to install TensorFlow and numpy. To install nltk:

```
pip install nltk
python -c "import nltk; nltk.download('punkt')"
```
(Note: if you are in using a Docker image this is already done)

This will give you the necessary corpuses to use `nltk.word_tokenize`.

### Downloading the training data

The training data can be found at [http://mattmahoney.net/dc/text8.zip](http://mattmahoney.net/dc/text8.zip) (25 MB compressed, ~100 MB uncompressed)

The rest of this tutorial assumes you have downloaded this data and unzipped it in this directory, you can do this with:

```sh
curl http://mattmahoney.net/dc/text8.zip -o text8.zip
unzip text8.zip
```

## Preproccessing Data

All the following commands are run in a python interpreter. So start up a python intepreter with 

```sh
python
```

or copy paste into Jupyter cells.

### In memory with Numpy

```
import word2vec
with open('text8', 'r') as f:
    index, word_indices = word2vec.build_string_index(f.read())
```
(this may take ~1 minute)

`word2vec.build_string_index` reads in a string tokenizes it using `nltk.word_tokenize` and builds

 * `index`: A `np.ndarray` that lists all unique words from most common to least common (+ `'UNK'` a token used to denote words which do not appear often enough in our fixed size vocabulary)`

 * `word_indices`: A `np.ndarray` that replaces all the words in the string with their index in `index`. 

So `index[word_indices]` reconstructs the list, with less common words replaced by `UNK`.

```
targets, contexts = word2vec.generate_batches(word_indices)
```
(this may take 5-10 minutes)

`word2vec.generate_batches` applies SkipGram processing to rolling windows in `word_indices`. The first element in the returned tuple is a vector of target word indices, while the second is a vector of the context word indices we'd like to predict.

### From TFRecords

If you don't want to do preprocessing in memory, or you want to preprocess via another distributed graph computation framework, Tensorflow supports reading from a binary (protobuf) format called TFRecords. We have provided this file publicly available online. See below for how to train with this data.

## Training the model

To define your model, run:

```
import tensorflow as tf
import os
import time
workshop_root = os.path.dirname(os.path.dirname(os.getcwd()))
output_dir = os.path.join(workshop_root, 'workshop-data', 'output', str(int(time.time())))
tf.logging.set_verbosity(tf.logging.INFO)
word2vec_model = tf.contrib.learn.Estimator(model_fn=word2vec.make_model_fn(), model_dir=output_dir)
```

To train with the in memory numpy arrays, run:

```
word2vec_model.fit(x=targets, y=contexts, batch_size=256, max_steps=50000)
```

To train with the pre-preprocessed TFRecords, run:
```
word2vec_model.fit(input_fn=word2vec.input_from_files(word2vec.GCS_SKIPGRAMS, 256), max_steps=50000)
```

In another shell (or outside your docker container), run

```sh
tensorboard --logdir wordshop
```

to watch the training outputs.

## Using the model to predict

This word2vec model predicts similarities between words using it's trained embeddings.

When given a vector of word_indices, the word2vec model outputs a prediction dictionary with two values: `'predictions'` are the indices of the words most similar to the provided words, while `'values'` is a measure of their similarity to the provided words. You can use `index` to convert these word indices back into strings.

NOTE: if you used TFRecords for training and as such don't have an `index` you can download it [here](https://storage.googleapis.com/oscon-tf-workshop-materials/stringindex.npy), and load it from this file using:

```
import numpy as np
index = np.load('stringindex.npy')
```

To predict, simply input a pandas dataframe or numpy array into `word2vec_model.predict`. The `num_sim` parameter in the original `make_model_fn` call controls the number of similar words we predict for each word in the input vector. You can make a new estimator object without retraining as long as you specify the same `model_dir` as your trained model. Trained parameters will be loaded from the most recent checkpoint!


For example, predicting the 5 most common known words:

```
index[word2vec_model.predict(x=np.array(range(1, 6)))['predictions']]
```

Or predicting a specific word, like:

```
index[word2vec_model.predict(x=np.where(index == 'son')[0])['predictions']]
```

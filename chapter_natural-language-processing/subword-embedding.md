# Subword Embedding
:label:`sec_fasttext`

English words usually have internal structures and formation methods. For example, we can deduce the relationship between "dog", "dogs", and "dogcatcher" by their spelling. All these words have the same root, "dog", but they use different suffixes to change the meaning of the word. Moreover, this association can be extended to other words. For example, the relationship between "dog" and "dogs" is just like the relationship between "cat" and "cats". The relationship between "boy" and "boyfriend" is just like the relationship between "girl" and "girlfriend". This characteristic is not unique to English. In French and Spanish, a lot of verbs can have more than 40 different forms depending on the context. In Finnish, a noun may have more than 15 forms. In fact, morphology, which is an important branch of linguistics, studies the internal structure and formation of words.


## fastText

In word2vec, we did not directly use morphology information.  In both the
skip-gram model and continuous bag-of-words model, we use different vectors to
represent words with different forms. For example, "dog" and "dogs" are
represented by two different vectors, while the relationship between these two
vectors is not directly represented in the model. In view of this, fastText :cite:`Bojanowski.Grave.Joulin.ea.2017`
proposes the method of subword embedding, thereby attempting to introduce
morphological information in the skip-gram model in word2vec.

In fastText, each central word is represented as a collection of subwords. Below we use the word "where" as an example to understand how subwords are formed. First, we add the special characters “&lt;” and “&gt;” at the beginning and end of the word to distinguish the subwords used as prefixes and suffixes. Then, we treat the word as a sequence of characters to extract the $n$-grams. For example, when $n=3$, we can get all subwords with a length of $3$:

$$\textrm{"<wh"}, \ \textrm{"whe"}, \ \textrm{"her"}, \ \textrm{"ere"}, \ \textrm{"re>"},$$

and the special subword $\textrm{"<where>"}$.

In fastText, for a word $w$, we record the union of all its subwords with length of $3$ to $6$ and special subwords as $\mathcal{G}_w$. Thus, the dictionary is the union of the collection of subwords of all words. Assume the vector of the subword $g$ in the dictionary is $\mathbf{z}_g$. Then, the central word vector $\mathbf{u}_w$ for the word $w$ in the skip-gram model can be expressed as

$$\mathbf{u}_w = \sum_{g\in\mathcal{G}_w} \mathbf{z}_g.$$

The rest of the fastText process is consistent with the skip-gram model, so it is not repeated here. As we can see, compared with the skip-gram model, the dictionary in fastText is larger, resulting in more model parameters. Also, the vector of one word requires the summation of all subword vectors, which results in higher computation complexity. However, we can obtain better vectors for more uncommon complex words, even words not existing in the dictionary, by looking at other words with similar structures.


## Byte Pair Encoding

In fastText, all the extracted subwords have to be of the specified lengths, such as $3$ to $6$, thus the vocabulary size cannot be predefined.
To allow for variable-length subwords in a fixed-size vocabulary,
we can apply a compression algorithm
called *byte pair encoding* (BPE) to extract subwords :cite:`Sennrich.Haddow.Birch.2015`.
Starting from symbols of length $1$,
BPE iteratively merges the frequent pair of symbols, such as consecutive characters of
arbitrary length within a word, to produce new symbols.
Note that for efficiency, pairs crossing word boundaries are not considered.
In the end, we can use such symbols as subwords to segment words.
In the following, we will illustrate how BPE works.

First, we initialize the vocabulary of symbols as all the English characters, a special end-of-word symbol `'_'`, and a special unknown symbol `'[UNK]'`.

```{.python .input}
import collections

symbols = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm',
           'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z',
           '_', '[UNK]']
```

Since we do not consider symbol pairs that cross boundaries of words,
we only need a dictionary `raw_token_freqs` that maps words to frequencies as the representation of a dataset.
Note that the special symbol `'_'` is appended to each word so that
we can easily convert a sequence of symbols ( e.g., "a_ tall er_ man")
to its corresponding word sequence (e.g., "a taller man").
Since we start the merging process from a vocabulary of only single characters and special symbols, space is inserted between every pair of consecutive characters within each word.

```{.python .input}
raw_token_freqs = {'fast_': 4, 'faster_': 3, 'tall_': 5, 'taller_': 4}
token_freqs = {}
for token, freq in raw_token_freqs.items():
    token_freqs[' '.join(list(token))] = raw_token_freqs[token]
token_freqs
```

```{.python .input}
def get_max_freq_pair(token_freqs):
    pairs = collections.defaultdict(int)
    for token, freq in token_freqs.items():
        symbols = token.split()
        for i in range(len(symbols) - 1):
            # Key of pairs is a tuple composed of two adjacent units
            pairs[symbols[i], symbols[i + 1]] += freq
    return max(pairs, key=pairs.get)  # Key of pairs with the max value
```

```{.python .input}
def merge_vocab(max_freq_pair, token_freqs, symbols):
    symbols.append(''.join(max_freq_pair))
    new_token_freqs = {}
    for token, freq in token_freqs.items():
        new_token = token.replace(' '.join(max_freq_pair),
                                  ''.join(max_freq_pair))
        new_token_freqs[new_token] = token_freqs[token]
    return new_token_freqs
```

```{.python .input}
num_merges = 10
for i in range(num_merges):
    max_freq_pair = get_max_freq_pair(token_freqs)
    token_freqs = merge_vocab(max_freq_pair, token_freqs, symbols)
    print("Merge #%d:" % (i + 1), max_freq_pair)
```

```{.python .input}
print(symbols)
```

```{.python .input}
print("Original tokens:", list(raw_token_freqs.keys()))
print("Tokens with BPE segmentation:", list(token_freqs.keys()))
```

```{.python .input}
inputs = ['tallest_', 'fatter_']
outputs = []
for word in inputs:
    start, end = 0, len(word)
    cur_output = []
    while start < len(word) and start < end:
        if word[start : end] in symbols:
            cur_output.append(word[start : end])
            start = end
            end = len(word)
        else:
            end -= 1
    if start < len(word):
        cur_output.append('[UNK]')
    outputs.append(' '.join(cur_output))
print('Words:', inputs)
print('Wordpieces:', outputs)
```

## Summary

* FastText proposes a subword embedding method. Based on the skip-gram model in word2vec, it represents the central word vector as the sum of the subword vectors of the word.
* Subword embedding utilizes the principles of morphology, which usually improves the quality of representations of uncommon words.


## Exercises

1. When there are too many subwords (for example, 6 words in English result in about $3\times 10^8$ combinations), what problems arise? Can you think of any methods to solve them? Hint: Refer to the end of section 3.2 of the fastText paper[1].
1. How can you design a subword embedding model based on the continuous bag-of-words model?
1. To get a vocabulary of size $m$, how many merging operations are needed when the initial symbol vocabulary size is $n$?



## [Discussions](https://discuss.mxnet.io/t/2388)

![](../img/qr_subword-embedding.svg)
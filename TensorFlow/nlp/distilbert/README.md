# DistilBERT

For more information about training deep learning models on Gaudi, visit [developer.habana.ai](https://developer.habana.ai/resources/).

## Table of Contents

   * [Model Overview](#model-overview)
   * [Setup](#setup)
   * [Fine-Tuning with SQuAD](#fine-tuning-with-squad)
   * [Change Log](#change-log)

## Model Overview

The [DistilBERT model](https://huggingface.co/distilbert-base-uncased) was proposed in the blog post [Smaller, faster, cheaper, lighter: Introducing DistilBERT, a distilled version of BERT](https://medium.com/huggingface/distilbert-8cf3380435b5), and the paper [DistilBERT, a distilled version of BERT: smaller, faster, cheaper and lighter](https://arxiv.org/abs/1910.01108) by HuggingFace. DistilBERT is a small, fast, cheap and light Transformer model trained by distilling BERT base.

The DistilBERT model is a distilled version of the BERT base model, a 12-layer, 768-hidden, 12-heads, 110M parameter neural network architecture, trained on the BooksCorpus with 800M words, and a version of the English Wikipedia with 2,500M words.

The DistilBERT scripts reuse part of [BERT scripts](../bert) with necessary changes, for the details on changes, go to [CHANGES.md](./CHANGES.md). For now, the scripts only contain fine-tuning scripts taken from BERT scripts and using the DistilBERT pre-trained model from Huggingface transformers. We converted the training scripts to TensorFlow 2, added Habana device support and wrapped horovod import with a try-catch block so that the user is not required to install this library when the model is being run on a single card.

Please visit [this page](../../../README.md#tensorflow-model-performance) for performance information.

## Setup

Please follow the instructions given in the following link for setting up the
environment including the `$PYTHON` environment variable: [Gaudi Installation
Guide](https://docs.habana.ai/en/latest/Installation_Guide/GAUDI_Installation_Guide.html).
This guide will walk you through the process of setting up your system to run
the model on Gaudi.

## Fine-Tuning with SQuAD

- Located in: `Model-References/TensorFlow/nlp/distilbert`
- The main training script is **`run_squad.py`**.
  - **`run_squad.py`**: script implementing finetuning with SQuAD.
- Suited for tasks:
  - `squad`: Stanford Question Answering Dataset (**SQuAD**) is a reading comprehension dataset, consisting of questions posed by crowdworkers on a set of Wikipedia articles, where the answer to every question is a segment of text, or span, from the corresponding reading passage, or the question might be unanswerable.
- Uses optimizer: **AdamW** ("ADAM with Weight Decay Regularization").
- Based on model weights trained with pretraining, the script will download the pretrained model from HuggingFace transformers.
- Light-weight: the training takes a minute or so.

### Install Model Requirements

In the docker container, go to the DistilBERT directory
```bash
cd /root/Model-References/TensorFlow/nlp/distilbert
```
Install required packages using pip
```bash
$PYTHON -m pip install -r requirements.txt
```

### Download the datasets for Fine-Tuning

The SQuAD dataset needs to be manually downloaded to a location of your choice, preferably a shared directory.
The [SQuAD website](https://rajpurkar.github.io/SQuAD-explorer/) does not seem to link to the v1.1 datasets any longer,
but the necessary files can be found here:
- [train-v1.1.json](https://rajpurkar.github.io/SQuAD-explorer/dataset/train-v1.1.json)
- [dev-v1.1.json](https://rajpurkar.github.io/SQuAD-explorer/dataset/dev-v1.1.json)

### Fine-Tuning with run_squad.py

#### Pre-requisites

Please run the following additional setup steps before calling `run_squad.py`.

- Set the PYTHONPATH:
```bash
cd /path/to/Model-References/TensorFlow/nlp/distilbert/
export PYTHONPATH=../../../:$PYTHONPATH
```

- Download the pretrained model for the BERT Base and Huggingface DistilBERT model variant:
```bash
$PYTHON download/download_pretrained_model.py \
        "https://storage.googleapis.com/bert_models/2020_02_20/" \
        "uncased_L-12_H-768_A-12" \
        "distilbert-base-uncased" \
        False
```

#### Example for single card

```bash
cd /path/to/Model-References/TensorFlow/nlp/distilbert/

$PYTHON ./run_squad.py \
        --vocab_file=uncased_L-12_H-768_A-12/vocab.txt \
        --distilbert_config_file=uncased_L-12_H-768_A-12/distilbert_config.json \
        --init_checkpoint=uncased_L-12_H-768_A-12/distilbert-base-uncased.ckpt-1 \
        --bf16_config_path=distilbert_bf16.json \
        --do_train=True \
        --train_file=/data/tensorflow/bert/SQuAD/train-v1.1.json \
        --do_predict=True \
        --predict_file=/data/tensorflow/bert/SQuAD/dev-v1.1.json \
        --do_eval=True \
        --train_batch_size=32 \
        --learning_rate=5e-05 \
        --num_train_epochs=2 \
        --max_seq_length=384 \
        --doc_stride=128 \
        --output_dir=/root/tmp/squad_distilbert/ \
        --use_horovod=false
```

#### Example for multiple cards using mpirun

Users who prefer to run multi-card fine-tuning by directly calling `run_squad.py` can do so using the `mpirun` command and passing the `--use_horovod` flag to these scripts.

-  8 Gaudi cards finetuning of DistilBERT base in bfloat16 precision using SQuAD dataset:

*<br>mpirun map-by PE attribute value may vary on your setup and should be calculated as:<br>
socket:PE = floor((number of physical cores) / (number of gaudi devices per each node))*

```bash
cd /path/to/Model-References/TensorFlow/nlp/distilbert/

mpirun --allow-run-as-root \
       --tag-output \
       --merge-stderr-to-stdout \
       --output-filename /root/tmp/distilbert_log/ \
       --bind-to core \
       --map-by socket:PE=7 \
       -np 8 \
       $PYTHON ./run_squad.py \
                --vocab_file=uncased_L-12_H-768_A-12/vocab.txt \
                --distilbert_config_file=uncased_L-12_H-768_A-12/distilbert_config.json \
                --init_checkpoint=uncased_L-12_H-768_A-12/distilbert-base-uncased.ckpt-1 \
                --bf16_config_path=distilbert_bf16.json \
                --do_train=True \
                --train_file=/data/tensorflow/bert/SQuAD/train-v1.1.json \
                --do_predict=True \
                --predict_file=/data/tensorflow/bert/SQuAD/dev-v1.1.json \
                --do_eval=True \
                --train_batch_size=32 \
                --learning_rate=2e-04 \
                --num_train_epochs=2 \
                --max_seq_length=384 \
                --doc_stride=128 \
                --output_dir=/root/tmp/squad_distilbert/ \
                --use_horovod=true
```

## Supported Configuration

| Device | SynapseAI Version | TensorFlow Version(s)  |
|:------:|:-----------------:|:-----:|
| Gaudi  | 1.4.1             | 2.8.0 |
| Gaudi  | 1.4.1             | 2.7.1 |

## Change Log

### 1.3.0
* Change `python` or `python3` to `$PYTHON` to execute correct version based on environment setup.
### 1.4.0
* References to custom demo script were replaced by community entry points in README
* Import horovod-fork package directly instead of using Model-References' TensorFlow.common.horovod_helpers; wrapped horovod import with a try-catch block so that the user is not required to install this library when the model is being run on a single card
* remove setup_jemalloc from demo_distilbert.py
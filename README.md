# Coming soon 🔜 ❓

## Project Details
Question generation is the task of automatically generating questions from a text paragraph. The most straight-forward way for this is answer aware question generation which is the reverse task of QA. In answer aware question generation the model is presented with a answer and the passage and asked to generate a question for that answer by considering the passage context. While there are many papers available for QG, it's still not as mainstream as QA. One of the reasons is most of the earlier papers use complicated models/processing and have no pre-trained models available. Few recent papers, specifically UniLM and ProphetNet have SOTA pre-trained weights availble for QG but the usage seems quite complicated. 

This project is aimed as an open source study on question generation with pre-trained transformers (specifically seq-2-seq models) using straight-forward end-to-end methods without much complicated pipelines. The goal is to provide simplified data processing and training scripts and easy to use pipelines for inference.

One of the obivious benifit of question generation models is that it can allow the creation of synthetic QA corpora for training QA models. 

### Initial experiments
Initial experiments are conducted using the SQuADv1 dataset and T5 model with different input processing format as described below.
**1. answer aware question generation**

For answer aware models the input text can be processed in two ways.
1. prepend format
 Here the answer is simply added before the conext and seperated by sep token. For example

 `42 [SEP] 42 is the answer to life, the universe and everything.`

 for T5 model the input is processed like this

 answer: 42  context: 42 is the answer to life, the universe and everything.

2. highlight format
In this format the answer span is highlighted within the text with special highlight tokens.

`<hl> 42 <hl> is the answer to life, the universe and everything.`

**answer extraction models**
As the answer aware models need answers for generating question, we need something which can extarct answer like spans from the text. This can be done using various methods like NER, noun-phrase extarction etc. But here a model is trained to extract answer like spans, to see how it'll work. For answer extraction. With T5, answer extarction is done using the text-to-format. 

As the highlight format will need to know the position of extracted answer spans the input for answer extraction is processed as followes-
    1. split the text into senteces 
    2. for each sentence that has answers, highlight the sentence with <hl> tokens.
    3. for the target text join the answers in that sentence with <sep> tokens.

For example for this text 

`Python is a programming language. Created by Guido van Rossum and first released in 1991.` 

following examples will be created

Input text:

<hl> Python is a programming language. <hl> Created by Guido van Rossum and first released in 1991.

target text:
Python <sep>

and 

Input text:

Python is a programming language. <hl> Created by Guido van Rossum and first released in 1991 <hl>.

target text:
Guido van Rossum <sep> 1991 <sep>

At inference time the text is split into sentences and each sentence is highlighted.

**2. Multitask QA-QG**
For answer aware que generation we usually need 3 models, first which will extract answer like spans, second model will generate question on that answer and third will be a QA model which will take the question and produce an answer,
then we can compare the two answers to see if the generated question is correct or not.

Having 3 models for single task is lot of complexity, so goal is to create a multi-task model which can do all of these 3 tasks

1. extract ans like spans
2. generate question based on the answer
3. QA

T5 model is fine-tuned in multi-task way using task prefixes as described in the paper.

![t5-qa-qg](https://i.ibb.co/THk4tm5/t5-qa-qg.png)


## Results

|Name                 |BLEU-4 |METEOR |ROUGE-L|QA-EM |QA-F1 |QG-FORMAT|
|---------------------|-------|-------|-------|------|------|---------|
|t5-base-qg-highlight |21.3226|27.0854|43.5962|  -   |   -  |highlight|
|t5-base-multitask-qg |21.0141|26.9113|43.2484|82.46 |90.272|highlight|
|t5-small-multitask-qg|18.9872|25.2217|40.7893|76.121|84.904|highlight|
|t5-small-qg-highlight|18.5921|24.9915|40.1886|  -   |   -  |highlight|
|t5-small-qg-prepend  |18.2791|24.6722|39.958 |  -   |   -  |prepend  |


## Usage
Use the pipeline whch mimics HF transformers pipeline for easy inference.

The pipeline is divided into 2 tasks
1. "question-generation": for single task question generation models.
2. "multitask-qa-qg": for multi-task qa,qg models.

#### Question Generation

```python3
from pipelines import pipeline

nlp = pipeline("question-generation")
nlp("42 is the answer to life, the universe and everything.")
=> [{'answer': '42', 'question': 'What is the answer to life, the universe and everything?'}]
```

**prepend format**
```python3
nlp = pipeline("question-generation", model="valhalla/t5-small-qg-prepend", qg_format="prepend")
nlp("42 is the answer to life, the universe and everything.")
=> [{'answer': '42 ', 'question': 'What is the answer to life, the universe, and everything?'}]
```

#### Multitask QA-QG
```python3
nlp = pipeline("multitask-qa-qg")

# to generate questions simply pass the text
nlp("42 is the answer to life, the universe and everything.")
=> [{'answer': '42', 'question': 'What is the answer to life, the universe and everything?'}]

# for qa pass a dict with "question" and "context"
nlp({
    "question": "What is 42 ?",
    "context": "42 is the answer to life, the universe and everything."
})
=> 'the answer to life, the universe and everything'
```

#### End-to-end question generation (without answer supervision)
```python3
nlp = pipeline("e2e-qg")
nlp("Python is a programming language. Created by Guido van Rossum and first released in 1991.")
=> [
    'What is a programming language?',
    'Who created Python?',
    'When was Python first released?'
]
```

By default the `question-generation` pipeline will download the [valhalla/t5-small-qg-hl](https://huggingface.co/valhalla/t5-small-qg-hl) model with `highlight` qg format.

If you want to use prepend format then provide the path to the prepend model and set `qg_format` to `"prepend"`.
For extracting answer like spans it uses [valhalla/t5-small-qa-qg-hl](https://huggingface.co/valhalla/t5-small-qa-qg-hl) model, you can provide a different model through `ans_model` parameter.

The `multitask-qa-qg` model is for multitask models which can extract answer like spans, do qg and qa, so it won't need seperate `ans_model`. By default [valhalla/t5-small-qa-qg-hl](https://huggingface.co/valhalla/t5-small-qa-qg-hl) model is used with `highlight` format. If you want to use prepend format then provide the path to the prepend model and set `qg_format` to `"prepend"`

The `e2e-qg` pipeline is for end-to-end question generation. These models can generate multiple questions simultaneously without answer supervision. By default it uses [valhalla/t5-small-e2e-qg](https://huggingface.co/valhalla/t5-small-e2e-qg)

## Fine-tuning

### Data processing 

To support different data formats the trainer expects pre-processed cached dataset, so you can process the data the way you want.
The cached dataset should be saved using `torch.save` and it should return a `dict` with `source_ids`, `target_ids`, `attention_mask` keys for `__getitem__`.

- `source_ids`: encoded source text
- `target_ids`: encoded target text
- `attention_mask`: attention mask for the `source_ids`
  
The `T2TDataCollator` takes care of preparing right `input_ids` and `labels`. It also trims the batches dynamically to remove excessive padding tokens, to speed up the training.

The `data/squad_multitask` containes the modifed SQuAD dataset for answer aware question generation (using both prepend and highlight formats), question answering (text-to-text), answer extraction and end-to-end question generation. This dataset can be loaded using the awesome 🤗`nlp` library, this makes processing very easy.

To process and cache the dataset use `prepare_data.py` script. It will load the correct tokenizer depending on the `model_type` argument. It adds two new tokens `<sep>` and `<hl>` to the tokenizer and saves it at `{model_type}_qg_tokenizer` path. You should pass this tokenizer to the fine-tuning script.

The datasets will be saved in `data/` directory. You should provide filenames using `train_file_name` and `valid_file_name` arguments.

**process data for single task question generation with highlight_qg_format**
```bash
python prepare_data.py \
    --task qg \
    --model_type t5 \
    --dataset_path data/squad_multitask/ \
    --qg_format highlight_qg_format \
    --max_source_length 512 \
    --max_target_length 32 \
    --train_file_name train_data_qg_hl_t5.pt \
    --valid_file_name valid_data_qg_hl_t5.pt \
```

**process data for multi-task qa-qg with highlight_qg_format**

`valid_for_qg_only` argument is used to decide if the validation set should only contain data for qg task. For my multi-task experiments I used validation data with only qg task so that the eval loss curve can be easly compared with other single task models

```bash
python prepare_data.py \
    --task multi \
    --valid_for_qg_only \ 
    --model_type t5 \
    --dataset_path data/squad_multitask/ \
    --qg_format highlight_qg_format \
    --max_source_length 512 \
    --max_target_length 32 \
    --train_file_name train_data_qa_qg_hl_t5.pt \
    --valid_file_name valid_data_qg_hl_t5.pt \
```

**process dataset for end-to-end question generation**
```bash
python prepare_data.py \
    --task e2e_qg \
    --valid_for_qg_only \ 
    --model_type t5 \
    --dataset_path data/squad_multitask/ \
    --qg_format highlight_qg_format \
    --max_source_length 512 \
    --max_target_length 32 \
    --train_file_name train_data_e2e_qg_t5.pt \
    --valid_file_name valid_data_e2e_qg_t5.pt \
```

### training
Use the `run_qg.py` script to  start training. It uses transformers `Trainer` class for training the models.


```bash
python run_qg.py \
    --model_name_or_path t5-small \
    --model_type t5 \
    --tokenizer_name_or_path t5_qg_tokenizer \
    --output_dir t5-small-qg-hl \
    --train_file_path data/train_data_qg_hl_t5.pt \
    --valid_file_path data/valid_data_qg_hl_t5.pt \
    --per_device_train_batch_size 32 \
    --per_device_eval_batch_size 32 \
    --gradient_accumulation_steps 8 \
    --learning_rate 1e-4 \
    --num_train_epochs 10 \
    --seed 42 \
    --do_train \
    --do_eval \
    --evaluate_during_training \
    --logging_steps 100
```

or if you want to train it from script or notebook then

```python3
from run_qg import run_qg

args_dict = {
    "model_name_or_path": "t5-small",
    "model_type": "t5",
    "tokenizer_name_or_path": "t5_qg_tokenizer",
    "output_dir": "t5-small-qg-hl",
    "train_file_path": "data/train_data_qg_hl_t5.pt",
    "valid_file_path": "data/valid_data_qg_hl_t5.pt",
    "per_device_train_batch_size": 32,
    "per_device_eval_batch_size": 32,
    "gradient_accumulation_steps": 8,
    "learning_rate": 1e-4,
    "num_train_epochs": 10,
    "seed": 42,
    "do_train": True,
    "do_eval": True,
    "evaluate_during_training": True,
    "logging_steps": 100
}

# start training
run_qg(args_dict)
```

## Relevant papers
- https://arxiv.org/abs/2005.01107v1
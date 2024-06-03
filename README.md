**Codebase and data are uploaded in progress. **

VOLT(-py) is a vocabulary learning codebase that allows researchers and developers to automaticaly generate a vocabulary with suitable granularity for machine translation.  
To help more readers understand our work better, I write a [blog](https://jingjing-nlp.github.io/volt-blog/) at this repo. 

### What's New:

* July 2021: Support vocabulary learning for classification. 
* July 2021: Support En-De translation, TED bilingual translation, and multilingual translation.  
* July 2021: Support subword-nmt tokenization. 
* July 2021: Support sentencepiece tokenization.

## What's On-going:

* Support pip usage.

### Features:

* Efficient: CPU learning on one machine.
* Easy-to-use: Support widely-used tokenization toolkits, subword-nmt and sentencepiece.   
  
# Requirements and Installation

The required environments:

* python 3
* tqdm
* mosedecoder
* subword-nmt
* POT (local POT)

**To use VOLT** and develop locally:

``` bash
git clone https://github.com/Jingjing-NLP/VOLT/
cd VOLT
git clone https://github.com/moses-smt/mosesdecoder.git
git clone https://github.com/rsennrich/subword-nmt.git
pip3 install sentencepiece
pip3 install tqdm 
cd POT
pip3 install --editable ./ -i https://pypi.doubanio.com/simple --user
cd ../
```

# Usage

* The first step is to get vocabulary candidates based on tokenized texts. Notice: the tokenized texts should be in charater level. Please do not use segmentation tools to segment your texts. The sub-word vocabulary can be generated by subword-nmt and sentencepiece. Here are two examples.
  * This example shows how to learn a vocabulary for seq2seq tasks ( including source data and target data). 

  ```
  #Assume source_file is the file stroing texts in the source data
  #Assume target_file is the file stroing texts in the target data
  size=30000 # the size of BPE
  cat source_file > training_data
  cat target_file >> training_data 

 
  #subword-nmt style:
  mkdir bpeoutput
  BPE_CODE=bpeoutput/code # the path to save vocabulary
  python3 subword-nmt/learn_bpe.py -s $size  < training_data > $BPE_CODE
  python3 subword-nmt/apply_bpe.py -c $BPE_CODE < source_file > bpeoutput/source.file
  python3 subword-nmt/apply_bpe.py -c $BPE_CODE < target_file > bpeoutput/target.file 

  #sentencepiece style:
  cd examples
  mkdir spmout
  python3 spm/spm_train.py --input=training_data --model_prefix=spm --vocab_size=$size --character_coverage=1.0 --model_type=bpe
  #After this step, you will see spm.vocab and spm.model. 
  #Change spm.vocab to a file where each line is splited via a single space like example "abc 100"
  sed -i 's/\t/ /g' spm.vocab
  python3 spm/spm_encoder.py --model spm.model --inputs source_file --outputs spmout/source.file --output_format piece
  python3 spm/spm_encoder.py --model spm.model --inputs target_file --outputs spmout/target.file --output_format piece
  ```

  * This example shows how to get a vocabulary from a single file for non-seq2seq tasks.

  ```
  #Assume source_file is the file stroing your data
  size=30000 # the size of BPE

  #subword-nmt style:
  mkdir bpeoutput
  BPE_CODE=bpeoutput/code # the path to save vocabulary
  python3 subword-nmt/learn_bpe.py -s $size  < source_file > $BPE_CODE
  python3 subword-nmt/apply_bpe.py -c $BPE_CODE < source_file > bpeoutput/source.file
  

  #sentencepiece style:
  cd examples
  mkdir spmout
  python3 spm/spm_train.py --input=source_file --model_prefix=spm --vocab_size=$size --character_coverage=1.0 --model_type=bpe
  #After this step, you will see spm.vocab and spm.model
  python3 spm/spm_encoder.py --model spm.model --inputs source_file --outputs spmout/source.file --output_format piece
  ```

* The second step is to run VOLT scripts. It accepts the following parameters:
  * --source_file: the file storing source data for seq2seq tasks or the file string all raw texts for non-seq2seq tasks.
  * --token_candidate_file: the file storing token candidates. Each line is splited via a single space like example "abc 100"
  * --tokenizer: which toolkit you use to get token candidates.  Only two choices are supported: subword-nmt and sentencepiece. 
  * --size_file: the file to store the vocabulary size recommended by VOLT.
  * --target_file: (optional) the file storing target data for seq2seq tasks. None by default.
  * --max_number: (optional) the maximum size of the vocabulary generated by VOLT. 10,000 by default. 
  * --interval: (optional) the search granularity in VOLT. 1,000 by default. 
  * --loop_in_ot: (optional) the maximum interation loop in the Sinkhorn solution. 500 by default.
  * --threshold: (optional) the threshold to decide which tokens are added into the final vocabulary from the optimal matrix. Small threshold means that the final vocabulary is more like BPE-style vocabulary. 1e-5 by default.

  ```
  #For seq2seq tasks with source file and target file, you can use the following commands:
  #subword-nmt style
  python3 ../ot_run.py --source_file bpeoutput/source.file --target_file bpeoutput/target.file \
            --token_candidate_file $BPE_CODE \
            --vocab_file bpeoutput/vocab --max_number 10000 --interval 1000  --loop_in_ot 500 --tokenizer subword-nmt --size_file bpeoutput/size 
  #sentencepiece style
  python3 ../ot_run.py --source_file spmoutput/source.file --target_file spmoutput/target.file \
            --token_candidate_file spm.vocab \
            --vocab_file spmoutput/vocab --max_number 10000 --interval 1000  --loop_in_ot 500 --tokenizer sentencepiece --size_file spmoutput/size 

  #For non-seq2seq tasks with one source file, you can use the following commands:
  #subword-nmt style
  python3 ../ot_run.py --source_file bpeoutput/source.file \
            --token_candidate_file $BPE_CODE \
            --vocab_file bpeoutput/vocab --max_number 10000 --interval 1000  --loop_in_ot 500 --tokenizer subword-nmt --size_file bpeoutput/size 
    
  #sentencepiece style
  BPE_CODE=spm.vocab
  python3 ../ot_run.py --source_file spmoutput/source.file \
            --token_candidate_file spm.vocab  \
            --vocab_file spmoutput/vocab --max_number 10000 --interval 1000  --loop_in_ot 500 --tokenizer sentencepiece --size_file spmoutput/size 
  ```

* The third step is to use the generated vocabulary to segment your texts:
  
  ```
  #subword-nmt style
  echo "#version: 0.2" > bpeoutput/vocab.seg # add version info
  echo bpeoutput/vocab >> bpeoutput/vocab.seg
  python3 $BPEROOT/apply_bpe.py -c bpeoutput/vocab.seg < source_file > bpeoutput/source.file
  python3 $BPEROOT/apply_bpe.py -c bpeoutput/vocab.seg < target_file > bpeoutput/source.file #optional if your task does not contain target texts

  #sentencepiece style
  #for sentencepiece toolkit, here we only keep the optimal size
  best_size=$(cat spmoutput/size)
  #training_data contains source data and target data (optional if target data is provided)
  python3 spm/spm_train.py --input=training_data --model_prefix=spm --vocab_size=$best_size --character_coverage=1.0 --model_type=bpe
  python3 spm/spm_encoder.py --model spm.model --inputs source_file --outputs spmout/source.file --output_format piece
  python3 spm/spm_encoder.py --model spm.model --inputs target_file --outputs spmout/target.file --output_format piece #optional if your task does not contain target texts
  ```

* The last step is to use the segmented texts for downstream tasks. You can use the repo [Fairseq](https://github.com/pytorch/fairseq) for training and evaluation. We also upload the training and evaluation code in path "examples/". Notice: For a comparison of BLEU, you need to do "remove-bpe" operations for the generated texts. 

# Examples

We have given several examples in path "examples/", including En-De translation, En-Fr translation, multilingual translation, and En-De translation without joint vocabularies. 

* En-De translation: run_ende.sh
* En-De translation without joint vocabularies: run_ende_withoutjoint.sh
* En-Fr translation:  run_enfr.sh 
* TED bilingual translation:   run_ted_bilingual.sh
* TED bilingual translation with sentencepiece: run_ted_bilingual_senencepiece.sh
* TED many-to-one translation: run_ted_multilingual.sh

# Datasets

The WMT-14 En-de translation data can be downloaed via the running scripts.

For TED X-EN data, you can download at [X-EN](https://drive.google.com/drive/folders/1FNH7cXFYWWnUdH2LyUFFRYmaWYJJveKy?usp=sharing).
For TED EN-X data, you can download at [EN-X](https://drive.google.com/drive/u/1/folders/1du13KQG6JM9u1JLhnS47Pu4BQtfP2AK3)

# Citation

Please cite as:

``` bibtex
@inproceedings{volt,
  title = {Vocabulary Learning via Optimal Transport for Neural Machine Translation},
  author= {Jingjing Xu and
               Hao Zhou and
               Chun Gan and
               Zaixiang Zheng and
               Lei Li},
  booktitle = {Proceedings of ACL 2021},
  year = {2021},
}
```


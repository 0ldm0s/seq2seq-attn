# Sequence-to-Sequence Learning with Attentional Neural Networks

Implementation of a standard sequence-to-sequence model with attention where the encoder-decoder
are LSTMs. Also has the option to use characters on the input side (output is still at the
word-level) by doing a CNN+Highway over character embeddings to use as inputs.

## Dependencies

You will need the following packages:
* hdf5
* nngraph

GPU usage will additionally require:
* cutorch
* cunn

If running the character model, you should also install:
* cudnn
* luautf8

## Quickstart

We are going to be working with some example data in `data/` folder.
First run the data-processing code

```
python preprocess.py --srcfile data/src-train.txt --targetfile data/targ-train.txt
--srcvalfile data/src-val.txt --targetvalfile data/targ-val.txt --outputfile data/demo
```

This will take the source/target train/valid files (`src-train.txt, targ-train.txt,
src-val.txt, targ-val.txt`) and make some hdf5 files to be consumed by Lua.

`demo.src.dict`: Dictionary of source vocab to index mappings
`demo.targ.dict`: Dictionary of target vocab to index mappings
`demo-train.hdf5`: hdf5 containing the train data
`demo-val.hdf5`: hdf5 file containing the validation data

The `*.dict` files will be needed when predicting on new data.

Now run the model

```
th train.lua -data_file data/demo-train.hdf5 -val_data_file data/demo-val.hdf5 -savefile demo-model
```
You can also add `-gpuid 1` to use (say) GPU 1 in the cluster.

Now you have a model which you can use to predict on new data. To do this we are
going to be running beam search

```
th beam.lua -srcfile demo/src-train.txt -outfile pred.txt -srcdict demo/demo.src.dict
-targdict demo.targ.dict
```
This will output predictions into `pred.txt`. The predictions are going to be quite terrible,
as the demo dataset is small. Try running on some larger datasets.

## Details
Options for `preprocess.py`
`srcvocabsize, targetvocabsize`: Size of source/target vocabularies. This is constructed
by taking the top X most frequent words. Rest are replaced with special <unk> tokens
`srcfile, targetfile`: Path to source/target training data, where each line represents a single
source/target sequence.
`srcvalfile, targetvalfile`: Path to source/target validation data.
`batchsize`: Size of each mini-batch
`seqlength`: Maximum sequence length (sequences longer than this are dropped)
`outputfile`: Prefix of the output files
`maxwordlength`: For the character models, words are truncated (or zero-padded) to `maxwordlength`
`chars`: If 1, construct the character-level dataset as well.
`srcvocabfile, targetvocabfile`: If working with a preset vocab, then including these paths
will ignore the `srcvocabsize,targetvocabsize`
`unkfilter`: Ignore sentences with too many <unk> tokens. Can be an absolute count limit (if > 1)
or a proportional limit (0 < unkfilter < 1).

Options for `train.lua`

Options for `beam.lua`
`modelfile`: Path to model .t7 file
`srcfile`: Source sequence to decode (one line per sequence)
`targfile`: True target sequence (optional)
`outfile`: Where to output the predictions
`srcdict`: Source vocabulary
`targdict`: Target vocabulary
`chardict`: Character vocabulary
`beam`: Beam size
`max_sent_l`: Maximum sentence length. If any of the sequences in `srcfile` are longer than this
it will error out
`simple`: If = 1, output prediction is simply the first time the top of the beam
ends with an end-of-sentence token. If = 0, the model considers all hypotheses that have
been generated so far and ends with end-of-sentence token and takes the highest scoring
of all of them
`replace_unk`: Replace the generated <unk> tokens with the source token that had the highest
attention weight. If `srctarg_dict` is provided, it will lookup the identified source token
and give the corresponding target token. If it is not provided (or the identified source token
does not exist in the table) then it will copy the source token
`srctarg_dict`: Source-target dictionary to replace <unk> tokens
`score_gold`: If = 1, score the true target output as well
`gpuid`: ID of the GPU to use
`gpuid2`: ID if the second GPU (if specified)
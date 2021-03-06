### About Lookback RNN

Lookback RNN introduces custom inputs and labels. The custom inputs allow the model to more easily recognize patterns that occur across 1 and 2 bars. They also help the model recognize patterns related to an events position within the measure. The custom labels reduce the amount of information that the RNN’s cell state has to remember by allowing the model to more easily repeat events from 1 and 2 bars ago. This results in melodies that wander less and have a more musical structure. For more information about the custom inputs and labels, and to hear some generated sample melodies, check out the [blog post](https://magenta.tensorflow.org/2016/07/15/lookback-rnn-attention-rnn/). You can also read through the `melody_to_input` and `melody_to_label` methods in `lookback_rnn_encoder_decoder.py` to see how the custom inputs and labels are actually being encoded. The rest of this README leads you through the steps of training the model and generating melodies from it.

### File Structure

In these examples we store all our data in `/tmp`, but feel free to choose a different directory on your local machine. To get an idea of how data is used and generated by the model, this is what your file structure will look like after you have trained your model and generated some melodies from it.

```
tmp
├── lookback_rnn
│   ├── generated
│   │   ├── 2016-07-15_094500_01.mid
│   │   ├── 2016-07-15_094500_02.mid
│   │   ├── 2016-07-15_094500_03.mid
│   │   └── ...
│   │       (Original melodies generated by your trained model.
│   │        The filename format is date_time_count.mid)
│   ├── logdir
│   │   ├── run1
│   │   │   ├── eval
│   │   │   │   └── events.out.tfevents.1467425436.name
│   │   │   │       (Saved evaluation data used by TensorBoard.)
│   │   │   └── train
│   │   │       ├── checkpoint
│   │   │       ├── events.out.tfevents.1467422802.name
│   │   │       ├── events.out.tfevents.1467423145.name
│   │   │       ├── events.out.tfevents.1467423162.name
│   │   │       ├── graph.pbtxt
│   │   │       ├── model.ckpt-78
│   │   │       ├── model.ckpt-78.meta
│   │   │       ├── model.ckpt-80
│   │   │       ├── model.ckpt-80.meta
│   │   │       ├── model.ckpt-83
│   │   │       └── model.ckpt-83.meta
│   │   │           (Saved checkpoints, graph data, and training
│   │   │            data used by TensorBoard.)
│   │   ├── run2
│   │   │   ├── eval
│   │   │   │   └── ...
│   │   │   └── train
│   │   │       └── ...
│   │   └── ...
│   │       (Multiple runs can be stored in logdir. Each run can
│   │        use different hyperparameters or a different dataset
│   │        if multiple datasets are created. All runs can be
│   │        visualized together in TensorBoard.)
│   └── sequence_examples
│       ├── eval_melodies.tfrecord
│       └── training_melodies.tfrecord
│           (TFRecord files of SequenceExamples. These are used
│            to train and evaluate the model. Each SequenceExample
│            has a sequence of inputs and a sequence of labels that
│            represent a melody. These melodies are extracted from
│            the NoteSequences in /tmp/notesequences.tfrecord.)
└── notesequences.tfrecord
    (A TFRecord file of NoteSequences. Each NoteSequence represents a
     song. If the NoteSequence was generated from a MIDI file, it will
     contain the same music data that was in that MIDI file.)
```

### Create NoteSequences

Our first step will be to convert a collection of MIDI files into NoteSequences. NoteSequences are [protocol buffers](https://developers.google.com/protocol-buffers/), which is a fast and efficient data format, and easier to work with than MIDI files. See [Building your Dataset](https://github.com/tensorflow/magenta#building-your-dataset) for instructions on generating a TFRecord file of NoteSequences. In this example, we assume the NoteSequences were output to ```/tmp/notesequences.tfrecord```.

### Create SequenceExamples

SequenceExamples are fed into the model during training and evaluation. Each SequenceExample will contain a sequence of inputs and a sequence of labels that represent a melody. Run the command below to extract melodies from our NoteSequences and save them as SequenceExamples. If we specify an `--eval_output` and an `--eval_ratio`, two collections of SequenceExamples will be generated, one for training, and one for evaluation. With an eval ratio of 0.10, 10% of the extracted melodies will be saved in the eval collection, and 90% will be saved in the training collection.

```
bazel run //magenta/models/lookback_rnn:lookback_rnn_create_dataset -- \
--input=/tmp/notesequences.tfrecord \
--train_output=/tmp/lookback_rnn/sequence_examples/training_melodies.tfrecord \
--eval_output=/tmp/lookback_rnn/sequence_examples/eval_melodies.tfrecord \
--eval_ratio=0.10
```

### Train and Evaluate the Model

Build lookback_rnn_train first so that it can be run multiple times in parallel.

```
bazel build //magenta/models/lookback_rnn:lookback_rnn_train
```

Run the command below to start a training job. `--run_dir` is the directory where checkpoints and TensorBoard data for this run will be stored. `--sequence_example_file` is the TFRecord file of SequenceExamples that will be fed to the model. `--num_training_steps` (optional) is how many update steps to take before exiting the training loop. If left unspecified, the training loop will run until terminated manually. `--hparams` (optional) can be used to specify hyperparameters other than the defaults. For this example, we specify a custom batch size of 64 instead of the default batch size of 128. Using smaller batch sizes can help reduce memory usage, which can resolve potential out-of-memory issues when training larger models. We'll also use a 2 layer RNN with 64 units each, instead of the default of 2 layers of 128 units each. This will make our model train faster. However, if you have enough compute power, you can try using larger layer sizes for better results.

```
./bazel-bin/magenta/models/lookback_rnn/lookback_rnn_train \
--run_dir=/tmp/lookback_rnn/logdir/run1 \
--sequence_example_file=/tmp/lookback_rnn/sequence_examples/training_melodies.tfrecord \
--hparams="{'batch_size':64,'rnn_layer_sizes':[64,64]}" \
--num_training_steps=20000
```

Optionally run an eval job in parallel. `--run_dir`, `--hparams`, and `--num_training_steps` should all be the same values used for the training job. `--sequence_example_file` should point to the separate set of eval melodies. Include `--eval` to make this an eval job, resulting in the model only being evaluated without any of the weights being updated.

```
./bazel-bin/magenta/models/lookback_rnn/lookback_rnn_train \
--run_dir=/tmp/lookback_rnn/logdir/run1 \
--sequence_example_file=/tmp/lookback_rnn/sequence_examples/eval_melodies.tfrecord \
--hparams="{'batch_size':64,'rnn_layer_sizes':[64,64]}" \
--num_training_steps=20000 \
--eval
```

Run TensorBoard to view the training and evaluation data.

```
tensorboard --logdir=/tmp/lookback_rnn/logdir
```

Then go to [http://localhost:6006](http://localhost:6006) to view the TensorBoard dashboard.

### Generate Melodies

Melodies can be generated during or after training. Run the command below to generate a set of melodies using the latest checkpoint file of your trained model.

`--run_dir` should be the same directory used for the training job. The `train` subdirectory within `--run_dir` is where the latest checkpoint file will be loaded from. For example, if we use `--run_dir=/tmp/lookback_rnn/logdir/run1`. The most recent checkpoint file in `/tmp/lookback_rnn/logdir/run1/train` will be used.

`--hparams` should be the same hyperparameters used for the training job, although some of them will be ignored, like the batch size.

`--output_dir` is where the generated MIDI files will be saved. `--num_outputs` is the number of melodies that will be generated. `--num_steps` is how long each melody will be in 16th steps (128 steps = 8 bars).

At least one note needs to be fed to the model before it can start generating consecutive notes. We can use `--primer_melody` to specify a priming melody using a string representation of a Python list. The values in the list should be ints that follow the melodies_lib.Melody format (-2 = no event, -1 = note-off event, values 0 through 127 = note-on event for that MIDI pitch). For example `--primer_melody="[60, -2, 60, -2, 67, -2, 67, -2]"` would prime the model with the first four notes of Twinkle Twinkle Little Star. Instead of using `--primer_melody`, we can use `--primer_midi` to prime our model with a melody stored in a MIDI file. For example, `--primer_midi=<absolute path to magenta/models/shared/primer.mid>` will prime the model with the melody in that MIDI file. If neither `--primer_melody` nor `--primer_midi` are specified, a random note from the model's note range will be chosen as the first note, then the remaining notes will be generated by the model. In the example below we prime the melody with `--primer_melody="[60]"`, a single note-on event for the note C4.


```
bazel run //magenta/models/lookback_rnn:lookback_rnn_generate -- \
--run_dir=/tmp/lookback_rnn/logdir/run1 \
--hparams="{'batch_size':64,'rnn_layer_sizes':[64,64]}" \
--output_dir=/tmp/lookback_rnn/generated \
--num_outputs=10 \
--num_steps=128 \
--primer_melody="[60]"
```

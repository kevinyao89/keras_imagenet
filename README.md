keras_imagenet
==============

This repository contains code I use to train Keras ImageNet (ILSVRC2012) image classification models from scratch.

**Highlight #1**: I use [TFRecords](https://www.tensorflow.org/tutorials/load_data/tf_records) and [tf.data.TFRecordDataset API](https://www.tensorflow.org/api_docs/python/tf/data/TFRecordDataset) to speed up data ingestion of the training pipeline.  This way I could multi-process the data pre-processing (including online data augmentation) task, and keep the GPUs maximally utilized.

**Highlight #2**: In addition to heavy data augmentation, I also use various tricks in attempt to achieve best accuracy for the trained image classification models.  More specifically, I implement 'LookAhead' optimizer ([reference](https://arxiv.org/abs/1907.08610)), 'iter_size' and 'l2 regularization' for the Keras models, and have tried to use 'AdamW' (Adam optimizer with decoupled weight decay).

I took most of the dataset preparation code from tensorflow [models/research/inception](https://github.com/tensorflow/models/tree/master/research/inception).  It was under Apache license as specified [here](https://github.com/tensorflow/models/blob/master/LICENSE).

Otherwise, please refer to the following blog posts for some more implementation details about the code:

* [Training Keras Models with TFRecords and The tf.data API](https://jkjung-avt.github.io/tfrecords-for-keras/)
* [Displaying Images in TensorBoard](https://jkjung-avt.github.io/tensorboard-images/)

# Prerequisite

The dataset and CNN models in this repository are built and trained using the 'keras' API within tensorflow.  I myself have tested the code with tensorflow 1.11.0 and 1.12.2.  My implementation of [the 'LookAhead' optimizer and 'iter_size' do **not** work for 'tensorflow.python.keras.optimizer_v2.OptimizerV2' (tensorflow-1.13.0+)](https://github.com/keras-team/keras/issues/3556).  So I would recommend tensorflow-1.12.x if you'd like to use my code to train Keras ImageNet models.

In addition, the python code in this repository is for python3.  Make sure you have tensorflow and its dependencies working for python3.

# Step-by-step

1. Download the 'Training images (Task 1 & 2)' and 'Validation images (all tasks)' from the [ImageNet Large Scale Visual Recognition Challenge 2012 (ILSVRC2012) download page](http://www.image-net.org/challenges/LSVRC/2012/nonpub-downloads).

   ```shell
   $ ls -l ${HOME}/Downloads/
   -rwxr-xr-x 1 jkjung jkjung 147897477120 Nov  7  2018 ILSVRC2012_img_train.tar
   -rwxr-xr-x 1 jkjung jkjung   6744924160 Nov  7  2018 ILSVRC2012_img_val.tar
   ```

2. Untar the 'train' and 'val' files.  For example, I put the untarred files at ${HOME}/data/ILSVRC2012/.

   ```shell
   $ mkdir -p ${HOME}/data/ILSVRC2012
   $ cd ${HOME}/data/ILSVRC2012
   $ mkdir train
   $ cd train
   $ tar xvf ${HOME}/Downloads/ILSVRC2012_img_train.tar
   $ find . -name "*.tar" | while read NAME ; do \
         mkdir -p "${NAME%.tar}"; \
         tar -xvf "${NAME}" -C "${NAME%.tar}"; \
         rm -f "${NAME}"; \
     done
   $ cd ..
   $ mkdir validation
   $ cd validation
   $ tar xvf ${HOME}/Downloads/ILSVRC2012_img_val.tar
   ```

3. Clone this repository.

   ```shell
   $ cd ${HOME}/project
   $ git clone https://github.com/jkjung-avt/keras_imagenet.git
   $ cd keras_imagenet
   ```

4. Pre-process the validation image files.  (The script would move the JPEG files into corresponding subfolders.)

   ```shell
   $ cd data
   $ python3 ./preprocess_imagenet_validation_data.py \
             ${HOME}/data/ILSVRC2012/validation \
             imagenet_2012_validation_synset_labels.txt
   ```

5. Build TFRecord files for 'train' and 'validation'.  (This step could take a couple of hours, since there are 1,281,167 training images and 50,000 validation images in total.)

   ```shell
   $ mkdir ${HOME}/data/ILSVRC2012/tfrecords
   $ python3 build_imagenet_data.py \
             --output_directory ${HOME}/data/ILSVRC2012/tfrecords \
             --train_directory ${HOME}/data/ILSVRC2012/train \
             --validation_directory ${HOME}/data/ILSVRC2012/validation
   ```

6. As an example, train a 'GoogLeNet_BN' (GoogLeNet with Batch Norms) model.

   Take a peek at [train_googlenet_bn.sh](https://github.com/jkjung-avt/keras_imagenet/blob/master/train_googlenet_bn.sh) and [models/googlenet.py](https://github.com/jkjung-avt/keras_imagenet/blob/master/models/googlenet.py) before running it.  You could adjust the learning rate schedule, weight decay and epochs in the script to see if it produces a model with better accuracy.

   ```shell
   $ ./train_googlenet_bn.sh
   ```

   On my desktop PC with an NVIDIA GTX-1080 Ti GPU, it takes 7~8 days to train this model for 60 epochs.  And top-1 accuracy of this [trained googelnet_bn model](https://drive.google.com/open?id=1-f06EebkIldPADxrZiEeQad8kiNZ6x5g) is 0.6954.

   NOTE: I do random rotation of training images, which actually slows down data ingestion quite a bit.  If you don't need random rotation as one of the data augmentation schemes, you could comment out [the code](https://github.com/jkjung-avt/keras_imagenet/blob/master/utils/image_processing.py#L354) to speed up training.

   For reference, here is a list of options for the `train.py` script which gets called inside `train_googlenet_bn.sh`:

   * `--add_dropout`: add a DropOut(0.5) layer before the last Dense layer, default is False
   * `--optimizer`: 'sgd', 'adam' or 'rmsprop'
   * `--use_lookahead`: use 'LookAhead' optimizer, default is False
   * `--batch_size`: batch size for both training and validation
   * `--iter_size`: aggregate gradients before doing 1 weight update, i.e. effective batch size = batch_size * iter_size
   * `--lr_sched`: 'linear' or 'exp' (exponential) decay of learning rate per epoch
   * `--initial_lr`: initial learning rate
   * `--final_lr`: final learning rate (for 'linear' lr_sched only)
   * `--lr_decay`: decaying factor for learning rate (for 'exp' lr_sched only)
   * `--weight_decay`: L2 regularization of weights in conv/dense layers
   * `--epochs`: total number of training epochs

7. Evaluate accuracy of the trained googlenet_bn model.

   ```shell
   $ python3 evaluate.py --dataset_dir ${HOME}/data/ILSVRC2012/tfrecords \
                         saves/googlenet_bn-model-final.h5
   ```

8. For training other CNN models, check out `models/models.py`.  In addition to `googlenet_bn`, `mobilenet_v2` and `resnet50`, you could implement your own Keras CNN models by extending the code.

# Additional notes about MobileNetV2

For some reason, Keras has trouble loading a trained/saved MobileNetV2 model.  The load_model() call would fail with this error message:

  `TypeError: '<' not supported between instances of 'dict' and 'float'`

To work around this problem, I followed [this post](https://github.com/tensorflow/tensorflow/issues/22697#issuecomment-436301471) and added the following at line 309 (after the `super()` call of `ReLU`) lines in `/usr/local/lib/python3.6/dist-packages/tensorflow/python/keras/layers/advanced_activations.py`.

  ```python
      if type(max_value) is dict:
          max_value = max_value['value']
      if type(negative_slope) is dict:
          negative_slope = negative_slope['value']
      if type(threshold) is dict:
          threshold = threshold['value']
  ```

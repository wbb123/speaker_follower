# Speaker-Follower Models for Vision-and-Language Navigation

This repository contains the code for the following paper:

* D. Fried*, R. Hu*, V. Cirik*, A. Rohrbach, J. Andreas, L.-P. Morency, T. Berg-Kirkpatrick, K. Saenko, D. Klein**, T. Darrell**, *Speaker-Follower Models for Vision-and-Language Navigation*. in NIPS, 2018. ([PDF](https://arxiv.org/pdf/1806.02724.pdf))
```
@inproceedings{fried2018speaker,
  title={Speaker-Follower Models for Vision-and-Language Navigation},
  author={Fried, Daniel and Hu, Ronghang and Cirik, Volkan and Rohrbach, Anna and Andreas, Jacob and Morency, Louis-Philippe and Berg-Kirkpatrick, Taylor and Saenko, Kate and Klein, Dan and Darrell, Trevor},
  booktitle={Advances in Neural Information Processing Systems (NIPS)},
  year={2018}
}
```
(*, **: indicates equal contribution)

Project Page: http://ronghanghu.com/speaker_follower

## Installation

1. Install Python 3 (Anaconda recommended: https://www.continuum.io/downloads).
2. Install PyTorch following the instructions on https://pytorch.org/ (we used PyTorch 0.3.1 in our experiments).
3. Download this repository or clone **recursively** with Git, and then enter the root directory of the repository:  
```
# Make sure to clone with --recursive
git clone --recursive https://github.com/ronghanghu/speaker_follower.git
cd speaker_follower
```

If you didn't clone with the `--recursive` flag, then you'll need to manually clone the pybind submodule from the top-level directory:
```
git submodule update --init --recursive
```
4. Install the dependencies for the Matterport3D Simulator:
```
sudo apt-get install libopencv-dev python-opencv freeglut3 freeglut3-dev libglm-dev libjsoncpp-dev doxygen libosmesa6-dev libosmesa6 libglew-dev
```
5. Compile the Matterport3D Simulator:
```
mkdir build && cd build
cmake ..
make
cd ../
```

*Note:* This repository is built upon the [Matterport3DSimulator](https://github.com/peteanderson80/Matterport3DSimulator) codebase. Additional details on the Matterport3D Simulator can be found in [`README_Matterport3DSimulator.md`](README_Matterport3DSimulator.md).

## Train and evaluate on the Room-to-Room (R2R) dataset

### Download and preprocess the data

1. Download the Precomputing ResNet Image Features, and extract them into `img_features/`:
```
mkdir -p img_features/
cd img_features/
wget https://storage.googleapis.com/bringmeaspoon/img_features/ResNet-152-imagenet.zip -O ResNet-152-imagenet.zip
unzip ResNet-152-imagenet.zip
cd ..
```
After this step, `img_features/` should contain `ResNet-152-imagenet.tsv`. (Note that you only need to download the features extracted from ImageNet-pretrained ResNet to run the following experiments. Places-pretrained ResNet features or actual images are not required.)

2. Download the R2R dataset and our sampled trajectories for data augmentation:
```
./tasks/R2R/data/download.sh
```

### Training

1. Train the speaker model:  
```
python tasks/R2R/train_speaker.py
```

2. Generate synthetic instructions from the trained speaker model as data augmentation:
```
# the path prefix to the speaker model (trained in Step 1 above)
export SPEAKER_PATH_PREFIX=tasks/R2R/speaker/snapshots/speaker_teacher_imagenet_mean_pooled_train_iter_20000

python tasks/R2R/data_augmentation_from_speaker.py \
    $SPEAKER_PATH_PREFIX \
    tasks/R2R/data/R2R
```
After this step, `R2R_literal_speaker_data_augmentation_paths.json` be generated under `tasks/R2R/data/`. This JSON file contains synthetic instructions generated by the speaker model on sampled new trajectories in the train environment (i.e. the speaker-driven data augmentation in our paper).

Alternatively, you can directly download our precomputed speaker-driven data augmentation with
`./tasks/R2R/data/download_precomputed_augmentation.sh`.

3. Train the follower model on the combination of the original and the augmented training data.
```
python tasks/R2R/train.py \
  --use_pretraining --pretrain_splits train literal_speaker_data_augmentation_paths
```
The follower will be first trained on the combination of the original `train` environment and the new `literal_speaker_data_augmentation_paths` (generated in Step 2 above) for 50000 iterations, and then fine-tuned on the
original `train` environment for 20000 iterations. This step may take a long time. (It look approximately 50 hours using a single GPU on our local machine.)

#### Note

* All the commands above run on a single GPU. You may choose a specific GPU by setting `CUDA_VISIBLE_DEVICES` environment variable (e.g. `export CUDA_VISIBLE_DEVICES=1` to use GPU 1).
* You may directly download our trained speaker model and follower model with
```
./tasks/R2R/snapshots/release/download_speaker_release.sh  # Download speaker
./tasks/R2R/snapshots/release/download_follower_release.sh  # Download follower
```
The scripts above will save the downloaded models under `./tasks/R2R/snapshots/release/`. To use these downloaded models, set the speaker and follower path prefixes as follows:
```
export SPEAKER_PATH_PREFIX=tasks/R2R/snapshots/release/speaker_final_release
export FOLLOWER_PATH_PREFIX=tasks/R2R/snapshots/release/follower_final_release
```

### Inference

1. Set the path prefixes for the trained speaker and follower model:  
```
# the path prefixes to the trained speaker and follower model
# change these path prefixes if you are using downloaded models.
export SPEAKER_PATH_PREFIX=tasks/R2R/speaker/snapshots/speaker_teacher_imagenet_mean_pooled_train_iter_20000
export FOLLOWER_PATH_PREFIX=tasks/R2R/snapshots/follower_with_pretraining_sample_imagenet_mean_pooled_train_iter_11100
```

2. Generate top-ranking trajectory predictions with pragmatic inference:
```
# Specify the path prefix to the output evaluation file
export EVAL_FILE_PREFIX=tasks/R2R/eval_outputs/pragmatics

CUDA_VISIBLE_DEVICES=0 python tasks/R2R/rational_follower.py \
    $FOLLOWER_PATH_PREFIX \
    $SPEAKER_PATH_PREFIX \
    --batch_size 15 --beam_size 40 --state_factored_search \
    --use_test_set \
    --eval_file $EVAL_FILE_PREFIX
```
This will generate the prediction files in the directory of `EVAL_FILE_PREFIX`, and also print the performance on `val_seen` and `val_unseen` splits. (The displayed performance will be zero on the `test` split, since the test JSON file does not contain ground-truth target locations.) The predicted trajectories with the above script contain only the **top-scoring trajectories** among all candidate trajectories, ranked with pragmatic inference.

3. For participating in the [Vision-and-Language Navigation Challenge](https://evalai.cloudcv.org/web/challenges/challenge-page/97/overview), add `--physical_traversal` option to generate physically-plausible trajectory predictions with pragmatic inference:
```
# Specify the path prefix to the output evaluation file
export EVAL_FILE_PREFIX=tasks/R2R/eval_outputs/pragmatics_physical

CUDA_VISIBLE_DEVICES=0 python tasks/R2R/rational_follower.py \
    $FOLLOWER_PATH_PREFIX \
    $SPEAKER_PATH_PREFIX \
    --batch_size 15 --beam_size 40 --state_factored_search \
    --use_test_set --physical_traversal \
    --eval_file $EVAL_FILE_PREFIX
```
This will generate the prediction files in the directory of `EVAL_FILE_PREFIX`. These prediction files can be submitted to https://evalai.cloudcv.org/web/challenges/challenge-page/97/overview for evaluation.

The major difference with `--physical_traversal` is that now the generated trajectories contain **all states visited by the search algorithm in the order they are traversed**. The agent expands each route one step forward at a time, and then switches to expand the next route. The details are explained in Appendix E in [our paper](https://arxiv.org/pdf/1806.02724.pdf).

## Acknowledgements

This repository is built upon the [Matterport3DSimulator](https://github.com/peteanderson80/Matterport3DSimulator) codebase.

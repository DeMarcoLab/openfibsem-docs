# Concepts

Overview of FIBSEM system
Skip to Core API Concepts if you are familar with FIBSEMs

## System Overview

FIBSEM Systems

## Chamber

Overview
Vaccuum, Venting and Pumping
Cryogenic Temperature

## Imaging

Overview
Electron Beam Imaging
Ion Beam Imaging
Fluorescence Imaging
Imaging Parameters

## Stage

Overview
Axes and Coordinate Systems
Shuttle, and Pre-Tilt
Stage Movement

## Milling

Overview
Milling Patterns
Milling Parameters
Usage

## Manipulator

Overview
Axes and Coordinate Systems
Manipulator Movement
Manipulator Calibration
Manipulator Exchange

## Gas Injection and Sputter Coater

Overview
Specific Deposition
Cryogenic Deposition
Sputter Coating

## Core API Concepts

### Imaging

Changing Imaging Parameters

Acquiring Images

### Movement

Relative Movement
Absolute Movement
Beam Coincidence
Stable Movement
Vertical Movement

Microscope State

### Milling

Milling Patterns
Changing Milling Patterns
Changing Milling Parameters
Running Milling Operations

### Logging

Event Log

### Automation

- Image Registration, Cross-Correlation
- Machine Learning, Segmentation, Feature Detection

### Configuring your Microscope

ip_address
manufacturer
shuttle_pre_tilt
reference_rotation: 0-rotation

system fingerprint

advanced configuration

## User Interface

### User Interface Widgets

### Supervised Mode

Machine Learning

Milling Operations

## Machine Learning Data Engine

We have implemented a data engine into the AutoLamella program. This data engine enables efficient model improvement across a wide range of tasks by using human feedback, and a set of model enhanced tools. Moreover, the data engine allows the models to improve faster, and more efficiently as they improve in performance by using the models to label data and generate test sets.

[DATA ENGINE IMAGE]

### Data Collection

When running AutoLamella, we automatically save the images, masks and keypoint detections from the machine learning systems. In addition, any image can be manually added to the dataset, but will have to be manually pre-processed to conform. By default, machine learning data (images, masks, keypoints) are saved to the fibsem/log/data/ml directory.

### Data Curation

In order to efficiently improve the model, as well as automatically generate test datasets we implemented an active learning system into the user interface. When running AutoLamella in [Supervised Mode](#supervised-mode), the user has the opportunity to correct the keypoint detections produced by the model. We log this intervention, and flag that image to be added to the training dataset. Images that the model gets correct are also logged, and added to the test dataset.

This form of active learning allows us to collect images that the model is currently failing on, and not feed it any more of images that it already succeeds at. This is crucical to balance the dataset, and efficiently improve when the dataset is relative small (~100s of images).

For more details on this approach to data curation, please see [AutoLamella Datasets and Models](../autolamella/case_study_dataset_and_models.md)

### Data Labelling

We developed a napari plugin for labelling images for semantic segmentation. The plugin supports three complementary labelling modes; manual labelling, model assisted labelling and sam assisted labelling.

[Image Labelling Napari Plugin]
[Example Labelling GIF for each mode]

For details about how this was used, please see [AutoLamella Datasets and Models](../autolamella/case_study_dataset_and_models.md)

### Manual Labelling

The default mode is manual labelling, which enables the use the napari paint tools to manually label images. The manual labelling mode is also used to edit or 'touch up' model generated labels.

The user can define the segmentation labels and colors used by editing the fibsem/segementation/segmentation_config.yaml file.  

### Model Assisted Labelling

The model assisted labelling tool allows you to use a trained model to assist in the labelling of new data. This is useful for labelling large datasets. The model will make a prediction and the user can correct the prediction using the same drawing tools.

To use, go to the Model tab and load your model, and then tick 'model assisted' to enable the model assisted labelling.

### SegmentAnything Assisted Labelling

We have implemented the Segment Anything Model from MetaAI. This model is trained to segment any object. Here we use it as part of the model assisted labelling. We currently support two versions of SAM; SegmentAnything and MobileSAM.

SegmentAnything is the original model from MetaAI. It is powerful, but requires a decently large GPU.
MobileSAM is a recent, faster implementation of SAM, that can be run without a large GPU.

To use either model, you will need to install some additional dependencies, and download the model weights as described below.

##### Segment Anything

For more detailed about SAM see: <https://github.com/facebookresearch/segment-anything>

To use SAM:
  
  ```python
pip install git+https://github.com/facebookresearch/segment-anything.git
pip install opencv-python pycocotools matplotlib onnxruntime onnx

```

Download weights: [SAM ViT-H](https://dl.fbaipublicfiles.com/segment_anything/sam_vit_h_4b8939.pth)

##### MobileSAM

The labelling UI also supports using MobileSAM which is a faster version of Segment Anything (+ less gpu memory).

``` bash
pip install git+https://github.com/ChaoningZhang/MobileSAM.git

```

Download weights: [MobileSAM ViT-T](https://drive.google.com/file/d/1dE-YAG-1mFCBmao2rHDp0n-PP4eH7SjE/view?usp=sharing)

### Model Training

Model training is relatively simple, and by default we use models from the segmentation-models-pytorch package for the implementation. 

#### Data Augmentation

Data is augmented with standard data augmentation methods that are suitable for electron microscope data. 

We currently use; random rotation, random horizontal / vertical flip, random autocontrast, random equalize, gaussian blue and color jitter.

#### Losses, Optimiser

We use a standard Adam optimiser, and multi-loss (cross-entropy, dice, and focal) 

#### Training the Model

To train the model:

```bash
python fibsem/segementation/train.py --config config.yaml
```


Example Model Training Config

```yaml
# data
data_paths:  [/path/to/data, /path/to/second/data]                  # paths to image data (multiple supported)
label_paths: [/path/to/data/labels, /path/to/second/data/labels]    # paths to label data (multiple supported)
save_path: /path/to/save/checkpoints                                # path to save checkpoints (checkpointed each epoch)
checkpoint: null                                                    # checkpoint to resume from

# model
encoder: "resnet34"                             # segmentation model encoder (imagenet)
num_classes: 6                                  # number of classes

# training
epochs: 50                                      # number of epochs
split: 0.1                                      # train / val split
batch_size: 4                                   # batch size
lr: 3.0e-4                                      # initial learning rate

# logging
train_log_freq: 32                              # frequency to log training images
val_log_freq: 32                                # frequency to log validation images

# wandb
wandb: true                                     # enable wandb logging
wandb_project: "autolamella-mega"               # wandb project
wandb_entity: "openfibsem"                      # wandb user / org
model_type: "mega-model"                        # model type note (descriptive only)
note: "notes about this specific training run"  # additional trianing note (descriptive only)
```

#### External Integration - NNUnet

NNUnet is a popular library for training segmentation models. We provide a set of converters for converting datasets to the nnunet format, and converting nnunet models to compatible openfibsem formats.


Script to convert data labelled with OpenFIBSEM to NNUnet format.

```bash
python scripts/convert_to_nnunet_dataset.py -h
--data_path: the path to the images directory (source)
--label_path:  the path to the labels directory (source)
--nnunet_data_path: the path to nnunet data directory (destination)
--label_map : list of label names (text file)
--filetype: the file extension of the images / labels (.tif for fibsem)
```

Script to convert nnunet trained model directory to checkpoint

```bash
python scripts/export_nnunet_checkpoint.py -h
--path: path to nnunet model directory (source)
--checkpoint_path: the path to save the output checkpoint
--checkpoint_name: the filename of the output checkpoint 
```

Make sure you save your checkpoint with 'nnunet' in the name so the load_model helper can automatically load it:

```python
from fibsem.segementation.model import load_model

# load nnunet model checkpoint
model = load_model('my-nnunet-model-checkpoint.pt')

```


#### External Integration - ONNX

ONNX (Open Neural Network Exchange) is standardised format for machine learning models. We provide a script to convert trained models to onnx format. ONNX models are much more portable, and don't require the dependency on pytorch to run.

```bash
python scripts/convert_to_onnx.py -h
--checkpoint: path to openfibsem model checkpoint
--output: path to save onnx checkpoint
```

Code to load onnx model in openfibsem

```python
from fibsem.segmentation.model import load_model

# load onnx model
model = load_model("my-model-checkpoint.onnx")
```

### Model Evaluation

### Keypoint Labelling

We provide a Keypoint Labelling Napari Plugin that can be used to label or edit keypoint labels generated by openfibsem. You can start from an empty directory of images, or load a csv containing the keypoint detections. The keypoints are used to evaluate the model as if it were being used online.

When you run AutoLamella, these keypoints used for detections are automatically logged, ready be used for evaluation.

### Keypoint Evaluation

We provide evaluation tools for evaluating the perform of a number of different models on the keypoint detection task.

The evaluation will run each model checkpoint through the detection pipeline, save the results and compare them to the ground truth labels provided. Each indivudal image can be plotted, as well as the full evaluation statistics. This evaluation pipeline is useful for checking model improvement and preventing regressions on previously successful tasks.

To run the evaluation:

```python
python fibsem/detection/run_evaluation.py --config config.yaml
```

Example Evaluation Configuration

```yaml
data_path: "path/to/data/keypoints.csv"     # test data csv (keypoints)
images_path: "path/to/data"                 # test data image directory
save_path: "path/to/results"                # save path for evaluation results

checkpoints: # list of checkpoints to evaluate
  - checkpoint: "checkpoint-01.pt"
  - checkpoint: "checkpoint-02.pt"

thresholds: # pixel thresholds for 'matched' keypoints
- 250
- 100
- 50
- 25
- 10

# options
run_eval: True          # run the evaluation
plot_eval: True         # plot the evaluation

show_det_plot: False    # show the individual keypoint detection plots
save_det_plot: True     # save the indivudial keypoint detection plots
show_eval_plot: False   # show the complete evaluation plots
save_eval_plot: True    # save the complete evaluation plots
```

### Model Deployment

We provide a number of generic segmentation model interfaces. We currently support the following model backends:
backend = "smp" (default), "nnunet", "onnx"

The load model function will automatically check which backend to use based off the name of the checkpoint. The overall generality of this model interface will be improved in the future.

Code

```python
from fibsem.segmentation.model import load_model

# load smp model
model = load_model("my-model-checkpoint.pt")

# load onnx model
model = load_model("my-model-checkpoint.onnx")

# load nnunet model
model = load_model("my-nnunet-model-checkpoint.pt")

# explictly set backend
model = load_model("my-nnunet-model-checkpoint.pt", backend="nnunet")

```

---
title: "Number recognition with TensorflowJS"
date: 2020-03-06T11:40:39+01:00
showDate: true
draft: true
tags: ["blog","front end","machine learning", "angular"]
images:
    - /posts/front-end-only-ml/doggo.png
---

# The Goal
Once upon a time, in a country far far away, two software developers were instructed to create a front end only web application
capable to perform numbers recognition on photos.

I happen to be one of those two developers ['o'](https://en.meming.world/images/en/6/6e/Surprised_Pikachu.jpg).  
The initial requirements for the application were simple:
- The application must be front end only
- The recognition time should've been equal or faster then [Google Cloud Vision](https://cloud.google.com/vision/)
- Each number should be recognized as an object (i.e. "123" outputs "1", "2", "3")

# Firebase
[Firebase](https://firebase.google.com/) is a simple tool from Google that substitutes mobile and web applications' back end.
Deploying with Firebase requires a few clicks on the web console, and some commands on your favorite shell:
```shell
$ npm i firebase
$ firebase login
$ firebase deploy
```
For these commands to work properly a `firebase.json` file is required in the project folder. We'll check it out later.

# TensorFlow
[TensorFlow](https://www.tensorflow.org/) is a famous open-source machine learning platform.  
Using TensorFlow doesn't require much machine learning knowledge. One could simply pick up one of the [many pre-trained
models from the zoo](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md) and use it straight out of the box.

That's what we did initially. We picked one of the TensorFlowJS models, precisely [CocoSSD](https://github.com/tensorflow/tfjs-models/tree/master/coco-ssd), and tried to recognize... stuff. The first thing the model recognized was a cute dog.
![Doggo](/posts/front-end-only-ml/doggo.png)

Cool! Although, we want to recognize numbers, not cute animals. The solution is to retrain the CocoSSD model.
Retraining a model requires quite the infrastructure, lets list all the things!

- **Python**: we used [Anaconda](https://www.anaconda.com/) to share the same Conda Environment.

- **A data set and its records**: at first we thought about [Google's SVHN Dataset](http://ufldl.stanford.edu/housenumbers/), however setting it up requires an excessive amount of time. We opted for a custom data set, that provides `.csv` files to generate the images records.

- **Augmented images to increase accuracy**: using [this tool](https://github.com/Paperspace/DataAugmentationForObjectDetection) we were able to augment the images (i.e. rotation and mirroring).
- **A detection model, duh**: at first we opted for `faster_rcnn_inception_v2_coco`.

- **Some way to track the training progress**: TensorFlow comes with a neat tool, [TensorBoard](https://www.tensorflow.org/tensorboard)!

The model took some days to train completely.

# Model conversion
The training's output is an image tensor, it needs to be converted to a TensorFlowJS graph.  
No worries, [tensorflowjs_converter](https://github.com/tensorflow/tfjs-converter) is here! Let's write a simple script to export the image tensor to a saved model which will then be converted to a TensorFlowJS graph:

```bash
#!/usr/bin/bash

set -e

OBJECT_DETECTION_PATH='path/to/models/research'
PROJECT_PATH=.
CKPT_NO=420

# clean saved model
rm -rf ${PROJECT_PATH}/object_detection/saved_model

# export image tensor
python ${OBJECT_DETECTION_PATH}/object_detection/export_inference_graph.py
  --input_type=image_tensor
  --pipeline_config_path=${PROJECT_PATH}/object_detection/training/pipeline.config
  --trained_checkpoint_prefix=${PROJECT_PATH}/object_detection/training/model.ckpt-${CKPT_NO}
  --output_directory=${PROJECT_PATH}/object_detection/saved_model

# convert saved model
# tensorflowjs_converter 0.8.6:
tensorflowjs_converter
  --quantization_bytes=2
  --strip_debug_ops=true
  --input_format=tf_saved_model
  --output_format=tensorflowjs
  --saved_model_tags=serve
  --output_json=true
  --output_node_names='detection_boxes,detection_multiclass_scores'
  ${PROJECT_PATH}/object_detection/saved_model/saved_model
  ${PROJECT_PATH}/src/assets/web_model

```

[That's a lot of scripting!](https://i.imgur.com/p3Vd29z.jpg) The key points are the `quantization_bytes` and the `output_node_names` parameters.
The first one 



# other
The [first requirement](#, "The application must be front end only") means that the image recognition has to run on the client machine.
This practice avoids any sending/receiving of images between front and back end.
Neat! Does that mean we already satisfied the 
[second requirement](#, "The recognition time should've been equal or faster then Google Cloud Vision")?  
**Nope.**  
Unfortunately the first time a client opens our application, it must download the 


> THE ERROR
> Benchmark



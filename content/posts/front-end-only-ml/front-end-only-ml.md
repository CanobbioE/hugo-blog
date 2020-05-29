---
title: "From 1k to 100"
date: 2020-05-29T00:52:56+02:00
showDate: true
draft: false
tags: ["blog","front end","machine learning", "angular"]
images:
    - /posts/front-end-only-ml/tensorboard.PNG
    - /posts/front-end-only-ml/model.PNG
    - /posts/front-end-only-ml/comp.PNG
---

# What is this?
This is the story of how an ugly ~~little duckling~~ slow machine learning web-app grew up to be a ~~beautiful swan~~ fast front-end-only application.

# The setting
Once upon a time, in a country far far away, two software developers were instructed to improve a front end only web application
capable to perform numbers recognition on photos.

I happen to be one of those two developers ['o'](https://en.meming.world/images/en/6/6e/Surprised_Pikachu.jpg).  
The initial requirements for the application were simple:
- The application must be front end only
- Each number should be recognized as an object (i.e. "123" outputs "1", "2", "3")
- The recognition time should've been equal or faster then [Google Cloud Vision](https://cloud.google.com/vision/)

This last point being the most important one. When we got access to the application's source code it was using third parties services to perform number recognition:
 [Google Cloud Vision](https://cloud.google.com/vision/) and [Tesseract](https://github.com/tesseract-ocr/tesseract). These services are great, but the latency made every single recognition take up to 13 seconds.

# The solution
What do we do when we want something to **not** rely on the user's connection speed? Exactly, we use a client-side solution.

Enlightened by this revelation, we used [TensorFlow](https://www.tensorflow.org/) and its JavaScript counterpart: [TensorFlowJS](https://www.tensorflow.org/js) to perform image recognition.
[TensorFlow](https://www.tensorflow.org/) is a famous open-source machine learning platform.  
Using it doesn't require much machine learning knowledge. It is as easy as picking one of the [many pre-trained
models from the zoo](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md) and using it straight out of the box.

Sounds fun and easy, let's do it!

# Oh no!
**There is no model that recognize number's digit as objects (i.e. 123 as "1", "2" and "3")!**

No worries, we can train our model using a bit of Python and [Google's SVHN Dataset](http://ufldl.stanford.edu/housenumbers/).
Using a generous amount of patience and a pinch of [TensorBoard](https://www.tensorflow.org/tensorboard), we'll end up with our glorious trained model!
![TensorBoard](/posts/front-end-only-ml/tensorboard.PNG)
Disclaimer: TensorBoard is not required, but it really helps to oversee the training's process.

Finally, the model must be converted to the format supported by TensorFlowJS.  
Of course there's a CLI for that, [tensorflowjs_converter](https://github.com/tensorflow/tfjs-converter) is the tool for the job! Let's write a simple script to export the image tensor to a saved model which will then be converted into a TensorFlowJS graph:

```bash
#!/usr/bin/bash

set -e

# constants
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
# tensorflowjs_converter v0.8.6:
tensorflowjs_converter
  --quantization_bytes=4
  --strip_debug_ops=true
  --input_format=tf_saved_model
  --output_format=tensorflowjs
  --saved_model_tags=serve
  --output_json=true
  --output_node_names='detection_boxes,detection_multiclass_scores'
  ${PROJECT_PATH}/object_detection/saved_model/saved_model
  ${PROJECT_PATH}/src/assets/web_model
```
[That's a lot of scripting!](https://i.imgur.com/p3Vd29z.jpg)
Let's ignore the exporting script and take a look at the conversion command. The important bits are:
- `quantization_bytes`: higher number means bigger file's size. Lower number means worst performance. The boundaries go from 1 to 4, I found that 2 is the perfect value.
- `output_node_names`: specifies which tensors to export, the names varies from model to model. We are selecting only the tensor with the detection boxes' coordinates and the detection details.
- `output_format`: we obviously want this to be `tensorflowjs`.

What we have obtained so far is a bunch of files named `group1-shard1of7` and `model.json`, pretty unimpressive. Let's see how we can use them.

# Almost done, I just need to...
Let's fast forward a bit by skipping all the coding part.  
Probably I should answer some questions:
- The model? Imported in our code!
- The recognition service? Rewritten entirely with Angular!
- The recognition time? Uhm...
- The RECOGNITION TIME??! Uhmm, please do the meme thing!
- ...
- Hotel? TRIVAGO! :D

Thanks for going through this terrible joke, however, what's the recognition time?
Well, here's a little riddle for you: the first recognition took 20s but the next ones only took between 100 and 200ms, how is that possible?

Did you get it? The problem is that the client (a.k.a. our browser) has to download the whole model before starting to recognize images.  
Do you remember that option `quantization_bytes` in the conversion script? Well it was set to 4, lets change it to 2 and our model will be lighter.

Now the download is a bit faster, but still slow, sadly we can't go any lower in size. Fortunately, however, it's a one time only issue and it's fine for our use case.

# ...test it on mobile!
Let's now try on a smartphone! The recognition time is...  
```bash
Uncaught Error: Requested texture size [5120x5120] greater than WebGL maximum on this browser / GPU [4096x4096]
```  
The explanation is that we are using WebGL, which uses the GPU to perform image recognition. Additionally, the tensors are converted to textures, and each GPU has a maximum texture size, in this case the array dimensions cannot be more than `4096x4096`.  
The [only solution](https://github.com/tensorflow/tfjs/issues/689#issuecomment-503590183) to this problem is to chose a different starting model, in our case we used `ssd_mobilenet_v2_coco`, instead of the previously selected `faster_rcnn_inception_v2_coco`.

# Conclusion
After retraining the new model the result is an extremely fast front-end-only application that recognize numbers in under 200ms.
![comparison](/posts/front-end-only-ml/comp.PNG)

Most importantly, now we also have the nice detection boxes, courtesy of the option `output_node_names='detection_boxes'`.  

This story has a moral, and that is that even the ugliest and slowest website can grow into a nice and fast application!  
No, wait! Probably the moral is that there is always room for improvement, always try to improve your code to the best of your abilities!


\***Insert motivational music and image.**\*
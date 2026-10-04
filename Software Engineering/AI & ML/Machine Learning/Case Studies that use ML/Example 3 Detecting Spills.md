---
title: "Example 3 Detecting Spills"
tags: [ai-ml, ai-ml/machine-learning, course]
aliases: [Detecting Spills, Spill Detection CNN]
updated: 2026-09-27
---
# Example 3 Detecting Spills

<Note>
**TL;DR**

A **deep learning image classification** case study: detect chemical spills in a plant's security camera footage so the clean-up team responds faster. Build a dataset of spill / no-spill images (including safely staged spills), convert pixels to numbers, train a **convolutional neural network (CNN)**, and evaluate with **precision and recall**, not accuracy, because spills are rare. Deploy at the edge and page the team on detection.
</Note>

The first two examples used classical methods (linear models, k-means). This one uses a more modern model type.

<Note>
This example uses a neural network. How neural networks work was beyond the course lesson; see [Neural Networks & Deep Learning Basics](/Software%20Engineering/AI%20%26%20ML/Neural%20Networks%20%26%20Deep%20Learning%20Basics).
</Note>

## Step 1: Define the problem
You run a specialised on-site janitorial company. One client, an industrial chemical plant, needs a fast response to spills and other hazards. If you could **automatically** detect spills with the plant's surveillance cameras, you could mobilise the team faster.

![Untitled 10](/Attachments/Untitled%2010.png)

**Choosing the task**: predict which class each image belongs to:
- contains spill
- does not contain spill

That's **binary image classification** (supervised).

![Untitled 11](/Attachments/Untitled%2011.png)

## Step 2: Build the dataset
- **Collect**: use historical footage plus **safely staged spills** to quickly gather images with and without spills in multiple lighting conditions and environments.
- **Explore and clean**: check every photo so the spill is clearly in the shot. Image-enhancement tools can help later if you need to iterate.
- **Vectorise**: models need numbers, so each pixel becomes a number, e.g. between 0 (black) and 1 (white) for grayscale, or three numbers per pixel for colour.

![Untitled 12](/Attachments/Untitled%2012.png)

![Untitled 13](/Attachments/Untitled%2013.png)

- **Split** into training and test datasets.

## Step 3: Train the model
Traditionally you'd hand-engineer features on top of pixels (edges, corners) and train a model on those. Today, deep neural networks **learn the features themselves** from raw pixels.

**CNN (convolutional neural network)**: a neural network is a collection of very simple models (**neurons**) connected by trainable parameters (**weights**). A CNN adds layers of small learned filters that slide across the image, so early layers detect edges and textures and deeper layers detect shapes like a puddle on a floor. CNNs are particularly good at images.

In practice you'd rarely train from scratch: take a CNN (or vision transformer) **pretrained** on millions of images and **fine-tune** it on your spill images (transfer learning), which needs far less data.

```python
import torch, torchvision
model = torchvision.models.resnet18(weights="IMAGENET1K_V1")   # pretrained CNN
model.fc = torch.nn.Linear(model.fc.in_features, 2)          # new head: spill / no spill
# then train on your labelled images for a few epochs
```

## Step 4: Evaluate the model
Common classification metrics: accuracy, confusion matrix, F1 score, false positive rate, false negative rate, log loss, negative predictive value, precision, recall, ROC curve, specificity.

**Why not accuracy?** The model sees "no spill" almost all the time, so a model that always predicts "no spill" would look very accurate and never catch anything.

What you care about is a model that **rarely misses a real spill**. **Precision** answers "of all spill predictions, how many were right?" and **recall** answers "of all actual spills, how many did we detect?" Here recall matters most, with enough precision that the team isn't flooded with false alarms. See [Step 4 Evaluate the model](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%204%20Evaluate%20the%20model).

**Manual evaluation** matters: if you're unsure the staged spills look like real ones, test on real spills from historical records to confirm the model works on the real thing.

## Step 5: Use the model (inference)
Deploy the model on a system that runs ML workloads next to the cameras. The course used **AWS Panorama** (an edge appliance). *Update: AWS ended support for AWS Panorama on 2026-05-31, so today you'd pick another edge runtime (for example an NVIDIA Jetson device, or AWS IoT Greengrass running your model).* Most of the time the result is "does not contain spill".

![Untitled 14](/Attachments/Untitled%2014.png)

When "contains spill" is detected, a simple paging system alerts the team.

![Untitled 15](/Attachments/Untitled%2015.png)

## Takeaways
1. Some problems need more complex techniques. Neural networks are powerful but harder to **explain**.
2. High-quality data was critical, to the point of staging fake spills. Images again had to be **vectorised** into numbers.
3. During inference you keep inspecting predictions, especially because some training data was staged.

## Terminology
- **Neural network**: a collection of very simple models connected together. The simple models are **neurons**; the trainable connections are **weights**.
- **Convolutional neural network (CNN)**: a neural network specialised for grid data such as images.

## Interview questions

<Accordion title="Why fine-tune a pretrained model instead of training from scratch?">
Pretrained models already know general visual features (edges, textures, shapes). Fine-tuning reuses that knowledge, so you need far fewer labelled images, train faster and usually get better accuracy.
</Accordion>

<Accordion title="The model will run on edge cameras. What production concerns come up?">
Model size and latency on limited hardware (quantisation, smaller architectures), offline operation, how to update models on devices, monitoring drift (lighting, new camera angles), privacy of video data, and false-alarm rates that decide whether people trust the alerts.
</Accordion>

## Related
- [Case Studies that use ML](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Case%20Studies%20that%20use%20ML) · [Machine Learning](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Machine%20Learning) (course index) · [AI & ML Index](/Software%20Engineering/AI%20%26%20ML/AI%20%26%20ML%20Index)
- Previous: [Example 2 Micro-genre Book Genre](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Example%202%20Micro-genre%20Book%20Genre)
- [Neural Networks & Deep Learning Basics](/Software%20Engineering/AI%20%26%20ML/Neural%20Networks%20%26%20Deep%20Learning%20Basics) · [Step 4 Evaluate the model](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%204%20Evaluate%20the%20model)

## Sources
- AWS DeepRacer Student: [Introduction to machine learning](https://student.deepracer.com/learn/85a3a216-703b-40c8-ad70-374767eba381)
- AWS: [AWS Panorama end of support (2026-05-31)](https://docs.aws.amazon.com/panorama/latest/dev/panorama-end-of-support.html)
- PyTorch: [Transfer learning for computer vision tutorial](https://pytorch.org/tutorials/beginner/transfer_learning_tutorial.html)
- Stanford CS231n: [Convolutional Neural Networks](https://cs231n.github.io/convolutional-networks/)

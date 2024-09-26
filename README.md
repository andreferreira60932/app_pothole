# Object Detection with YOLOv5 on Android

## Introduction

[YOLO](https://pjreddie.com/darknet/yolo/) (You Only Look Once) is one of the fastest and most popular object detection models. [YOLOv5](https://github.com/ultralytics/yolov5) is an open-source implementation of the latest version of YOLO (for a quick test of loading YOLOv5 from PyTorch hub for inference, see [here](https://pytorch.org/hub/ultralytics_yolov5/#load-from-pytorch-hub)). This Object Detection with YOLOv5 Android sample app uses the PyTorch scripted YOLOv5 model to detect objects of the [80 classes](https://github.com/ultralytics/yolov5/blob/master/data/coco.yaml) trained with the model.

**Update 09-30-2021**: A new section of using a custom dataset to fine-tune the YOLOv5 model (aka transfer learning) and steps to change the Android project to use the custom model was added.

## Prerequisites

* PyTorch 1.10.0 and torchvision 0.11.1 (Optional)
* Python 3.8 (Optional)
* Android Pytorch library pytorch_android_lite:1.10.0, pytorch_android_torchvision_lite:1.10.0
* Android Studio 4.0.1 or later

## Quick Start

To Test Run the Object Detection Android App, follow the steps below:

### 1. Prepare the model

If you don't have the PyTorch environment set up to run the script, you can download the model file `yolov5s.torchscript.ptl` [here](https://pytorch-mobile-demo-apps.s3.us-east-2.amazonaws.com/yolov5s.torchscript.ptl) to the `android-demo-app/ObjectDetection/app/src/main/assets` folder, then skip the rest of this step and go to step 2 directly.

The Python script `export.py` in the `models` folder of the [YOLOv5 repo](https://github.com/ultralytics/yolov5) is used to generate a TorchScript-formatted YOLOv5 model named `yolov5s.torchscript.pt` for mobile apps.

Open a Mac/Linux/Windows Terminal, run the following commands (note that we use the fork of the original YOLOv5 repo to make sure the code changes work, but feel free to use the original repo):

```
git clone https://github.com/ultralytics/yolov5
cd yolov5
pip install -r requirements.txt wanb
```

Note the steps below have been tested with the commit `cd35a009ba964331abccd30f6fa0614224105d39` and if there's any issue with running the script or using the model, try `git reset --hard cd35a009ba964331abccd30f6fa0614224105d39`.

Edit `export.py` to make the following two changes:

* After `f = file.with_suffix('.torchscript.pt')`, add a line `fl = file.with_suffix('.torchscript.ptl')`

* After `(optimize_for_mobile(ts) if optimize else ts).save(f)`, add `(optimize_for_mobile(ts) if optimize else ts)._save_for_lite_interpreter(str(fl))`

Now run the script below to generate the optimized TorchScript lite model `yolov5s.torchscript.ptl` and copy it to the `android-demo-app/ObjectDetection/app/src/main/assets` folder (the original full JIT model `yolov5s.torchscript.pt` was also generated for comparison):

```
python export.py --weights yolov5s.pt --include torchscript
```

Note that small sized version of the YOLOv5 model, which runs faster but with less accuracy, is generated by default when running the `export.py`. You can also change the value of the `weights` parameter in the `export.py` to generate the medium, large, and extra large version of the model.

### 2. Build with Android Studio

Start Android Studio, then open the project located in `android-demo-app/ObjectDetection`. Note the app's `build.gradle` file has the following lines:

```
implementation 'org.pytorch:pytorch_android_lite:1.10.0'
implementation 'org.pytorch:pytorch_android_torchvision_lite:1.10.0'
```

### 3. Run the app

Select an Android emulator or device to run the app. You can go through the included example test images to see the detection results. You can also select a picture from your Android device's Photos library, take a picture with the device camera, or even use live camera to do object detection - see this [video](https://drive.google.com/file/d/1-5AoRONUqZPZByM-fy0m7r8Ct11OnlIT/view) for a screencast of the app running.

Some example images and the detection results are as follows:

![app_yolo5_1](https://github.com/user-attachments/assets/2f80dd07-a293-4a6f-af4a-f36f79ce14a3)

![app_yolo5_2](https://github.com/user-attachments/assets/6eb99d77-52fa-4c31-9cf0-7aa05dc7afda)


## Transfer Learning

In this section, you'll see how to use an example dataset called [aicook](https://universe.roboflow.com/karel-cornelis-q2qqg/aicook-lcv4d/4), used to detect ingredients in your fridge, to fine-tune the YOLOv5 model. For more info on the YOLOv5 transfer learning, see [here](https://github.com/ultralytics/yolov5/issues/1314). If you use the default YOLOv5 model to do object detection on what's inside your fridge, you'll likely not get good results. That's why you need to have a custom model trained with a dataset like aicook.

### 1. Download the custom dataset

Simply go to [here](https://universe.roboflow.com/karel-cornelis-q2qqg/aicook-lcv4d/4) to download the aicook dataset in a zip file. Unzip the file to your `yolov5` repo directory, then run `cd yolov5; mv train ..; mv valid ..;` as the aicook `data.yaml` specifies the `train` and `val` folders to be up one level.

### 2. Retrain the YOLOv5 with the custom dataset

Run the script below to generate a custom model `best.torchscript.pt` located in `runs/train/exp/weights`:

```
python train.py --img 640 --batch 16 --epochs 3 --data  data.yaml  --weights yolov5s.pt
```

The precision of the model with the epochs set as 3 is very low - less than 0.01 actually; with a tool such as [Weights and Biases](https://wandb.ai), which can be set up in a few minutes and has been integrated with YOLOv5, you can find that with `--epochs` set as 80, the precision gets to be 0.95. But on a CPU machine, you can quickly train a custom model using the command above, then test it in the Android demo app. Below is a sample wandb metrics from 3, 30, and 100 epochs of training:

![](metrics.png)

### 3. Convert the custom model to lite version

With the `export.py` modified in step 1 `Prepare the model` of the section `Quick Start`, you can convert the new custom model to its TorchScript lite version:

```
python export.py --weights runs/train/exp/weights/best.pt --include torchscript
```

The resulting `best.torchscript.ptl` is located in `runs/train/exp/weights`, which needs to be copied to the Android demo app's assets folder.

### 4. Update the demo app

In Android Studio, first in `MainActivity.java`, change line:

```
private String[] mTestImages = {"test1.png", "test2.jpg", "test3.png"};
```
and change lines:
```
mModule = LiteModuleLoader.load(MainActivity.assetFilePath(getApplicationContext(), "yolov5s.torchscript.ptl"));
BufferedReader br = new BufferedReader(new InputStreamReader(getAssets().open("classes.txt")));
```
to:
```
mModule = LiteModuleLoader.load(MainActivity.assetFilePath(getApplicationContext(), "best.torchscript.ptl"));
BufferedReader br = new BufferedReader(new InputStreamReader(getAssets().open("aicook.txt")));
```
(aicook.txt defines the 30 custom class names, copied from `data.yaml` in the custom dataset downloaded in step 1.)

Then in `PrePostProcessor.java`, change line `private static int mOutputColumn = 85;` to `private static int mOutputColumn = 35;`.

Run the app in Android Studio and you should see the custom model working on the first three aicook test images:

![](aicook1.png)
![](aicook2.png)

![](aicook3.png)
![](aicook4.png)

In order to do live object detection with the new custom model, just open `ObjectDetectionActivity.java` and replace `yolov5s.torchscript.ptl` in `mModule = LiteModuleLoader.load(MainActivity.assetFilePath(getApplicationContext(), "yolov5s.torchscript.ptl"));` with `best.torchscript.ptl`.

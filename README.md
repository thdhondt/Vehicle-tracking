# Vehicle tracking

## Introduction

Autonomous vehicles need to understand the environment in which they evolve in order to adapt their behavior. The car uses sensors for this purpose, which gives it information on its surroundings in a step called perception. During this task, the vehicle should also be able to detect and localize other road users such as vehicles. The aim of the present project is to extract the position of cars in an image captured by a front facing cameras. Computer vision algorithms are combined with machine learning techniques for this purpose. The positions of those vehicles can thereafter be used in control and planing algorithms, so that the ego-vehicle can drive safely on the road.

Firstly, a support vector machine is trained in order to predict whether a car is present in a given image or not. This classifier is then applied over the images fed by the camera, using a windowing approach. The window sweeps over the image and the positive predictions are saved. Finally, the detected windows are averaged over several consecutive frames in order to limit the effect of false-positives.

## Data exploration

A data set containing images of vehicles and non-vehicles is needed in order to train the classifier. This data set is the result of a combination of the KITTI , the Udacity and the GTI sets. This results in the following total amount of images. One should note that the amount of images in both classes is quite similar. This is needed in order to avoid a prediction bias towards a class which is more present in the data set.

| Parameter  | Value |
| ------------- | ------------- |
| Total number of vehicles  | 11303  |
| Total number of non-vehicles  | 8968  |

All images are saved in a 64 by 64 pixels resolution and with three color channels. A small amount of them are plotted in the following figure.

![enter image description here](https://github.com/thdhondt/Vehicle-tracking/blob/master/Schematics/set.PNG?raw=true)

No data augmentation scheme has been used in the present project, since the initial data was well balanced.

## Training

The first step of the training process is the splitting of the data set into a training set and a test set. The latter is used in order to verify that the classifier is not over-fitting on the former. Over-fitting is the result of the classifier memorizing the entire training set, which would induce a very high training accuracy, but bad generalization to new data. Therefore, a test set is used to verify the performance of a classifier on data it has not yet seen, which gives an idea about its ability to generalize. It was decided to keep 20% of the total data set as a test set, which results in the following numbers.

| Parameter  | Value |
| ------------- | ------------- |
| Total number of train images  | 14207  |
| Total number of test images  | 3553  |
| Test to train ratio| 0.20005|
| Ratio of vehicles to non-vehicles in the train set | 0.98|
| Ratio of vehicles to non-vehicles in the test set | 0.98|

One may observe that the split ratio is well respected and that both the training and the test set are balanced. One of the provided data sets included images extracted from consecutive video frames. Hence, several images in this set are very close to one another. If those images of a same video segment are split over both the test and the train set, the test set would no longer serve its purpose. Therefore, the sets have been split in two continuous list of images, before being shuffled.

The support vector machine needs features as an input, on which it will base itself for its prediction. Those features should contain enough information to allow the algorithm to differentiate a vehicle from a non-vehicle. On the other hand, the higher the dimension of the feature space, the longer the training and the online prediction will take. A good trade off should be found between performance and accuracy.

The information that's included in an image can be summarized as:
- Color information
- Gradient information

A first feature is obtained by spatial binning of the image, which means that the resolution of the image is greatly reduced. This results in a much lower amount of pixels, but which still allow to recognize the car. The color information stored in those pixels is then put in a feature vector.

A second feature is obtained by removing all spatial information of the image and only keeping the color information. For this purpose, the different pixels in the channels of the image are binned according to their intensities. This results in a color histogram of the image, where the highest bins correspond to the principal color components of the image.

Finally, the gradient is used as a feature through the HOG algorithm. For this purpose, the image is divided in a grid. The gradient orientation and amplitude is then computed for each pixel in the image. The gradient intensity for each pixel is then binned according to the associated orientation, which results in information on the main gradients in a given cell.

The different features that are fed to the SVM are computed considering the following settings, which result in the best classification performance.
| Feature type | Parameter  | Value |
| ------------- |---------- | ------------- |
| Spatial binning  | Size   | 32|
||Color space|YCrCb|
|Histogram binning|Number of bins|32|
||Color space|YCrCb|
|HOG|Orientations|9|
||Pix per cell|8|
||Pix per block|2|
||Channels|all|
||Color space|YCrCb|

All the previously computed features are concatenated in one big input vector, which is then normalized over the training set. This last steps avoids a classifier that relies more heavily on high amplitude features for its prediction. The training of the SVM results in a test accuracy of 99.01%

![enter image description here](https://github.com/thdhondt/Vehicle-tracking/blob/master/Schematics/classification.PNG?raw=true)


## Multi-scale windowing

The previously described classifier is able to predict with high accuracy whether a car is present or not in an image with a fixed size. This classifier can then be applied on different parts of an image captured by a front-facing camera. By moving the region of interest all over this image, one is able to determine which of those regions contain a car or not. In other words, a window is moved across the image and the content of this window is fed to the classifier.

Moreover, the size of a vehicle will vary with its distance to our camera. Therefore, its size will not always match that of the input of the classifier. Therefore, the size of the window may be varied as well, which is called multi-scale windowing. The content of this window is then resized to the correct input size of the classifier. An example of different window positions and sizes is shown in the following image.

![enter image description here](https://github.com/thdhondt/Vehicle-tracking/blob/master/Schematics/windows.PNG?raw=true)

The features are extracted for each window size and position. Those features are fed to the classifier for the predictions. Windows with positive returns are memorized for the following steps. The entire process of detection on a given video frame is summarized in the following image.

![enter image description here](https://github.com/thdhondt/Vehicle-tracking/blob/master/Schematics/detection.PNG?raw=true)

The output of this processing step for an example frame can be seen in this image.

![enter image description here](https://github.com/thdhondt/Vehicle-tracking/blob/master/Schematics/detections.PNG?raw=true)

## Time averaging

This classification step usually results in false-positives, where the classifier predicts that a car is present in windows that do not contain a car. Such detections should be filtered, since they could result in harsh braking of our autonomous car. This can be achieved by averaging the predictions over several consecutive frames.

For this purpose, the windows computed for the N previous predictions are kept in a FIFO stack. A cumulative heat map is then computed over this stack. By thresholding, low heat values in the heat map, the false positives can be removed. Indeed, the position of those false predictions tend to be a lot more variable, whereas the prediction on actual cars is more stable. Finally, the total bounding box is extracted from the heat map, which is described in the following block diagram.


![enter image description here](https://github.com/thdhondt/Vehicle-tracking/blob/master/Schematics/frame_averaging.PNG?raw=true)

The output of this processing step is a unique bounding box per car, which is more accurate and more stable.

![enter image description here](https://github.com/thdhondt/Vehicle-tracking/blob/master/Schematics/grouping.PNG?raw=true)

## Result

The previously described steps were grouped in a pipeline and applied on two videos. This resulted in good detection performance. Therefore, the aim of this project is achieved.

In future work, one should focus on improving the computational speed of such an algorithm. Indeed, the windowing approach is not very efficient and real-time constraints are very important in time critical applications such as autonomous vehicles. Additionally, a perspective transform could be used to extract the relative position of the other vehicles with respect to the ego car. Tracking algorithms combined with kalman filters can be used to estimate the state of all actors in the environment, hereby filtering their position and estimating their velocity.

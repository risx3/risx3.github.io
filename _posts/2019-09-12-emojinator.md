---
title: "Emojinator"
date: 2019-09-12
tags: [Emojinator, emoji, computer vision, keras, machine learning]
excerpt: "Classification of Emojis using Convolutional Neural Network."
header:
  overlay_image: "/images/emojinator/emojinator.jpg"
  caption: "Emojinator"
mathjax: "true"
---

## In this project

In this project, we will create our data set of emojis that can be created through hand gestures. We will collect hundreds of image samples for particular emoji.

This project is implemented in Python 3.7.

And, the libraries used are-

1. [Numpy](https://numpy.org/)
2. [Pandas](https://pandas.pydata.org/)
3. [TensorFlow](https://www.tensorflow.org/)
4. [Keras](https://keras.io/)
5. [OpenCV](https://opencv-python-tutroals.readthedocs.io/en/latest/py_tutorials/py_gui/py_image_display/py_image_display.html)
6. [imageio](https://pypi.org/project/imageio/)

## Design

First, we will create a program that will capture all the gestures for our dataset. 

Next, we will convert that dataset into CSV format which we will use in our code.

We will use this file in creating our model. Once the model is created, we will implement the classification of our real-time gestures.

<img src="{{ site.url }}{{ site.baseurl }}/images/emojinator/emojinator_diagram.jpg" alt="emojinator flow">

## Creating Gestures (CreateGestures.py)

```python
import cv2
import numpy as np
import os
```

```python
image_x, image_y = 50, 50
```


```python
def create_folder(folder_name):
    if not os.path.exists(folder_name):
        os.mkdir(folder_name)
```


```python
def store_images(g_id):
    total_pics = 1200
    cap = cv2.VideoCapture(0)
    x, y, w, h = 300, 50, 350, 350

    create_folder("gestures/" + str(g_id))
    pic_no = 0
    flag_start_capturing = False
    frames = 0
    while True:
        ret, frame = cap.read()
        frame = cv2.flip(frame, 1)
        hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)

        mask2 = cv2.inRange(hsv, np.array([2, 50, 60]), np.array([25, 150, 255]))
        res = cv2.bitwise_and(frame, frame, mask=mask2)
        gray = cv2.cvtColor(res, cv2.COLOR_BGR2GRAY)
        median = cv2.GaussianBlur(gray, (5, 5), 0)

        kernel_square = np.ones((5, 5), np.uint8)
        dilation = cv2.dilate(median, kernel_square, iterations=2)
        opening=cv2.morphologyEx(dilation,cv2.MORPH_CLOSE,kernel_square)

        ret, thresh = cv2.threshold(opening, 30, 255, cv2.THRESH_BINARY)
        thresh = thresh[y:y + h, x:x + w]
        contours = cv2.findContours(thresh.copy(), cv2.RETR_TREE, cv2.CHAIN_APPROX_NONE)[0]
        if len(contours) > 0:
            contour = max(contours, key=cv2.contourArea)
            if cv2.contourArea(contour) > 10000 and frames > 50:
                x1, y1, w1, h1 = cv2.boundingRect(contour)
                pic_no += 1
                save_img = thresh[y1:y1 + h1, x1:x1 + w1]
                if w1 > h1:
                    save_img = cv2.copyMakeBorder(save_img, int((w1 - h1) / 2), int((w1 - h1) / 2), 0, 0, cv2.BORDER_CONSTANT, (0, 0, 0))
                elif h1 > w1:
                    save_img = cv2.copyMakeBorder(save_img, 0, 0, int((h1 - w1) / 2), int((h1 - w1) / 2), cv2.BORDER_CONSTANT, (0, 0, 0))
                save_img = cv2.resize(save_img, (image_x, image_y))
                cv2.putText(frame, "Capturing...", (30, 60), cv2.FONT_HERSHEY_TRIPLEX, 2, (127, 255, 255))
                cv2.imwrite("gestures/" + str(g_id) + "/" + str(pic_no) + ".jpg", save_img)

        cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 255, 0), 2)
        cv2.putText(frame, str(pic_no), (30, 400), cv2.FONT_HERSHEY_TRIPLEX, 1.5, (127, 127, 255))
        cv2.imshow("Capturing gesture", frame)
        cv2.imshow("thresh", thresh)
        keypress = cv2.waitKey(1)
        if keypress == ord('c'):
            if flag_start_capturing == False:
                flag_start_capturing = True
            else:
                flag_start_capturing = False
                frames = 0
        if flag_start_capturing == True:
            frames += 1
        if pic_no == total_pics:
            break
```

```python
g_id = input("Enter gesture number: ")
store_images(g_id)
```

## Create CSV (createcsv.py)

```python
import imageio
import numpy as np
import pandas as pd
import os
root = './gestures' # or ‘./test’ depending on for which the CSV is being created
```

```python
# go through each directory in the root folder given above
for directory, subdirectories, files in os.walk(root):
    # go through each file in that directory
    for file in files:
    # read the image file and extract its pixels
        print(file)
        im = imageio.imread(os.path.join(directory,file))
        value = im.flatten()
        # I renamed the folders containing digits to the contained digit itself. For example, digit_0 folder was renamed to 0.
        # so taking the 9th value of the folder gave the digit (i.e. "./gestures/8" ==> 9th value is 8), which was inserted into the first column of the dataset.
        value = np.hstack((directory[11:],value))
        df = pd.DataFrame(value).T
        df = df.sample(frac=1) # shuffle the dataset
        with open('train_foo.csv', 'a') as dataset:
            df.to_csv(dataset, header=False, index=False)
```

## Creating Model (model.py)

```python
import numpy as np
from keras import layers
from keras.layers import Input, Dense, Activation, ZeroPadding2D, BatchNormalization, Flatten, Conv2D
from keras.layers import AveragePooling2D, MaxPooling2D, Dropout, GlobalMaxPooling2D, GlobalAveragePooling2D
from keras.utils import np_utils, print_summary
from keras.models import Sequential
from keras.callbacks import ModelCheckpoint
import pandas as pd
import keras.backend as K
```

```python
data = pd.read_csv("train_foo.csv")
dataset = np.array(data)
np.random.shuffle(dataset)
X = dataset
Y = dataset
X = X[:, 1:2501]
Y = Y[:, 0]
X_train = X[0:12000, :]
X_train = X_train / 255.
X_test = X[12000:13201, :]
X_test = X_test / 255.
```

```python
# Reshape
Y = Y.reshape(Y.shape[0], 1)
Y_train = Y[0:12000, :]
Y_train = Y_train.T
Y_test = Y[12000:13201, :]
Y_test = Y_test.T

print("number of training examples = " + str(X_train.shape[0]))
print("number of test examples = " + str(X_test.shape[0]))
print("X_train shape: " + str(X_train.shape))
print("Y_train shape: " + str(Y_train.shape))
print("X_test shape: " + str(X_test.shape))
print("Y_test shape: " + str(Y_test.shape))
```

### So, let's see what we have here...

> number of training examples = 12000<br/>
> number of test examples = 1199<br/>
> X_train shape: (12000, 2500)<br/>
> Y_train shape: (1, 12000)<br/>
> X_test shape: (1199, 2500)<br/>
> Y_test shape: (1, 1199)<br/>

### Back to code...

```python
image_x = 50
image_y = 50
train_y = np_utils.to_categorical(Y_train)
test_y = np_utils.to_categorical(Y_test)
train_y = train_y.reshape(train_y.shape[1], train_y.shape[2])
test_y = test_y.reshape(test_y.shape[1], test_y.shape[2])
X_train = X_train.reshape(X_train.shape[0], 50, 50, 1)
X_test = X_test.reshape(X_test.shape[0], 50, 50, 1)
print("X_train shape: " + str(X_train.shape))
print("X_test shape: " + str(X_test.shape))
print("Y_train shape: " + str(train_y.shape))
```

### Output for this is...

> X_train shape: (12000, 50, 50, 1)<br/>
> X_test shape: (1199, 50, 50, 1)<br/>
> Y_train shape: (12000, 12)<br/>

### Back to code...

```python
def keras_model(image_x, image_y):
    num_of_classes = 12
    model = Sequential()
    model.add(Conv2D(32, (5, 5), input_shape=(image_x, image_y, 1), activation='relu'))
    model.add(MaxPooling2D(pool_size=(2, 2), strides=(2, 2), padding='same'))
    model.add(Conv2D(64, (5, 5), activation='sigmoid'))
    model.add(MaxPooling2D(pool_size=(5, 5), strides=(5, 5), padding='same'))
    model.add(Flatten())
    model.add(Dense(1024, activation='relu'))
    model.add(Dropout(0.6))
    model.add(Dense(num_of_classes, activation='softmax'))
    model.compile(loss='categorical_crossentropy', optimizer='adam', metrics=['accuracy'])
    filepath = "handEmo.h5"
    checkpoint1 = ModelCheckpoint(filepath, monitor='val_acc', verbose=1, save_best_only=True, mode='max')
    callbacks_list = [checkpoint1]
    return model, callbacks_list
```

```python
model, callbacks_list = keras_model(image_x, image_y)
model.fit(X_train, train_y, validation_data=(X_test, test_y), epochs=10, batch_size=64, callbacks=callbacks_list)
scores = model.evaluate(X_test, test_y, verbose=0)
print("CNN Error: %.2f%%" % (100 - scores[1] * 100))
print_summary(model)
model.save('handEmo.h5')
```

    CNN Error: 0.00%
    Model: "sequential_1"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    conv2d_1 (Conv2D)            (None, 46, 46, 32)        832       
    _________________________________________________________________
    max_pooling2d_1 (MaxPooling2 (None, 23, 23, 32)        0         
    _________________________________________________________________
    conv2d_2 (Conv2D)            (None, 19, 19, 64)        51264     
    _________________________________________________________________
    max_pooling2d_2 (MaxPooling2 (None, 4, 4, 64)          0         
    _________________________________________________________________
    flatten_1 (Flatten)          (None, 1024)              0         
    _________________________________________________________________
    dense_1 (Dense)              (None, 1024)              1049600   
    _________________________________________________________________
    dropout_1 (Dropout)          (None, 1024)              0         
    _________________________________________________________________
    dense_2 (Dense)              (None, 12)                12300     
    =================================================================
    Total params: 1,113,996
    Trainable params: 1,113,996
    Non-trainable params: 0
    _________________________________________________________________

### So this program will create handEmo.h5 file.

### Now we have our model file, lets implement it to classify realtime emojis...

## Application (application.py)

```python
import cv2
from keras.models import load_model
import numpy as np
import os
```

```python
model = load_model('handEmo.h5')
```

```python
def get_emojis():
    emojis_folder = 'hand_emo/'
    emojis = []
    for emoji in range(len(os.listdir(emojis_folder))):
        print(emoji)
        emojis.append(cv2.imread(emojis_folder+str(emoji)+'.png', -1))
    return emojis
```

```python
def keras_predict(model, image):
    processed = keras_process_image(image)
    pred_probab = model.predict(processed)[0]
    pred_class = list(pred_probab).index(max(pred_probab))
    return max(pred_probab), pred_class
```

```python
def keras_process_image(img):
    image_x = 50
    image_y = 50
    img = cv2.resize(img, (image_x, image_y))
    img = np.array(img, dtype=np.float32)
    img = np.reshape(img, (-1, image_x, image_y, 1))
    return img
```

```python
def overlay(image, emoji, x,y,w,h):
    emoji = cv2.resize(emoji, (w, h))
    try:
        image[y:y+h, x:x+w] = blend_transparent(image[y:y+h, x:x+w], emoji)
    except:
        pass
    return image
```

```python
def blend_transparent(face_img, overlay_t_img):
    # Split out the transparency mask from the colour info
    overlay_img = overlay_t_img[:,:,:3] # Grab the BRG planes
    overlay_mask = overlay_t_img[:,:,3:]  # And the alpha plane

    # Again calculate the inverse mask
    background_mask = 255 - overlay_mask

    # Turn the masks into three channel, so we can use them as weights
    overlay_mask = cv2.cvtColor(overlay_mask, cv2.COLOR_GRAY2BGR)
    background_mask = cv2.cvtColor(background_mask, cv2.COLOR_GRAY2BGR)

    # Create a masked out face image, and masked out overlay
    # We convert the images to floating point in range 0.0 - 1.0
    face_part = (face_img * (1 / 255.0)) * (background_mask * (1 / 255.0))
    overlay_part = (overlay_img * (1 / 255.0)) * (overlay_mask * (1 / 255.0))

    # And finally just add them together, and rescale it back to an 8bit integer image
    return np.uint8(cv2.addWeighted(face_part, 255.0, overlay_part, 255.0, 0.0))
```

```python
emojis = get_emojis()
cap = cv2.VideoCapture(0)
x, y, w, h = 300, 50, 350, 350
```

```python
while (cap.isOpened()):
    ret, img = cap.read()
    img = cv2.flip(img, 1)
    hsv = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)
    mask2 = cv2.inRange(hsv, np.array([2, 50, 60]), np.array([25, 150, 255]))
    res = cv2.bitwise_and(img, img, mask=mask2)
    gray = cv2.cvtColor(res, cv2.COLOR_BGR2GRAY)
    median = cv2.GaussianBlur(gray, (5, 5), 0)

    kernel_square = np.ones((5, 5), np.uint8)
    dilation = cv2.dilate(median, kernel_square, iterations=2)
    opening = cv2.morphologyEx(dilation, cv2.MORPH_CLOSE, kernel_square)
    ret, thresh = cv2.threshold(opening, 30, 255, cv2.THRESH_BINARY)

    thresh = thresh[y:y + h, x:x + w]
    contours = cv2.findContours(thresh.copy(), cv2.RETR_TREE, cv2.CHAIN_APPROX_NONE)[0]
    if len(contours) > 0:
        contour = max(contours, key=cv2.contourArea)
        if cv2.contourArea(contour) > 2500:
            x, y, w1, h1 = cv2.boundingRect(contour)
            newImage = thresh[y:y + h1, x:x + w1]
            newImage = cv2.resize(newImage, (50, 50))
            pred_probab, pred_class = keras_predict(model, newImage)
            print(pred_class, pred_probab)
            img = overlay(img, emojis[pred_class], 400, 250, 90, 90)

    x, y, w, h = 300, 50, 350, 350
    cv2.imshow("Frame", img)
    cv2.imshow("Contours", thresh)
    k = cv2.waitKey(10)
    if k == 27:
        break
```

## We are done here...

## On executing application.py file, this will open system's webcam and start capturing the hand gestures

### Like this...

<video width="640" height="360" controls="controls">
  <source src="/videos/emojinator.mp4" type="video/mp4">
</video>

## Complete Code [here](https://github.com/risx3/emojinator-repository)
# Extracting EKG waves using Neural Network
Segmenting cardiovascular signals using U-Net model with Keras backend. 

Technologies utilized: Python, Keras, OpenCV (for preprocessing)

{
 "cells": [
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Image Augmentation"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Image augmentation\n",
    "import cv2\n",
    "import random\n",
    "import os"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Visualization\n",
    "from matplotlib import pyplot as plt"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Training\n",
    "from keras.callbacks import ModelCheckpoint\n",
    "from keras.callbacks import CSVLogger\n",
    "from keras.callbacks import EarlyStopping\n",
    "from keras.optimizers import Adam\n",
    "\n",
    "from sklearn.utils import class_weight\n",
    "\n",
    "import numpy as np\n",
    "import model\n",
    "import time"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "SIZE = 512\n",
    "channel = 1"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Image augmentation\n",
    "def data_gen(img_folder, mask_folder, batch_size):\n",
    "    c = 0\n",
    "    n = os.listdir(img_folder) #List of training images\n",
    "    random.shuffle(n)\n",
    "\n",
    "    while (True):\n",
    "        img = np.zeros((batch_size, SIZE, SIZE, channel)).astype('float')\n",
    "        mask = np.zeros((batch_size, SIZE, SIZE, 1)).astype('float')\n",
    "\n",
    "        for i in range(c, c+batch_size): #initially from 0 to 16, c = 0. \n",
    "\n",
    "            # If training on RGB\n",
    "#             train_img = cv2.imread(img_folder+'/'+n[i])/255.       \n",
    "#             train_img = cv2.resize(train_img, (SIZE, SIZE))# Read an image from folder and resize\n",
    "            \n",
    "            # If training on grayscale images \n",
    "            train_img = cv2.imread(img_folder+'/'+n[i], cv2.IMREAD_GRAYSCALE)/255.\n",
    "            train_img = cv2.resize(train_img, (SIZE, SIZE))# Read an image from folder and resize\n",
    "            train_img = train_img.reshape(SIZE, SIZE, channel) # Add extra dimension for parity with train_img size [512 * 512 * 3]\n",
    "    \n",
    "            img[i-c] = train_img #add to array - img[0], img[1], and so on.\n",
    "\n",
    "            train_mask = cv2.imread(mask_folder+'/'+n[i], cv2.IMREAD_GRAYSCALE)/255.\n",
    "            train_mask = cv2.resize(train_mask, (SIZE, SIZE))\n",
    "            train_mask = train_mask.reshape(SIZE, SIZE, 1) # Add extra dimension for parity with train_img size [512 * 512 * 3]\n",
    "\n",
    "            mask[i-c] = train_mask\n",
    "\n",
    "        c+=batch_size\n",
    "        if(c+batch_size>=len(os.listdir(img_folder))):\n",
    "            c=0\n",
    "            random.shuffle(n)\n",
    "          \n",
    "        yield img, mask\n",
    "\n",
    "\n",
    "train_frame_path = 'train_frames/train'\n",
    "train_mask_path = 'train_masks/train'\n",
    "\n",
    "val_frame_path = 'val_frames/val'\n",
    "val_mask_path = 'val_masks/val'\n",
    "\n",
    "# Create generator objects\n",
    "train_gen = data_gen(train_frame_path,train_mask_path, batch_size = 4)\n",
    "val_gen = data_gen(val_frame_path,val_mask_path, batch_size = 4)\n",
    "\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Check to see if mask is saved correctly for training set\n",
    "image_batch, mask_batch = next(train_gen)\n",
    "\n",
    "r = random.randint(0, len(image_batch)-1)\n",
    "\n",
    "fig = plt.figure()\n",
    "fig.subplots_adjust(hspace=0.4, wspace=0.4)\n",
    "ax = fig.add_subplot(1, 2, 1)\n",
    "ax.imshow(np.reshape(image_batch[r], (SIZE, SIZE)))\n",
    "ax = fig.add_subplot(1, 2, 2)\n",
    "ax.imshow(np.reshape(mask_batch[r], (SIZE, SIZE)), cmap=\"gray\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Check to see if mask is saved correctly for validation set\n",
    "# image_batch, mask_batch = next(val_gen)\n",
    "\n",
    "# r = random.randint(0, len(image_batch)-1)\n",
    "\n",
    "# fig = plt.figure()\n",
    "# fig.subplots_adjust(hspace=0.4, wspace=0.4)\n",
    "# ax = fig.add_subplot(1, 2, 1)\n",
    "# ax.imshow(image_batch[r].reshape(SIZE, SIZE, 3))\n",
    "# ax = fig.add_subplot(1, 2, 2)\n",
    "# ax.imshow(np.reshape(mask_batch[r], (SIZE, SIZE)), cmap=\"gray\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Training"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "m = model.unet(input_size= (SIZE, SIZE, channel))"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "import loss"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "batch_size = 5\n",
    "NO_OF_TRAINING_IMAGES = len(os.listdir('train_frames/train/'))\n",
    "NO_OF_VAL_IMAGES = len(os.listdir('val_frames/val/'))\n",
    "\n",
    "NO_OF_EPOCHS = 50\n",
    "\n",
    "BATCH_SIZE = batch_size\n",
    "\n",
    "weights_path = 'weights/weights_00.h5'\n",
    "\n",
    "opt = Adam(lr=1E-5, beta_1=0.9, beta_2=0.999, epsilon=1e-08)\n",
    "\n",
    "m.compile(loss=loss.dice_coef_loss,\n",
    "          optimizer=opt,\n",
    "          metrics=['acc', 'mae'])\n",
    "\n",
    "checkpoint = ModelCheckpoint(filepath=weights_path, monitor='val_loss', \n",
    "                             verbose=1, save_best_only=True, save_weights_only=True)\n",
    "\n",
    "csv_logger = CSVLogger('./log.out', append=True, separator=';')\n",
    "\n",
    "earlystopping = EarlyStopping(monitor = 'val_loss', verbose = 1,\n",
    "                              min_delta = 1e-4, patience = 2, mode = 'auto')\n",
    "\n",
    "callbacks_list = [checkpoint, csv_logger, earlystopping]\n",
    "\n",
    "# class_weights = {0: 1.,\n",
    "#                  1: 197.}"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "start_time = time.time()\n",
    "results = m.fit_generator(train_gen, epochs=NO_OF_EPOCHS, \n",
    "                          steps_per_epoch = (NO_OF_TRAINING_IMAGES//BATCH_SIZE),\n",
    "                          validation_data=val_gen, \n",
    "                          validation_steps=(NO_OF_VAL_IMAGES//BATCH_SIZE), \n",
    "                          callbacks=callbacks_list)\n",
    "time_passed = time.time() - start_time\n",
    "# print('Ellapsed time: {}'.format(hms_string(time_passed))\n",
    "m.save('Model_00.h5')\n",
    "# print(str(time_passed))"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "print (\"It took {} hours to execute this\".format((time_passed) / 3600.))"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Evaluate model on test dataset"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Create testing dataset\n",
    "test_frame_path = 'test_frames'\n",
    "test_mask_path  = 'test_masks'\n",
    "\n",
    "test_gen = data_gen(test_frame_path, test_mask_path, batch_size = batch_size)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Loading test data\n",
    "image_batch, mask_batch = next(test_gen)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "m.metrics_names"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "m.evaluate(x = image_batch,\n",
    "               y = mask_batch,\n",
    "               batch_size = batch_size)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Making predictions with model"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "def predict_one_mask():\n",
    "    image_batch, mask_batch = next(test_gen)\n",
    "#     print(type(image_batch))\n",
    "#     print(type(mask_batch))\n",
    "    predicted_mask_batch = m.predict(image_batch)\n",
    "    image = image_batch[0]\n",
    "#     print(type(predicted_mask_batch))\n",
    "#     print(type(image))\n",
    "    print(image.shape)\n",
    "    predicted_mask = predicted_mask_batch[0].reshape(SIZE, SIZE)\n",
    "    plt.imshow(image.squeeze())\n",
    "    plt.imshow(predicted_mask, alpha=0.7)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "predict_one_mask()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "image_batch, mask_batch = next(test_gen)\n",
    "pred = m.predict(image_batch)\n",
    "pred = pred > 0.5"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "fig = plt.figure()\n",
    "fig.subplots_adjust(hspace=0.4, wspace=0.4)\n",
    "\n",
    "ax = fig.add_subplot(1, 3, 1)\n",
    "ax.imshow(np.reshape(image_batch[0]*255, (SIZE, SIZE)), cmap='gray')\n",
    "\n",
    "ax = fig.add_subplot(1, 3, 2)\n",
    "ax.imshow(np.reshape(mask_batch[0]*255, (SIZE, SIZE)), cmap='gray')\n",
    "\n",
    "ax = fig.add_subplot(1, 3, 3)\n",
    "ax.imshow(np.reshape(pred[0]*255, (SIZE, SIZE)), cmap=\"gray\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "fig = plt.figure()\n",
    "fig.subplots_adjust(hspace=0.4, wspace=0.4)\n",
    "\n",
    "ax = fig.add_subplot(1, 2, 1)\n",
    "ax.imshow(np.reshape(mask_batch[1]*255, (SIZE, SIZE)), cmap=\"gray\")\n",
    "\n",
    "ax = fig.add_subplot(1, 2, 2)\n",
    "ax.imshow(np.reshape(pred[1]*255, (SIZE, SIZE)), cmap=\"gray\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": []
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.7.3"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 2
}

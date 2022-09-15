---
layout: post
title: "google tensorflow notes"
keywords: ["ML","mechine learning","SVM","","Dtree","classification","tensorflow"]
description: "svm"
category: "Datascience"
tags: ["ML","Datascience"]
---
## google tensorflow的安装

可以参考文档[Google tensorflow Download and Setup](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/g3doc/get_started/os_setup.md#virtualenv-installation）,有很多中方式)

在自己的Mac上安装，这里选择了[Virtualenv installation](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/g3doc/get_started/os_setup.md#virtualenv-installation)，也可以选择Anaconda或者其他安装方式

Install pip and Virtualenv:

```
sudo  pip  install --upgrade virtualenv 
```

Create a Virtualenv environment in the directory ~/tensorflow:

```
virtualenv   --system-site-packages  ./tensorflow
```
Activate the environment and use pip to install TensorFlow inside it:

```
tensorflow/bin/activate
pip install --upgrade https://storage.googleapis.com/tensorflow/mac/tensorflow-0.7.1-cp27-none-any.whl
```
When you are done using TensorFlow, deactivate the environment.

```
(tensorflow)$ deactivate
```
To use TensorFlow later you will have to activate the Virtualenv environment again:

```
source  tensorflow/bin/activate
```
测试是否安装ok
 
```
tensorflow) bogon:develop bogon$ python
Python 2.7.5 (default, Aug 25 2013, 00:04:04) 
[GCC 4.2.1 Compatible Apple LLVM 5.0 (clang-500.0.68)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> hello = tf.constant('Hello, TensorFlow!')
>>> sess = tf.Session()
>>> print(sess.run(hello))
Hello, TensorFlow!
>>> a = tf.constant(10)
>>> b = tf.constant(32)
>>> print(sess.run(a + b))
42
>>> 
```
 
实例学习[TensorFlow-Examples](https://github.com/aymericdamien/TensorFlow-Examples)
这个感觉挺好的，有代码，notebook。相关算法有Linear Regression,Logistic Regression，Convolutional Neural Network，Recurrent Neural Network (LSTM) .etc
 
```
 
bogon$ git clone https://github.com/aymericdamien/TensorFlow-Examples.git
```
 
 运行一个CNN的例子
 
```
tensorflow) bogon:3 - Neural Networks bogon$ python convolutional_network.py 
Succesfully downloaded train-images-idx3-ubyte.gz 9912422 bytes.
Extracting /tmp/data/train-images-idx3-ubyte.gz
/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/gzip.py:268: VisibleDeprecationWarning: converting an array with ndim > 0 to an index will result in an error in the future
  chunk = self.extrabuf[offset: offset + size]
/Users/bogon/develop/tensorflow/TensorFlow-Examples/examples/3 - Neural Networks/input_data.py:35: VisibleDeprecationWarning: converting an array with ndim > 0 to an index will result in an error in the future
  data = data.reshape(num_images, rows, cols, 1)
Succesfully downloaded train-labels-idx1-ubyte.gz 28881 bytes.
Extracting /tmp/data/train-labels-idx1-ubyte.gz
Succesfully downloaded t10k-images-idx3-ubyte.gz 1648877 bytes.
Extracting /tmp/data/t10k-images-idx3-ubyte.gz
Succesfully downloaded t10k-labels-idx1-ubyte.gz 4542 bytes.
Extracting /tmp/data/t10k-labels-idx1-ubyte.gz
Iter 1280, Minibatch Loss= 30910.765625, Training Accuracy= 0.25781
Iter 2560, Minibatch Loss= 16850.156250, Training Accuracy= 0.39062
Iter 3840, Minibatch Loss= 18461.570312, Training Accuracy= 0.51562
Iter 5120, Minibatch Loss= 10053.062500, Training Accuracy= 0.67188
...
Iter 99840, Minibatch Loss= 738.179016, Training Accuracy= 0.91406
Optimization Finished!
Testing Accuracy: 0.941406
```


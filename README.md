# PEGASOS-SVM-CLASSIFIER
Implementation of a support vector machine classifier using primal estimated sub-gradient solver in C++

## About
This project was about building a Suport Vector Machine binary classifier from scratch using the methods described in [this paper](https://www.cs.huji.ac.il/~shais/papers/ShalevSiSrCo10.pdf), and tweaking it to a mini batch version that, in theory, would benefit from the massive parallelizing available for GPU devices.

Two main SVM dinary classifiers were then built for comparison purposes, a sequential based pegasos basic algorithm described in section 2.1 of the article, and a mini-batch version described in section 2.3. The sequential version trains with data on CPU, while the mini-batch version trains with data natively in GPU using CUDA kernels. There are also two other experimental versions using CUDA THRUST API on ohter_versions folder, though be aware that they are not totally completed/optimized, and therefore wont be covered in the scope of this document.

The goal of a Support Vector Machine is to find the best separating hyperplane for two types of samples (the ones labeled as “negative” and the ones labeled as “positive”). A good separating hyperplane is one that can divide two kinds of sample while maximizing its distance from the two sets of sample. For that to happen, at each epoch, the algorithm must take a sample, or a batch of samples, compute if the current inclination + offset (also known as weight + bias) in that epoch of that hyperplane in the dimension of that sample(s) feature correctly predicts its label, and adjusting its weight values according to the result.

If you would like to know more about Support vector machines and how they work, i suggest looking into this very didatic video from MIT OpenCourseWare [here](https://www.youtube.com/watch?v=_PwhiWxHK8o&t=1324s). If you want details into building the necessary environment to run this code and how to operate it, continue reading this guide, or if you are more interested on data analysis and the results i've got, skip to the Results section.

## Requirements
* have a NVIDIA GPU (this is a must, if you want to use the mini-batch version);
* g++ compiler;
* Install Boost library;
```
$ sudo apt-get install libboost-all-dev
```
* Install CUDA;
* Have nvcc compilation tool that supports your CUDA version;
* If you dont want to use the provided datasets on this repo, download it by yourself and make sure its in csv file format (doesnt doesnt have to be in .csv extension, just its comma separated value data format). Also, make sure that your sample class is represented by a number and its located in the last value on the line of the respective sample;

## How to use:
Inside each version folder there's a makefile to compile all the filed needed. If you are all set with the requirements, typing the below command shouldn't present any problems.
```
$ make
```
Once compiled, the program is ready to be runned (./main). Before you do it though, if you want to use any of the datasets provided, just unzip the dataset folder to the same place the zipped version was. If you do this and dont set any of the environment variables presented below, it will run the diabetes.csv dataset with the default parameter values (also presented below). If you dont unzip the dataset, and dont use the data_path variable, you'll get yourself a seg fault error! :(

You can use the following environment variables when running the code:
* DATA_PATH - Path to your dataset (Default: ../datasets/diabetes.csv);
* C - Lambda regularization parameter to contrrol step size (Default: 0.001);
* SAMPLES_LIMIT - If you dont want to use your whole dataset for speed purposes, specify the maximum samples you'd like to read (Default: 9999999);
* EPOCHS - Number of epochs the training part will run;
* BATCH_SIZE - The desired size of your batch (ONLY AVAILABLE IN MINI BATCH ALGORITHM) (Default: 10);
* TRAIN_SIZE - Percentage of the dataset allocated for training (0-1) (Default: 0.8);
* NUM_ITERATIONS - Number of times the algorithm will run, good to mitigate some weird results due to bad seed, and return mean accuracy of all iterations (Default: 10);
* POSITIVE_CLASS - Class whose samples are gonna be labeled as positive, while the rest of the samples will be labeled negative (Default: 1).

Example:
```
$  DATA_PATH=../datasets/mnist_train.csv C=0.0001 EPOCHS=500000 BATCH_SIZE=200 TRAIN_SIZE=0.8 NUM_ITERATIONS=20 ./main
```


## Results:

The following tests were run on this configuration:
* Core i7-7700 @3.6 GHz;
* GTX 1060 6GB;

### 1. Performance:
The premise of building a mini-batch gpu version was that it would have a better performance as it would be able to fetch more samples at once while also benefitting from datasets that have a large number of features (e.g, MNIST). As numbers of samples/time go, this has proven to be the case:

The four graphics below represent time per number of samples of each database i used:

![Alt text](imgs/charts.png?raw=true "Title")


On the CPU sequential version (marked in red), since there is not the batch concept, the number of samples directly reflects the number of epochs the alogrithm used, limited to one single sample used per epoch. On the GPU mini-batch version (marked in blue) the number of samples are calculated using always a fixed epoch number (10000) multiplied by the number of samples contained in a batch, and increascing only the batch number at each data point.

As the charts represent, training with a small number of used samples yields a better performance on the CPU side, since the data doesnt have to be passed on to the gpu. However, as the number of samples used increase, the gpu starts compensating its slow data copy with the ability to iterate each weight at the same time, and process the whole batch in parallel as well. MNIST is clearly one that takes advantage of Mini-batch processing, given its large feature set (one per pixel in a 28x28 image).

OBS1: Keep in mind that the parallel potential of cuda is limited by your Graphics Card specs. The number of threads-per-block i used might not work on another, older GPU, and the same goes with threads-per-grid. The reasion is that there might not have the amount needed to run big batches as i do on the above tests.

OBS2: Time accounts both fit and predict steps.

### 2. Accuracy:

The main point of a classifier is to have a solid accuracy over a classification test. The tests below tries to get to approximate accuracy results compared to scikit python SVM version. Then it compares both the performance of the cpu and gpu accuracies against the time they took to get to the approximate results of scikits tool. There were no scientific method to determine the regularization parameters, batch size or epochs in any of these datasets, other than reading some other solutions online, performing some accuacy/epochs tests and seeing what works best. Be aware that you might get similar results with different parameters.

The class number beside every dataset name has to do with the class number I chose to be the positive sample, and all others to be negative ones. This has to be done once we are talking about a binary classifier, and therefore can't handle multiple classes.

OBS1: Time now accounts only fit step.

#### Iris class 2 dataset:
Percentage of class 2 in dataset: 0.33

Scikit SVM for python:
* accuracy = 0.966667;

Sequential version:
`C=0.0001 EPOCHS=10000 TRAIN_SIZE=0.8 NUM_ITERATIONS=100`
* mean accuracy = 0.968333;
* time/iteration = 0.0036 seconds;

GPU version:
`C=0.0001 EPOCHS=1000 BATCH_SIZE=100 TRAIN_SIZE=0.8 NUM_ITERATIONS=100`
* mean accuracy = 0.975;
* time/iteration = 0.021 seconds;

It turns out that iris dataset converges to a solution very quickly, and since it doesn't have many features and can't have batches greater than 100 (due to training size and dataset size) it won't benefit from massive parallelizing and it's cons.

#### Diabetes class 1 Dataset:
Percentage of class 1 in dataset: 0.348958

Scikit SVM for python:
* accuracy = 0.679674;

Sequential version:
`C=0.001 EPOCHS=10000000 TRAIN_SIZE=0.8 NUM_ITERATIONS=20`
* mean accuracy = 0.665752;
* time/iteration = 4.37 seconds;

GPU version:
`C=0.001 EPOCHS=1000000 BATCH_SIZE=200 TRAIN_SIZE=0.8 NUM_ITERATIONS=20 ./main`
* mean accuracy = 0.657358;
* time/iteration = 37.06 seconds;

Diabetes dataset is another one that won't benefit from GPU perks for the exact same reasons stated for Iris dataset.

#### Covtype class 1 Dataset:
Percentage of class 1 in dataset: 0.364605

Scikit SVM for python:
* accuracy = 0.753333;

Sequential version:
`C=0.0001 EPOCHS=10000000 TRAIN_SIZE=0.8 NUM_ITERATIONS=20`
* mean accuracy = 0.679460;
* time/iteration = 13.59 seconds;

GPU version:
`C=0.0001 EPOCHS=500000 BATCH_SIZE=400 TRAIN_SIZE=0.8 NUM_ITERATIONS=20`
* mean accuracy = 0.667627;
* time/iteration = 39.42 seconds;

This dataset starts to be more interesting to the gpu because it has lots of samples and does not limit how much of them can be inserted in a batch. for that reason, and the fact that is has 50 features, considerably more than both datasets before, it performs good in gpu. However its still not enough to beat cpu on time ellapsed. Both algorithms had a hard time getting closer to scikits results in a suitable time for me to write this report, and for time limiting reasons, i decided not to push that far, but it could converge to scikits results with enough time and better parameters.


#### Mnist class 5 Dataset:
Percentage of class 5 in dataset: 0.09035

Scikit SVM for python:
* accuracy = 0.970166;

Sequential version:
`C=0.0001 EPOCHS=100000 TRAIN_SIZE=0.8 NUM_ITERATIONS=100`
* mean accuracy = 0.9561;
* time/iteration = 1.16 seconds;

GPU version:
`C=0.0001 EPOCHS=100000 BATCH_SIZE=100 TRAIN_SIZE=0.8 NUM_ITERATIONS=100`
* mean accuracy = 0.953983;
* time/iteration = 1.7414 seconds;

The last result, mnist dataset, was another one that converged pretty quickly to an approximate result. Even though it still took more time to run on the gpu, its possible to see the pattern that the more features it has, the time difference between both version gets shorter, up to a point where datasets with thousand of features will run better on the mini-batch gpu version.

### 3. Cuda implementation versus paper result:

Finally, its important to check if my implementation matched accruracy-wise the results presented by the paper i based on my implementation (the one cited in the beginning nof this document). There are two datasets i've used that the paper also uses (mnist and covtype), and this will be the ones i'll be comparing. The results below are the best results i've got using the same lambda (c) parameter and big batches and epochs.

#### Mnist class 8 Dataset:

Paper results:
`C=0.0000167`
* accuracy = 0.94;

GPU version:
`POSITIVE_CLASS=8 C=0.0000167 EPOCHS=1000000 BATCH_SIZE=70 TRAIN_SIZE=0.8 NUM_ITERATIONS=5`
* best accuracy = 0.943583;
* time/iteration = 125.03 seconds;

#### Covtype class 1 Dataset:

Paper results:
`C=0.000001`
* accuracy = 0.768;

GPU version:
`POSITIVE_CLASS=1 C=0.000001 EPOCHS=1000000 BATCH_SIZE=100 TRAIN_SIZE=0.8 NUM_ITERATIONS=5`
* best accuracy = 0.733507;
* time/iteration = 24 seconds;

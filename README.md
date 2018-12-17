# Educational Data Mining EEG Confusion Detection
This is a project for the Helsinki University course on Educational Data Mining. 

## Our goal
We follow the example of the paper [Confused or not confused?](https://dl.acm.org/citation.cfm?id=3107513) by Ni et al and try to create a recurrent neural network (RNN) model capable of inferring students' confusion i.e. brain fog during watching online lecture videos. 

The authors show in the paper that data obtained from a single-channel EEG headset is enough to classify binary confusion with reasonable accuracy. Our first aim is to replicate the work done in the paper, since the [data](#data) is openly available.

Furthermore, we test the effect of adding additional information of the videos themselves. We extract information of the video by adding raw image data and sentence vectors produced from subtitle captions.

## Data
The data we use is hosted on the website [Kaggle](https://www.kaggle.com/wanghaohan/confused-eeg/home) and it is provided by the authors of [Using EEG to Improve Massive Open Online Courses Feedback Interaction](http://www.cs.cmu.edu/~kkchang/paper/WangEtAl.2013.AIED.EEG-MOOC.pdf). 10 subjects, wearing Mindset EEG headphones, were shown 20 lecture videos of content classified as 'easy' or 'difficult.' After each video, subjects were asked if they found the video confusing or not. These responses were then encoded as a boolean value and added to the dataset. The collected EEG data was then averaged over 0.5 second intervals. The hosted data included results from only 10 of these videos (5 classified as 'easy' and 5 as 'difficult.') 

The source videos were also included in the dataset. Each video was uploaded to YouTube, and the automatic subtitles generated by YouTube were then downloaded. These subtitles were converted into 1024-dimensional vectors using ELMo word embeddings. More information regarding ELMo can be found [here](https://allennlp.org/elmo) and [here](https://github.com/allenai/allennlp/blob/master/tutorials/how_to/elmo.md). Finally, the videos were sliced into images, one every 0.5 seconds. This data was then loaded as grayscale into a numpy array.
 
## Classifying confusion
We run multiple models readily implemented for use in Python's Scikit-Learn library in order to have a good baseline for our
LSTM model. We compute accuracy as our primary metric, since it is the only one used in the papers we base our study on.
Additionally we provide [F1-score](https://en.wikipedia.org/wiki/F1_score) and [ROC-AUC](https://en.wikipedia.org/wiki/Receiver_operating_characteristic) to gain more insight on the performance of the models.

Models used are as follows:
  * (Naive) Predicts all labels as zero regardless of input
  * (Logreg) Logistic Regression 
  * (Ptron) Perceptron
  * (KNN) K-nearest neighbours
  * (GNB) Naive Bayes with Gaussian priors
  * (BNB) Naive Bayes with Bernoulli priors
  * (RF) Random Forest
  * (GBT) Gradient Boosted Decision Trees
  * (MLP2) Multilayer Perceptron with 2 hidden layers
  * (MLP3) Multilayer Perceptron with 3 hidden layers
  * (SVC Linear) Support Vector Machine with linear kernel
  
The models are all tested by 5-fold cross-validation. We didn't however finetune parameters for the various models.

#### Results for cross-validating models:
<img src="https://github.com/taikamurmeli/edm_eeg_confusion_detection/blob/master/plots_and_images/plot_original_data.png" height="250"/>

From the plot we can see that the best models here (GBT, SVC) achieve near 70% accuracy.
When compared to Zhuoheng et al's reported results (picture below), the SVC performs similarly but interestingly our KNN performs clearly better than the one in the paper. The paper doesn't report KNN configuration, but their [code](https://github.com/nateanl/EEG_Classification/blob/master/KNN.py) reveals that their input shape for KNN is (12811, 14) and output shape is (12811,). This means that the KNN is attempting to classify confusion for a whole video by just one interval. With proper input, i.e. 100 data points with 112 concatenated feature vectors resulting in shape (100, 112*12=1344), the KNN achieves accuracy of 70.9%. The small difference to our KNN result (68% accuracy) is likely due to differences in the implementation of cross-validation. We use our own cross-validation whereas the other code uses sklearn's ''[cross_val_score](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.cross_val_score.html)'' 
.

#### Results from the paper "Confused or not confused?":
<img src="https://github.com/taikamurmeli/edm_eeg_confusion_detection/blob/master/plots_and_images/plot_confused_paper_results.png" height="250"/>

### Attempting to reproduce "Confused or not confused?" -paper's results 
Our starting point was to create the model described in Ni et al's paper and try to improve on that. The model consists of  batch normalization layer, a 50 neuron Bidirectional Long-Short Term Memory (LSTM) Recurrent layer as described in the paper, and surprisingly a 112 neuron output layer. We first assumed that the model was supposed to predict 1 output for each subject-video pair that were our data points. However, we had difficulties producing stable results, such as described in the paper. Our similar LSTM model's accuracy varied from 45% to 80% and even the cross-validation accuracy varied slightly while remaining around 60%. This indicates a high dependence for random initialization of the model and that a regular LSTM probably isn't the best choice for this task.

Luckily we have the [original LSTM model's code](https://github.com/nateanl/EEG_Classification/blob/master/EEG_LSTM.py) and we tested that also. First we simply ran it and got lower results than those in the paper. Then for proper validation, we removed the random seeds from the code and put the whole [cross-validation in a for-loop](https://github.com/taikamurmeli/edm_eeg_confusion_detection/blob/master/test_eeg_classification_lstm.ipynb). The accuracy was 61.3% , which is much closer to our attempts than what is in the paper. Thus, if there is no magic performed that has been omitted from the published code, it seems that the paper's results are not reproducible.
## Leveraging subtitle information
We decided to test the effect of adding subtitles to test if there was some semantic information in the videos that might help classifying confusion. For this we used YouTube's automatic captioning to get the subtitles as captions. We then mapped the captions to the videos' middle minute intervals so that it is in line with the kaggle EEG data. After that, we used pre-trained [ELMo word embedding model](https://github.com/HIT-SCIR/ELMoForManyLangs/) to convert the subtitles to word vectors and created subtitle vectors by averaging the genereted ELMo word vectors. Finally we added the subtitle vectors to the dataset and tested the models with the combined set.    

#### Results for adding subtitle vecs
<img src="https://github.com/taikamurmeli/edm_eeg_confusion_detection/blob/master/plots_and_images/plot_subvecs.png" height="250"/>

It seems that the subtitle vectors didn't help at all in explaining the students' perceived confusion. For some reason perceptron gets good results, but this might be due to chance as all other models don't seem affected. The reason for GBT's decreased performance is simply reducing the number of trees drastically from 777 to 3. Thus GBT modification was necessary since training it with the vectors was extremely slow. 

The LSTM model is not shown on the graph, but we simply note that adding subtitle vectors to its inputs did not have a consistent effect on the model's performance.

Additionally, we tried using PCA to reduce subtitle vector dimensions, since it [can be used to improve word vectors](https://ieeexplore.ieee.org/abstract/document/8500303).
#### Results for subtitle vecs' 12 principal components
<img src="https://github.com/taikamurmeli/edm_eeg_confusion_detection/blob/master/plots_and_images/plot_pca_sub_vecs.png" height="250"/>

As can be seen, the subtitle vectors with reduced dimensionality don't seem to affect the models and now the GBT performs normally with its 777 estimator trees. And the LSTM yet again was unaffected. We also tried several dimension values for PCA, but the effect was minimal.

## Leveraging image data
Each video was sliced into frames for 0.5 second intervals (using the 1st and 15th frame of each second). These were then loaded into a numpy array as grayscale images. This array was then flattened and merged with the original EEG data. The same methods used in the subtitle vectors benchmark were used to train a variety of classifiers. <img src="https://github.com/taikamurmeli/edm_eeg_confusion_detection/blob/master/plots_and_images/plot_image_classification_rescale.png" height="250"/>

Using image data to classify predefined difficulty seems to pick up on the fact that the videos of different difficulties also have distinct visual characteristics. The videos defined as 'easy' are almost all videos from Khan Academy. The 'difficult' videos typically feature a physical lecturer in front of a blackboard. The initial intention was for the model to learn interesting features from the video data using convolutional neural networks, however given the small size of the dataset and the clear visual distinctions between the two classes, it's likely to only learn that many grayscale values close to 0 (black) indicate 'easy' videos. It is likely that this explains the relatively good accuracy obtained with the baseline methods. Using a visually diverse array of videos would most likely reduce the performance of these baseline methods.

## Classifying predefined difficulty
We stumbled across [Ali Mehmani's repository] where he had used a Neural Network that combined a 2-dimensional convolution layer and two LSTM layers. Looking at his code we found that his model was predicting the pre-defined labels of the videos. Also, in comparison to Ni et al, Mehmani used pre-normalized data instead of batch normalization for the neural net.

We tested the model ourselves and truly, the model performed well on the pre-defined labels, however, not so much on the student-defined In addition, the model worked well only with zero padded data and not when the data was truncated to the minimum amount of intervals in the watched video data. In any case, the model's capability to infer predefined difficulty is rather interesting.

#### Mehmani model performance
<img src="https://github.com/taikamurmeli/edm_eeg_confusion_detection/blob/master/plots_and_images/mehmani_results.png" height="300"/> 
Mehmani used 10-fold cross-validation, but we got the similar results for our 5-fold cross-validation, accuracy: 0.780, F1: 0.828, and ROC-AUC: 0.809 

#### Baseline model performances for pre-defined difficulty truncated data
<img src="https://github.com/taikamurmeli/edm_eeg_confusion_detection/blob/master/plots_and_images/plot_predefined_labels.png" height="250"/>

Results for Mehmani's model with truncated data for pre-defined labels: accuracy 0.550, F1: 0.536, and ROC-AUC: 0.558

#### Baseline model performances for pre-defined difficulty zero padded data
<img src="https://github.com/taikamurmeli/edm_eeg_confusion_detection/blob/master/plots_and_images/plot_zero_pad_baseline_predefined.png" height="250"/>

The accuracy line here lies behind the ROC-AUC score and we can see that the Mehmani's model performance seems similar to decision tree based classifiers when predicting the pre-defined difficulty. Interestingly the way we even out the data has a huge efect on model performance as can be seen from comparing the plots with truncated and zero padded data. For predicting student-defined labels the effect was the opposite for all models.  

### Subtitle Vectors
Using the baseline models as a comparison, we found a couple of models worked well in utilizing the information encoded within the subtitle vectors. This indicates that the averaged ELMo embeddings carry enough semantic meaning to distuingish the difficult videos from non-difficult videos in this dataset.

#### Model performances for pre-defined difficulty with subtitle vectors
<img src="https://github.com/taikamurmeli/edm_eeg_confusion_detection/blob/master/plots_and_images/plot_subvecs_for_predefined_labels.png" height="250"/>
Mehmani's model achieved 1.0 for all scores regardless of how the intervals were evened out in the data.

## Conclusion and Proposed Future Work
While we were unable to properly replicate the paper's findings, other papers on the subject suggest that it is indeed possible to infer confusion using some form of EEG or EEG-derived data. 

This project and its related paper suffered from a small dataset and sample size. Additionally, the data that has been collected is quite noisy, using a less accurate 'Mindset' headset to collect electrical data from the brain. Using an actual EEG device and a standard 10-20 system would have yieled a better dataset. Additionally, increasing the number of videos from 10 to 20-30, and the number of participants from 10 to 30-50, would also be a worthwhile modification for future work. While it would increase the length of the study, granting students a user interface to highlight sections of video that induced confusion (if any) after watching each video would provide an even more compelling dataset. Identifying the approximate time at which a student became confused and attempting to infer changes in measured ERPs would be a worthwhile research project.


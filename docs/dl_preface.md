# (PART) Deep Learning Methods {-}

# Foreword {#dlforeword .unnumbered}

In Chapters \@ref(mlregression) and \@ref(mlclassification), we use algorithms such as regularized linear models, support vector machines, and naive Bayes models to predict outcomes from predictors including text data. Deep learning models approach the same tasks and have the same goals, but the algorithms involved are different. Deep learning models are "deep" in the sense that they use multiple layers to learn how to map from input features to output outcomes; this is in contrast to the kinds of models we used in the previous two chapters which use a shallow (single) mapping. 

<div class="rmdnote">
<p>Deep learning models can be effective for text prediction problems because they use these multiple layers to capture complex relationships in language.</p>
</div>

The layers in a deep learning model are connected in a network and these models are called **neural networks**, although they do not work much like a human brain. The layers can be connected in different configurations called network architectures. Two of the most common architectures used for text data are a convolutional neural network (CNN), and long short-term memory (LSTM)^[In other situations you may do best using a different architecture, for example, when working with dense, tabular data.]. These architectures sometimes incorporate word embeddings, as described in Chapter \@ref(embeddings).

For the following chapters, we will use tidymodels packages along with [Tensorflow](https://www.tensorflow.org/) and the R interface to Keras [@R-keras] for preprocessing, modeling, and evaluation. Table \@ref(tab:comparedlml) presents some key differences between deep learning and what, in this book, we call machine learning methods.

<table>
<caption>(\#tab:comparedlml)Comparing deep learning with other machine learning methods</caption>
 <thead>
  <tr>
   <th style="text-align:left;font-weight: bold;"> Machine learning </th>
   <th style="text-align:left;font-weight: bold;"> Deep learning </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;width: 55mm; "> Faster to train </td>
   <td style="text-align:left;width: 55mm; "> Takes more time to train </td>
  </tr>
  <tr>
   <td style="text-align:left;width: 55mm; "> Software is typically easier to install </td>
   <td style="text-align:left;width: 55mm; "> Software can be more challenging to install </td>
  </tr>
  <tr>
   <td style="text-align:left;width: 55mm; "> Can achieve good performance with less data </td>
   <td style="text-align:left;width: 55mm; "> Requires more data for good performance </td>
  </tr>
  <tr>
   <td style="text-align:left;width: 55mm; "> Depends on preprocessing to model more than very simple relationships </td>
   <td style="text-align:left;width: 55mm; "> Can model highly complex relationships </td>
  </tr>
</tbody>
</table>

Deep learning and more traditional machine learning algorithms are different, but the structure of the modeling process is largely the same, no matter what the specific details of prediction or algorithm are.

## Spending your data budget {-}

A limited amount of data is available for any given modeling project, and this data must be allocated to different tasks to balance competing priorities. We espouse an approach of first splitting data in testing and training sets, holding the testing set back until all modeling tasks are completed, including feature engineering and tuning. This testing set is then used as a final check on model performance, to estimate how the final model will perform on new data. 

The training data is available for tasks from model parameter estimation to determining which features are important and more. To compare or tune model options or parameters, this training set can be further split so that models can be evaluated on a validation set, or it can be resampled as described in Section \@ref(firstregressionevaluation) to create new simulated data sets for the purpose of evaluation. 

## Feature engineering {-}

Text data requires extensive processing to be appropriate for modeling, whether via an algorithm like regularized regression or a neural network. Chapters \@ref(language) through \@ref(embeddings) covered several of the most important techniques that are used to transform language into a representation appropriate for computation. This feature engineering part of the modeling process can be intensive for text, sometimes more computationally expensive than the fitting a model algorithm. 

We espouse an approach of implementing feature engineering on training data only, typically using resampled data sets, to avoid obtaining an overly optimistic estimate of model performance. Feature engineering can sometimes be a part of the machine learning process where subtle _data leakage_ occurs, when practitioners use information (say, to preprocess data or engineer features) that will not be available at prediction time. One example of this is tf-idf, which we introduced in Chapter \@ref(embeddings) and used in both Chapters \@ref(mlregression) and \@ref(mlclassification). As a reminder, the term frequency of a word is how frequently a word occurs in a document, and the inverse document frequency of a word is a weight, typically defined as:

$$idf(\text{term}) = \ln{\left(\frac{n_{\text{documents}}}{n_{\text{documents containing term}}}\right)}$$

These two quantities are multiplied together to compute a term's tf-idf, a statistic that measures the frequency of a term adjusted for how rarely it is used. Computing inverse document frequency involves the whole corpus or collection of documents. When you are fitting a model, how should that corpus be defined? We strongly advise that you should use your _training data only_ in such a situation. Using all the data available to you (training plus testing) involves leakage of information from the testing set to the model, and any estimates you may make of how your model may perform on new data may be overly optimistic.

<div class="rmdwarning">
<p>This specific example focused on tf-idf, but data leakage is a serious challenge in general for building machine learning models and systems. The tidymodels framework is designed to encourage good statistical practice, such as learning feature engineering transformations from training data and then applying those transformation to othe data sets.</p>
</div>


## Fitting and tuning {-}

Many different kinds of models are appropriate for text data, from more straightforward models like the linear models explored deeply in Chapter \@ref(mlregression) to the neural network models we cover in Chapters  \@ref(dlcnn) and \@ref(dllstm). Some of these models have hyperparameters which cannot be learned from data during fitting, like the regularization parameter of the models in Chapter \@ref(mlregression); these hyperparameters can be tuned using resampled data sets.

## Model evaluation {-}

Once models are trained and perhaps tuned, we can evaluate their performance quantitatively using metrics appropriate for the kind of practical problem being dealt with. Model explanation analysis, such as feature importance, also helps us understand how well and why models are behaving the way they do.

## Putting the model process in context {-}

This outline of the model process depends on us as data practitioners coming prepared for modeling with a healthy understanding of our data sets from exploratory data analysis. @Silge2017 provide a guide for exploratory data analysis for text. 

Also, in practice, the structure of a real modeling project is iterative. After fitting and tuning a first model or set of a models, a practitioner will often return to build more or better features, then refit models, and evaluate in a more detailed way. Notice that we take this approach in each chapter, both for more straightforward machine learning and deep learning; we start with a simpler model and then go back again and again to improve it in several ways. This iterative approach is healthy and appropriate, as long as good practices in data "spending" are observed. The testing set cannot be used during this iterative back-and-forth, and using resampled data sets can set us as practitioners up for more accurate estimates of performance.

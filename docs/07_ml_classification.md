# Classification {#mlclassification}




In Chapter \@ref(mlregression), we focused on modeling to predict *continuous values* for documents, such as what year a Supreme Court opinion was published. This is an example of a regression model. We can also use machine learning to predict *labels* on documents using a classification model. For both types of prediction questions, we develop a learner or model to describe the relationship between a target or outcome variable and our input features; what is different about a classification model is the nature of that outcome. 

- A **regression model** predicts a numeric or continuous value.
- A **classification model** predicts a class label or group membership.

For our classification example in this chapter, let's consider the data set of consumer complaints submitted to the US Consumer Finance Protection Bureau. Let's read in the complaint data (Section \@ref(cfpb-complaints)) with `read_csv()`.


```r
library(tidyverse)
complaints <- read_csv("data/complaints.csv.gz")
```

We can start by taking a quick `glimpse()` at the data to see what we have to work with. This data set contains a text field with the complaint, along with information regarding what it was for,
how and when it was filed, and the response from the bureau. 


```r
glimpse(complaints)
```

```
#> Rows: 117,214
#> Columns: 18
#> $ date_received                <date> 2019-09-24, 2019-10-25, 2019-11-08, 2019…
#> $ product                      <chr> "Debt collection", "Credit reporting, cre…
#> $ sub_product                  <chr> "I do not know", "Credit reporting", "I d…
#> $ issue                        <chr> "Attempts to collect debt not owed", "Inc…
#> $ sub_issue                    <chr> "Debt is not yours", "Information belongs…
#> $ consumer_complaint_narrative <chr> "transworld systems inc. \nis trying to c…
#> $ company_public_response      <chr> NA, "Company has responded to the consume…
#> $ company                      <chr> "TRANSWORLD SYSTEMS INC", "TRANSUNION INT…
#> $ state                        <chr> "FL", "CA", "NC", "RI", "FL", "TX", "SC",…
#> $ zip_code                     <chr> "335XX", "937XX", "275XX", "029XX", "333X…
#> $ tags                         <chr> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, N…
#> $ consumer_consent_provided    <chr> "Consent provided", "Consent provided", "…
#> $ submitted_via                <chr> "Web", "Web", "Web", "Web", "Web", "Web",…
#> $ date_sent_to_company         <date> 2019-09-24, 2019-10-25, 2019-11-08, 2019…
#> $ company_response_to_consumer <chr> "Closed with explanation", "Closed with e…
#> $ timely_response              <chr> "Yes", "Yes", "Yes", "Yes", "Yes", "Yes",…
#> $ consumer_disputed            <chr> "N/A", "N/A", "N/A", "N/A", "N/A", "N/A",…
#> $ complaint_id                 <dbl> 3384392, 3417821, 3433198, 3366475, 33853…
```

In this chapter, we will build classification models to predict what type of financial `product` the complaints are referring to, i.e., a label or categorical variable. The goal of predictive modeling with text input features and a categorical outcome is to learn and model the relationship between those input features, typically created through steps as outlined in Chapters \@ref(language) through \@ref(embeddings), and the class label or categorical outcome. Most classification models do predict the probability of a class (a numeric output), but the particular characteristics of this output make classification models different enough from regression models that we handle them differently.

## A first classification model {#classfirstattemptlookatdata}

For our first model, let's build a binary classification model to predict whether a submitted complaint is about "Credit reporting, credit repair services, or other personal consumer reports" or not. 

<div class="rmdnote">
<p>This kind of “yes or no” binary classification model is both common and useful in real-world text machine learning problems.</p>
</div>

The outcome variable `product` contains more categories than this, so we need to transform this variable to only contains the values "Credit reporting, credit repair services, or other personal consumer reports" and "Other".

It is always a good idea to look at your data! Here are the first six complaints:


```r
head(complaints$consumer_complaint_narrative)
```

```
#> [1] "transworld systems inc. \nis trying to collect a debt that is not mine,
not owed and is inaccurate."
#> [2] "I would like to request the suppression of the following items from my
credit report, which are the result of my falling victim to identity theft.
This information does not relate to [ transactions that I have made/accounts
that I have opened ], as the attached supporting documentation can attest. As
such, it should be blocked from appearing on my credit report pursuant to
section 605B of the Fair Credit Reporting Act."
#> [3] "Over the past 2 weeks, I have been receiving excessive amounts of
telephone calls from the company listed in this complaint. The calls occur
between XXXX XXXX and XXXX XXXX to my cell and at my job. The company does not
have the right to harass me at work and I want this to stop. It is extremely
distracting to be told 5 times a day that I have a call from this collection
agency while at work."
#> [4] "I was sold access to an event digitally, of which I have all the
screenshots to detail the transactions, transferred the money and was provided
with only a fake of a ticket. I have reported this to paypal and it was for the
amount of {$21.00} including a {$1.00} fee from paypal. \n\nThis occured on
XX/XX/2019, by paypal user who gave two accounts : 1 ) XXXX 2 ) XXXX XXXX"
#> [5] "While checking my credit report I noticed three collections by a
company called ARS that i was unfamiliar with. I disputed these collections
with XXXX, and XXXX and they both replied that they contacted the creditor and
the creditor verified the debt so I asked for proof which both bureaus replied
that they are not required to prove anything. I then mailed a certified letter
to ARS requesting proof of the debts n the form of an original aggrement, or a
proof of a right to the debt, or even so much as the process as to how the bill
was calculated, to which I was simply replied a letter for each collection
claim that listed my name an account number and an amount with no other
information to verify the debts after I sent a clear notice to provide me
evidence. Afterwards I recontacted both XXXX, and XXXX, to redispute on the
premise that it is not my debt if evidence can not be drawn up, I feel as if I
am being personally victimized by ARS on my credit report for debts that are
not owed to them or any party for that matter, and I feel discouraged that the
credit bureaus who control many aspects of my personal finances are so
negligent about my information."
#> [6] "I would like the credit bureau to correct my XXXX XXXX XXXX XXXX
balance. My correct balance is XXXX"
```

The complaint narratives contain many series of capital `"X"`'s. These strings (like "XX/XX" or "XXXX XXXX XXXX XXXX") are used to to protect personally identifiable information (PII) in this publicly available data set. This is not a universal censoring mechanism; censoring and PII protection will vary from source to source. Hopefully you will be able to find information on PII censoring in a data dictionary, but you should always look at the data yourself to verify. 

We also see that monetary amounts are surrounded by curly brackets (like `"{$21.00}"`); this is another text preprocessing step that has been taken care of for us. We could craft a regular expression to extract all the dollar amounts. 


```r
complaints$consumer_complaint_narrative %>%
  str_extract_all("\\{\\$[0-9\\.]*\\}") %>%
  compact() %>%
  head()
```

```
#> [[1]]
#> [1] "{$21.00}" "{$1.00}" 
#> 
#> [[2]]
#> [1] "{$2300.00}"
#> 
#> [[3]]
#> [1] "{$200.00}"  "{$5000.00}" "{$5000.00}" "{$770.00}"  "{$800.00}" 
#> [6] "{$5000.00}"
#> 
#> [[4]]
#> [1] "{$15000.00}" "{$11000.00}" "{$420.00}"   "{$15000.00}"
#> 
#> [[5]]
#> [1] "{$0.00}" "{$0.00}" "{$0.00}" "{$0.00}"
#> 
#> [[6]]
#> [1] "{$650.00}"
```

In Section \@ref(customfeatures), we will use an approach like this for custom feature engineering from the text.

### Building our first classification model {#classfirstmodel}

This data set includes more possible predictors than the text alone, but for this first model we will only use the text variable `consumer_complaint_narrative`.
Let's create a factor outcome variable `product` with two levels, "Credit" and "Other".
Then, we split the data into training and testing data sets.
We can use the `initial_split()` function from **rsample** to create this binary split of the data. 
The `strata` argument ensures that the distribution of `product` is similar in the training set and testing set. 
Since the split uses random sampling, we set a seed so we can reproduce our results.


```r
library(tidymodels)

set.seed(1234)
complaints2class <- complaints %>%
  mutate(product = factor(if_else(
    product == paste("Credit reporting, credit repair services,",
                     "or other personal consumer reports"),
    "Credit", "Other"
  )))

complaints_split <- initial_split(complaints2class, strata = product)

complaints_train <- training(complaints_split)
complaints_test <- testing(complaints_split)
```

The dimensions of the two splits show that this first step worked as we planned.


```r
dim(complaints_train)
```

```
#> [1] 87911    18
```

```r
dim(complaints_test)
```

```
#> [1] 29303    18
```

Next we need to preprocess this data to prepare it for modeling; we have text data, and we need to build numeric features for machine learning from that text.

The **recipes** package, part of tidymodels, allows us to create a specification of preprocessing steps we want to perform. These transformations are estimated (or "trained") on the training set so that they can be applied in the same way on the testing set or new data at prediction time, without data leakage.
We initialize our set of preprocessing transformations with the `recipe()` function, using a formula expression to specify the variables, our outcome plus our predictor, along with the data set.


```r
complaints_rec <-
  recipe(product ~ consumer_complaint_narrative, data = complaints_train)
```

Now we add steps to process the text of the complaints; we use **textrecipes** to handle the `consumer_complaint_narrative` variable. First we tokenize the text to words with `step_tokenize()`. By default this uses `tokenizers::tokenize_words()`.
Before we calculate tf-idf we use `step_tokenfilter()` to only keep the 1000 most frequent tokens, to avoid creating too many variables in our first model. To finish, we use `step_tfidf()` to compute tf-idf.


```r
library(textrecipes)

complaints_rec <- complaints_rec %>%
  step_tokenize(consumer_complaint_narrative) %>%
  step_tokenfilter(consumer_complaint_narrative, max_tokens = 1e3) %>%
  step_tfidf(consumer_complaint_narrative)
```

Now that we have a full specification of the preprocessing recipe, we can build up a tidymodels `workflow()` to bundle together our modeling components.


```r
complaint_wf <- workflow() %>%
  add_recipe(complaints_rec)
```

Let's start with a naive Bayes model [@kim2006; @Kibriya2005; @Eibe2006], which is available in the tidymodels package **discrim**.
One of the main advantages of a naive Bayes model is its ability to handle a large number of features, such as those we deal with when using word count methods.
Here we have only kept the 1000 most frequent tokens, but we could have kept more tokens and a naive Bayes model would still be able to handle such predictors well. For now, we will limit the model to a moderate number of tokens.

<div class="rmdpackage">
<p>In <strong>tidymodels</strong>, the package for creating model specifications is <strong>parsnip</strong>. The <strong>parsnip</strong> package provides the functions for creating all the models we have used so far, but other extra packages provide more. The <strong>discrim</strong> package is an extension package for <strong>parsnip</strong> that contains model definitions for various discriminant analysis models, including naive Bayes.</p>
</div>


```r
library(discrim)
nb_spec <- naive_Bayes() %>%
  set_mode("classification") %>%
  set_engine("naivebayes")

nb_spec
```

```
#> Naive Bayes Model Specification (classification)
#> 
#> Computational engine: naivebayes
```

Now we have everything we need to fit our first classification model. We can add the naive Bayes model to our workflow, and then we can fit this workflow to our training data.


```r
nb_fit <- complaint_wf %>%
  add_model(nb_spec) %>%
  fit(data = complaints_train)
```

We have trained our first classification model!

### Evaluation

Like we discussed in Section \@ref(firstregressionevaluation), we should not use the test set to compare models or different model parameters. The test set is a precious resource that should only be used at the end of the model training process to estimate performance on new data. Instead, we will use **resampling** methods to evaluate our model.

Let's use resampling to estimate the performance of the naive Bayes classification model we just fit. We can do this using resampled data sets built from the training set. Let's create cross 10-fold cross-validation sets, and use these resampled sets for performance estimates.


```r
set.seed(234)
complaints_folds <- vfold_cv(complaints_train)

complaints_folds
```

```
#> #  10-fold cross-validation 
#> # A tibble: 10 x 2
#>    splits               id    
#>    <list>               <chr> 
#>  1 <split [79119/8792]> Fold01
#>  2 <split [79120/8791]> Fold02
#>  3 <split [79120/8791]> Fold03
#>  4 <split [79120/8791]> Fold04
#>  5 <split [79120/8791]> Fold05
#>  6 <split [79120/8791]> Fold06
#>  7 <split [79120/8791]> Fold07
#>  8 <split [79120/8791]> Fold08
#>  9 <split [79120/8791]> Fold09
#> 10 <split [79120/8791]> Fold10
```

Each of these splits contains information about how to create cross-validation folds from the original training data. In this example, 90% of the training data is included in each fold and the other 10% is held out for evaluation.

For convenience, let's again use a `workflow()` for our resampling estimates of performance. 

<div class="rmdwarning">
<p>Using a <code>workflow()</code> isn’t required (you can fit or tune a model plus a preprocessor) but it can make your code easier to read and organize.</p>
</div>


```r
nb_wf <- workflow() %>%
  add_recipe(complaints_rec) %>%
  add_model(nb_spec)

nb_wf
```

```
#> ══ Workflow ════════════════════════════════════════════════════════════════════
#> Preprocessor: Recipe
#> Model: naive_Bayes()
#> 
#> ── Preprocessor ────────────────────────────────────────────────────────────────
#> 3 Recipe Steps
#> 
#> ● step_tokenize()
#> ● step_tokenfilter()
#> ● step_tfidf()
#> 
#> ── Model ───────────────────────────────────────────────────────────────────────
#> Naive Bayes Model Specification (classification)
#> 
#> Computational engine: naivebayes
```

In the last section, we fit one time to the training data as a whole. Now, to estimate how well that model performs, let's fit the model many times, once to each of these resampled folds, and then evaluate on the heldout part of each resampled fold.


```r
nb_rs <- fit_resamples(
  nb_wf,
  complaints_folds,
  control = control_resamples(save_pred = TRUE)
)
```

We can extract the relevant information using `collect_metrics()` and `collect_predictions()`


```r
nb_rs_metrics <- collect_metrics(nb_rs)
nb_rs_predictions <- collect_predictions(nb_rs)
```

What results do we see, in terms of performance metrics?


```r
nb_rs_metrics
```

```
#> # A tibble: 2 x 6
#>   .metric  .estimator  mean     n  std_err .config             
#>   <chr>    <chr>      <dbl> <int>    <dbl> <chr>               
#> 1 accuracy binary     0.806    10 0.00184  Preprocessor1_Model1
#> 2 roc_auc  binary     0.878    10 0.000715 Preprocessor1_Model1
```

The default performance parameters for binary classification are accuracy and ROC AUC (area under the receiver operator characteristic curve). For these resamples, the average accuracy is 80.6%.

<div class="rmdnote">
<p>Accuracy and ROC AUC are performance metrics used for classification models. For both, values closer to 1 are better.</p>
<p>Accuracy is the proportion of the data that are predicted correctly. Be aware that accuracy can be misleading in some situations, such as for imbalanced data sets.</p>
<p>ROC AUC measures how well a classifier performs at different thresholds. The ROC curve plots the true positive rate against the false positive rate, and AUC closer to 1 indicates a better-performing model while AUC closer to 0.5 indicates a model that does no better than random guessing.</p>
</div>

Figure \@ref(fig:firstroccurve) shows the ROC curve, a visualization of how well a classification model can distinguish between classes, for our first classification model on each of the resampled data sets.


```r
nb_rs_predictions %>%
  group_by(id) %>%
  roc_curve(truth = product, .pred_Credit) %>%
  autoplot() +
  labs(
    color = NULL,
    title = "ROC curve for US Consumer Finance Complaints",
    subtitle = "Each resample fold is shown in a different color"
  )
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/firstroccurve-1.png" alt="ROC curve for naive Bayes classifier with resamples of US Consumer Finance Bureau complaints" width="672" />
<p class="caption">(\#fig:firstroccurve)ROC curve for naive Bayes classifier with resamples of US Consumer Finance Bureau complaints</p>
</div>

The area under each of these curves is the `roc_auc` metric we have computed. If the curve was close to the diagonal line, then the model's predictions would be no better than random guessing.

Another way to evaluate our model is to evaluate the confusion matrix. A confusion matrix tabulates a model's false positives and false negatives for each class.
The function `conf_mat_resampled()` computes a separate confusion matrix for each resample and takes the average of the cell counts. This allows us to visualize an overall confusion matrix rather than needing to examine each resample individually.


```r
conf_mat_resampled(nb_rs) %>%
  autoplot(type = "heatmap")
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/firstheatmap-1.png" alt="Confusion matrix for naive Bayes classifier, showing some bias towards predicting 'Credit'" width="672" />
<p class="caption">(\#fig:firstheatmap)Confusion matrix for naive Bayes classifier, showing some bias towards predicting 'Credit'</p>
</div>

In Figure \@ref(fig:firstheatmap), the squares for "Credit"/"Credit" and "Other"/"Other" have a darker shade than the off diagonal squares. This is a good sign, meaning that our model is right more often than not! However, this first model is struggling somewhat since many observations from the "Other" class are being mispredicted as "Credit".

<div class="rmdwarning">
<p>One metric alone cannot give you a complete picture of how well your classification model is performing. The confusion matrix is a good starting point to get an overview of your model performance, as it includes rich information.</p>
</div>

This is real data from a government agency, and these kinds of performance metrics must be interpreted in the context of how such a model would be used. What happens if the model we trained gets a classification wrong for a consumer complaint? What impact will it have if more "Credit" complaints are correctly identified than "Other" complaints, either for consumers or for policymakers? 

## Compare to the null model {#classnull}

Like we did in Section \@ref(regnull), we can assess a model like this one by comparing its performance to a "null model" or baseline model, a simple, non-informative model that always predicts the largest class for classification. Such a model is perhaps the simplest heuristic or rule-based alternative that we can consider as we assess our modeling efforts.

We can build a classification `null_model()` specification and add it to a `workflow()` with the same preprocessing recipe we used in the previous section, to estimate performance.


```r
null_classification <- null_model() %>%
  set_engine("parsnip") %>%
  set_mode("classification")

null_rs <- workflow() %>%
  add_recipe(complaints_rec) %>%
  add_model(null_classification) %>%
  fit_resamples(
    complaints_folds
  )
```

What results do we obtain from the null model, in terms of performance metrics?


```r
null_rs %>%
  collect_metrics()
```

```
#> # A tibble: 2 x 6
#>   .metric  .estimator  mean     n std_err .config             
#>   <chr>    <chr>      <dbl> <int>   <dbl> <chr>               
#> 1 accuracy binary     0.526    10 0.00149 Preprocessor1_Model1
#> 2 roc_auc  binary     0.5      10 0       Preprocessor1_Model1
```

The accuracy and ROC AUC indicate that this null model is, like in the regression case, dramatically worse than even our first model. The text of the CFPB complaints is predictive relative to the category we are building models for.


## Compare to a lasso classification model {#comparetolasso}

Regularized linear models are a class of statistical model that can be used in regression and classification tasks. Linear models are not considered cutting edge in NLP research, but are a workhorse in real-world practice. Here we will use a lasso regularized model [@Tibshirani1996], where the regularization method also performs variable selection. In text analysis, we typically have many tokens, which are the features in our machine learning problem. 

<div class="rmdnote">
<p>Using regularization helps us choose a simpler model that we expect to generalize better to new observations, and variable selection helps us identify which features to include in our model.</p>
</div>

Lasso regression or classification learns how much of a _penalty_ to put on some features (sometimes penalizing all the way down to zero) so that we can select only some features out of the high-dimensional space of original possible variables (tokens) for the final model.

Let's create a specification of lasso regularized model. Remember that in tidymodels, specifying a model has three components: the algorithm, the mode, and the computational engine. 


```r
lasso_spec <- logistic_reg(penalty = 0.01, mixture = 1) %>%
  set_mode("classification") %>%
  set_engine("glmnet")

lasso_spec
```

```
#> Logistic Regression Model Specification (classification)
#> 
#> Main Arguments:
#>   penalty = 0.01
#>   mixture = 1
#> 
#> Computational engine: glmnet
```

Then we can create another `workflow()` object with the lasso specification. Notice that we can reuse our text preprocessing recipe.


```r
lasso_wf <- workflow() %>%
  add_recipe(complaints_rec) %>%
  add_model(lasso_spec)

lasso_spec
```

```
#> Logistic Regression Model Specification (classification)
#> 
#> Main Arguments:
#>   penalty = 0.01
#>   mixture = 1
#> 
#> Computational engine: glmnet
```

Now we estimate the performance of this first lasso classification model with `fit_resamples()`.


```r
set.seed(2020)
lasso_rs <- fit_resamples(
  lasso_wf,
  complaints_folds,
  control = control_resamples(save_pred = TRUE)
)
```

Let's again extract the relevant information using `collect_metrics()` and `collect_predictions()`


```r
lasso_rs_metrics <- collect_metrics(lasso_rs)
lasso_rs_predictions <- collect_predictions(lasso_rs)
```

Now we can see that `lasso_rs_metrics` contains the same default performance metrics we have been using so far in this chapter.


```r
lasso_rs_metrics
```

```
#> # A tibble: 2 x 6
#>   .metric  .estimator  mean     n  std_err .config             
#>   <chr>    <chr>      <dbl> <int>    <dbl> <chr>               
#> 1 accuracy binary     0.868    10 0.000977 Preprocessor1_Model1
#> 2 roc_auc  binary     0.939    10 0.000849 Preprocessor1_Model1
```

This looks pretty promising, considering we haven't yet done any tuning of the lasso hyperparameters.
Figure \@ref(fig:lassoroccurve) shows the ROC curves for this regularized model on each of the resampled data sets.


```r
lasso_rs_predictions %>%
  group_by(id) %>%
  roc_curve(truth = product, .pred_Credit) %>%
  autoplot() +
  labs(
    color = NULL,
    title = "ROC curve for US Consumer Finance Complaints",
    subtitle = "Each resample fold is shown in a different color"
  )
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/lassoroccurve-1.png" alt="ROC curve for lasso regularized classifier with resamples of US Consumer Finance Bureau complaints" width="672" />
<p class="caption">(\#fig:lassoroccurve)ROC curve for lasso regularized classifier with resamples of US Consumer Finance Bureau complaints</p>
</div>

Let's finish this section by generating a confusion matrix, shown in Figure \@ref(fig:lassoheatmap).
Our lasso model is better at separating the classes than the naive Bayes model in Section \@ref(classfirstmodel), and our results are more symmetrical than those for the naive Bayes model in Figure \@ref(fig:firstheatmap).


```r
conf_mat_resampled(lasso_rs) %>%
  autoplot(type = "heatmap")
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/lassoheatmap-1.png" alt="Confusion matrix for a lasso regularized classifier, with more symmetric results" width="672" />
<p class="caption">(\#fig:lassoheatmap)Confusion matrix for a lasso regularized classifier, with more symmetric results</p>
</div>


## Tuning lasso hyperparameters {#tunelasso}

The value `penalty = 0.01` for regularization in Section \@ref(comparetolasso) was picked somewhat arbitrarily. How do we know the *right* or *best* regularization parameter penalty? This is a model hyperparameter and we cannot learn its best value during model training, but we can estimate the best value by training many models on resampled data sets and exploring how well all these models perform. Let's build a new model specification for **model tuning**. 


```r
tune_spec <- logistic_reg(penalty = tune(), mixture = 1) %>%
  set_mode("classification") %>%
  set_engine("glmnet")

tune_spec
```

```
#> Logistic Regression Model Specification (classification)
#> 
#> Main Arguments:
#>   penalty = tune()
#>   mixture = 1
#> 
#> Computational engine: glmnet
```

After the tuning process, we can select a single best numeric value.

<div class="rmdnote">
<p>Think of <code>tune()</code> here as a placeholder for the regularization penalty.</p>
</div>

We can create a regular grid of values to try, using a convenience function for `penalty()`.


```r
lambda_grid <- grid_regular(penalty(), levels = 30)
lambda_grid
```

```
#> # A tibble: 30 x 1
#>     penalty
#>       <dbl>
#>  1 1.00e-10
#>  2 2.21e-10
#>  3 4.89e-10
#>  4 1.08e- 9
#>  5 2.40e- 9
#>  6 5.30e- 9
#>  7 1.17e- 8
#>  8 2.59e- 8
#>  9 5.74e- 8
#> 10 1.27e- 7
#> # … with 20 more rows
```

The function `grid_regular()` is from the **dials** package. It chooses sensible values to try for a parameter like the regularization penalty; here, we asked for 30 different possible values.

Now it is time to tune! Let's use `tune_grid()` to fit a model at each of the values for the regularization penalty in our regular grid.

<div class="rmdpackage">
<p>In <strong>tidymodels</strong>, the package for tuning is called <strong>tune</strong>. Tuning a model uses a similar syntax compared to fitting a model to a set of resampled data sets for the purposes of evaluation (<code>fit_resamples()</code>) because the two tasks are so similar. The difference is that when you tune, each model that you fit has <em>different</em> parameters and you want to find the best one.</p>
</div>

We add our tunable model specification `tune_spec` to a workflow with the same preprocessing recipe we've been using so far, and then fit it to every possible parameter in `lambda_grid` and every resample in `complaints_folds` with `tune_grid()`.


```r
tune_wf <- workflow() %>%
  add_recipe(complaints_rec) %>%
  add_model(tune_spec)

set.seed(2020)
tune_rs <- tune_grid(
  tune_wf,
  complaints_folds,
  grid = lambda_grid,
  control = control_resamples(save_pred = TRUE)
)

tune_rs
```

```
#> # Tuning results
#> # 10-fold cross-validation 
#> # A tibble: 10 x 5
#>    splits             id     .metrics        .notes         .predictions        
#>    <list>             <chr>  <list>          <list>         <list>              
#>  1 <split [79119/879… Fold01 <tibble [60 × … <tibble [0 × … <tibble [263,760 × …
#>  2 <split [79120/879… Fold02 <tibble [60 × … <tibble [0 × … <tibble [263,730 × …
#>  3 <split [79120/879… Fold03 <tibble [60 × … <tibble [0 × … <tibble [263,730 × …
#>  4 <split [79120/879… Fold04 <tibble [60 × … <tibble [0 × … <tibble [263,730 × …
#>  5 <split [79120/879… Fold05 <tibble [60 × … <tibble [0 × … <tibble [263,730 × …
#>  6 <split [79120/879… Fold06 <tibble [60 × … <tibble [0 × … <tibble [263,730 × …
#>  7 <split [79120/879… Fold07 <tibble [60 × … <tibble [0 × … <tibble [263,730 × …
#>  8 <split [79120/879… Fold08 <tibble [60 × … <tibble [0 × … <tibble [263,730 × …
#>  9 <split [79120/879… Fold09 <tibble [60 × … <tibble [0 × … <tibble [263,730 × …
#> 10 <split [79120/879… Fold10 <tibble [60 × … <tibble [0 × … <tibble [263,730 × …
```

<div class="rmdwarning">
<p>Like when we used <code>fit_resamples()</code>, tuning in tidymodels can use multiple cores or multiple machines via parallel processing, because the resampled data sets and possible parameters are independent of each other. A discussion of parallel processing for all possible operating systems is beyond the scope of this book, but it is well worth your time to learn how to parallelize your machine learning tasks on <em>your</em> system.</p>
</div>

Now, instead of one set of metrics, we have a set of metrics for each value of the regularization penalty.


```r
collect_metrics(tune_rs)
```

```
#> # A tibble: 60 x 7
#>     penalty .metric  .estimator  mean     n  std_err .config              
#>       <dbl> <chr>    <chr>      <dbl> <int>    <dbl> <chr>                
#>  1 1.00e-10 accuracy binary     0.890    10 0.00102  Preprocessor1_Model01
#>  2 1.00e-10 roc_auc  binary     0.952    10 0.000823 Preprocessor1_Model01
#>  3 2.21e-10 accuracy binary     0.890    10 0.00102  Preprocessor1_Model02
#>  4 2.21e-10 roc_auc  binary     0.952    10 0.000823 Preprocessor1_Model02
#>  5 4.89e-10 accuracy binary     0.890    10 0.00102  Preprocessor1_Model03
#>  6 4.89e-10 roc_auc  binary     0.952    10 0.000823 Preprocessor1_Model03
#>  7 1.08e- 9 accuracy binary     0.890    10 0.00102  Preprocessor1_Model04
#>  8 1.08e- 9 roc_auc  binary     0.952    10 0.000823 Preprocessor1_Model04
#>  9 2.40e- 9 accuracy binary     0.890    10 0.00102  Preprocessor1_Model05
#> 10 2.40e- 9 roc_auc  binary     0.952    10 0.000823 Preprocessor1_Model05
#> # … with 50 more rows
```

Let's visualize these metrics, accuracy and ROC AUC, in Figure \@ref(fig:complaintstunevis) to see what the best model is.


```r
autoplot(tune_rs) +
  labs(
    title = "Lasso model performance across regularization penalties",
    subtitle = "Performance metrics can be used to identity the best penalty"
  )
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/complaintstunevis-1.png" alt="We can identify the best regularization penalty from model performance metrics, for example, at the highest ROC AUC. Note the logarithmic scale for the regularization penalty." width="672" />
<p class="caption">(\#fig:complaintstunevis)We can identify the best regularization penalty from model performance metrics, for example, at the highest ROC AUC. Note the logarithmic scale for the regularization penalty.</p>
</div>

We can view the best results with `show_best()` and a choice for the metric, such as ROC AUC.


```r
tune_rs %>%
  show_best("roc_auc")
```

```
#> # A tibble: 5 x 7
#>        penalty .metric .estimator  mean     n  std_err .config              
#>          <dbl> <chr>   <chr>      <dbl> <int>    <dbl> <chr>                
#> 1 0.000356     roc_auc binary     0.953    10 0.000824 Preprocessor1_Model20
#> 2 0.000788     roc_auc binary     0.953    10 0.000827 Preprocessor1_Model21
#> 3 0.000161     roc_auc binary     0.953    10 0.000822 Preprocessor1_Model19
#> 4 0.0000728    roc_auc binary     0.953    10 0.000821 Preprocessor1_Model18
#> 5 0.0000000001 roc_auc binary     0.952    10 0.000823 Preprocessor1_Model01
```



The best value for ROC AUC from this tuning run is 0.953. We can extract the best regularization parameter for this value of ROC AUC from our tuning results with `select_best()`, or a simpler model with higher regularization with `select_by_pct_loss()` or `select_by_one_std_err()` Let's choose the model with the best ROC AUC within one standard error of the numerically best model [@Breiman1984].


```r
chosen_auc <- tune_rs %>%
  select_by_one_std_err(metric = "roc_auc", -penalty)

chosen_auc
```

```
#> # A tibble: 1 x 9
#>    penalty .metric .estimator  mean     n  std_err .config          .best .bound
#>      <dbl> <chr>   <chr>      <dbl> <int>    <dbl> <chr>            <dbl>  <dbl>
#> 1 0.000788 roc_auc binary     0.953    10 0.000827 Preprocessor1_M… 0.953  0.952
```

Next, let's finalize our tunable workflow with this particular regularization penalty. This is the regularization penalty that our tuning results indicate give us the best model.


```r
final_lasso <- finalize_workflow(tune_wf, chosen_auc)

final_lasso
```

```
#> ══ Workflow ════════════════════════════════════════════════════════════════════
#> Preprocessor: Recipe
#> Model: logistic_reg()
#> 
#> ── Preprocessor ────────────────────────────────────────────────────────────────
#> 3 Recipe Steps
#> 
#> ● step_tokenize()
#> ● step_tokenfilter()
#> ● step_tfidf()
#> 
#> ── Model ───────────────────────────────────────────────────────────────────────
#> Logistic Regression Model Specification (classification)
#> 
#> Main Arguments:
#>   penalty = 0.000788046281566992
#>   mixture = 1
#> 
#> Computational engine: glmnet
```

Instead of `penalty = tune()` like before, now our workflow has finalized values for all arguments. The preprocessing recipe has been evaluated on the training data, and we tuned the regularization penalty so that we have a penalty value of 0.00079. This workflow is ready to go! It can now be fit to our training data.


```r
fitted_lasso <- fit(final_lasso, complaints_train)
```

What does the result look like? We can access the fit using `pull_workflow_fit()`, and even `tidy()` the model coefficient results into a convenient dataframe format.


```r
fitted_lasso %>%
  pull_workflow_fit() %>%
  tidy() %>%
  arrange(-estimate)
```

```
#> # A tibble: 1,001 x 3
#>    term                                         estimate  penalty
#>    <chr>                                           <dbl>    <dbl>
#>  1 tfidf_consumer_complaint_narrative_funds         26.5 0.000788
#>  2 tfidf_consumer_complaint_narrative_appraisal     22.1 0.000788
#>  3 tfidf_consumer_complaint_narrative_bonus         21.4 0.000788
#>  4 tfidf_consumer_complaint_narrative_debt          19.9 0.000788
#>  5 tfidf_consumer_complaint_narrative_escrow        17.8 0.000788
#>  6 tfidf_consumer_complaint_narrative_customers     17.2 0.000788
#>  7 tfidf_consumer_complaint_narrative_money         16.5 0.000788
#>  8 tfidf_consumer_complaint_narrative_emailed       15.9 0.000788
#>  9 tfidf_consumer_complaint_narrative_fees          15.1 0.000788
#> 10 tfidf_consumer_complaint_narrative_interest      14.5 0.000788
#> # … with 991 more rows
```

We see here, for the penalty we chose, what terms contribute the most to a complaint _not_ being about credit. The words are largely about mortgages and other financial products.

What terms contribute to a complaint being about credit reporting, for this tuned model? Here we see the names of the credit reporting agencies and words about credit inquiries.


```r
fitted_lasso %>%
  pull_workflow_fit() %>%
  tidy() %>%
  arrange(estimate)
```

```
#> # A tibble: 1,001 x 3
#>    term                                          estimate  penalty
#>    <chr>                                            <dbl>    <dbl>
#>  1 tfidf_consumer_complaint_narrative_reseller      -86.4 0.000788
#>  2 tfidf_consumer_complaint_narrative_experian      -59.2 0.000788
#>  3 tfidf_consumer_complaint_narrative_transunion    -51.9 0.000788
#>  4 tfidf_consumer_complaint_narrative_equifax       -48.0 0.000788
#>  5 tfidf_consumer_complaint_narrative_compliant     -21.8 0.000788
#>  6 tfidf_consumer_complaint_narrative_reporting     -21.5 0.000788
#>  7 tfidf_consumer_complaint_narrative_report        -17.1 0.000788
#>  8 tfidf_consumer_complaint_narrative_freeze        -17.1 0.000788
#>  9 tfidf_consumer_complaint_narrative_inquiries     -16.9 0.000788
#> 10 tfidf_consumer_complaint_narrative_method        -16.0 0.000788
#> # … with 991 more rows
```

<div class="rmdnote">
<p>Since we are using a linear model, the model coefficients are directly interpretable and transparently give us variable importance. Many models useful for machine learning with text do <em>not</em> have such transparent variable importance; in those situations, you can use other model-independent or model-agnostic approaches like <a href="https://juliasilge.com/blog/last-airbender/">permutation variable importance</a>.</p>
</div>

## Case study: sparse encoding {#casestudysparseencoding}

We can change how our text data is represented to take advantage of its sparsity, especially for models like lasso regularized models. The regularized regression model we have been training in previous sections used `set_engine("glmnet")`; this computational engine can be more efficient when text data is transformed to a sparse matrix (Section \@ref(motivatingsparse)), rather than a dense data frame or tibble representation.

To keep our text data sparse throughout modeling and use the sparse capabilities of `set_engine("glmnet")`, we need to explicitly set a non-default preprocessing blueprint, using the package [**hardhat**](https://hardhat.tidymodels.org/).

<div class="rmdpackage">
<p>The <strong>hardhat</strong> package is used by other tidymodels packages like recipes and parsnip under the hood. As a tidymodels user, you typically don’t use hardhat functions directly. The exception is when you need to customize something about your model or preprocessing, like in this sparse data example.</p>
</div>


```r
library(hardhat)
sparse_bp <- default_recipe_blueprint(composition = "dgCMatrix")
```

This "blueprint" lets us specify during modeling how we want our data passed around from the preprocessing into the model. The composition `"dgCMatrix"` is the most common sparse matrix type, from the Matrix package [@R-Matrix], used in R for modeling. We can use this `blueprint` argument when we add our recipe to our modeling workflow, to define how the data should be passed into the model.


```r
sparse_wf <- workflow() %>%
  add_recipe(complaints_rec, blueprint = sparse_bp) %>%
  add_model(tune_spec)

sparse_wf
```

```
#> ══ Workflow ════════════════════════════════════════════════════════════════════
#> Preprocessor: Recipe
#> Model: logistic_reg()
#> 
#> ── Preprocessor ────────────────────────────────────────────────────────────────
#> 3 Recipe Steps
#> 
#> ● step_tokenize()
#> ● step_tokenfilter()
#> ● step_tfidf()
#> 
#> ── Model ───────────────────────────────────────────────────────────────────────
#> Logistic Regression Model Specification (classification)
#> 
#> Main Arguments:
#>   penalty = tune()
#>   mixture = 1
#> 
#> Computational engine: glmnet
```

The last time we tuned a lasso model, we used the defaults for the penalty parameter and 30 levels. Let's restrict the values this time using the `range` argument, so we don't test out as small values for regularization, and only try 20 levels.


```r
smaller_lambda <- grid_regular(penalty(range = c(-5, 0)), levels = 20)
smaller_lambda
```

```
#> # A tibble: 20 x 1
#>      penalty
#>        <dbl>
#>  1 0.00001  
#>  2 0.0000183
#>  3 0.0000336
#>  4 0.0000616
#>  5 0.000113 
#>  6 0.000207 
#>  7 0.000379 
#>  8 0.000695 
#>  9 0.00127  
#> 10 0.00234  
#> 11 0.00428  
#> 12 0.00785  
#> 13 0.0144   
#> 14 0.0264   
#> 15 0.0483   
#> 16 0.0886   
#> 17 0.162    
#> 18 0.298    
#> 19 0.546    
#> 20 1
```

We can tune this lasso regression model, in the same way that we did in Section \@ref(tunelasso). We will fit and assess each possible regularization parameter on each resampling fold, to find the best amount of regularization.


```r
set.seed(2020)
sparse_rs <- tune_grid(
  sparse_wf,
  complaints_folds,
  grid = smaller_lambda
)

sparse_rs
```

```
#> # Tuning results
#> # 10-fold cross-validation 
#> # A tibble: 10 x 4
#>    splits               id     .metrics          .notes          
#>    <list>               <chr>  <list>            <list>          
#>  1 <split [79119/8792]> Fold01 <tibble [40 × 5]> <tibble [0 × 1]>
#>  2 <split [79120/8791]> Fold02 <tibble [40 × 5]> <tibble [0 × 1]>
#>  3 <split [79120/8791]> Fold03 <tibble [40 × 5]> <tibble [0 × 1]>
#>  4 <split [79120/8791]> Fold04 <tibble [40 × 5]> <tibble [0 × 1]>
#>  5 <split [79120/8791]> Fold05 <tibble [40 × 5]> <tibble [0 × 1]>
#>  6 <split [79120/8791]> Fold06 <tibble [40 × 5]> <tibble [0 × 1]>
#>  7 <split [79120/8791]> Fold07 <tibble [40 × 5]> <tibble [0 × 1]>
#>  8 <split [79120/8791]> Fold08 <tibble [40 × 5]> <tibble [0 × 1]>
#>  9 <split [79120/8791]> Fold09 <tibble [40 × 5]> <tibble [0 × 1]>
#> 10 <split [79120/8791]> Fold10 <tibble [40 × 5]> <tibble [0 × 1]>
```

How did this model turn out, especially compared to the tuned model that did not use the sparse capabilities of `set_engine("glmnet")`?


```r
sparse_rs %>%
  show_best("roc_auc")
```

```
#> # A tibble: 5 x 7
#>     penalty .metric .estimator  mean     n  std_err .config              
#>       <dbl> <chr>   <chr>      <dbl> <int>    <dbl> <chr>                
#> 1 0.000695  roc_auc binary     0.953    10 0.000825 Preprocessor1_Model08
#> 2 0.000379  roc_auc binary     0.953    10 0.000824 Preprocessor1_Model07
#> 3 0.000207  roc_auc binary     0.953    10 0.000821 Preprocessor1_Model06
#> 4 0.000113  roc_auc binary     0.953    10 0.000820 Preprocessor1_Model05
#> 5 0.0000616 roc_auc binary     0.952    10 0.000822 Preprocessor1_Model04
```

The best ROC AUC is nearly identical; the best ROC AUC for the non-sparse tuned lasso model in Section \@ref(tunelasso) was 0.953. The best regularization parameter (`penalty`) is a little different (the best value in Section \@ref(tunelasso) was 0.00036) but we used a different grid so didn't try out exactly the same values. We ended up with nearly the same performance and best tuned model.

Importantly, this tuning also took a bit less time to complete. 

- The _preprocessing_ was not much faster, because tokenization and computing tf-idf take a long time. 
- The _model fitting_ was much faster, because for highly sparse data, this implementation of regularized regression is much faster for sparse matrix input than any dense input. 

Overall, the whole tuning workflow is about 10% faster using the sparse preprocessing blueprint. Depending on how computationally expensive your preprocessing is relative to your model and how sparse your data is, you may expect to see larger (or smaller) gains from moving to a sparse data representation.

<div class="rmdnote">
<p>Since our model performance is about the same and we see gains in training time, let’s use this sparse representation for the rest of this chapter.</p>
</div>

## Two class or multiclass? {#mlmulticlass}

Most of this chapter focuses on binary classification, where we have two classes in our outcome variable (such as "Credit" and "Other") and each observation can either be one or the other. This is a simple scenario with straightforward evaluation strategies because the results only have a two-by-two contingency matrix.
However, it is not always possible to limit a modeling question to two classes. Let's explore how to deal with situations where we have more than two classes.
The CFPB complaints data set in this chapter has nine different `product` classes. In decreasing frequency, they are:

- Credit reporting, credit repair services, or other personal consumer reports
- Debt collection
- Credit card or prepaid card
- Mortgage
- Checking or savings account
- Student loan
- Vehicle loan or lease
- Money transfer, virtual currency, or money service
- Payday loan, title loan, or personal loan

We assume that there is a reason why these product classes have been created in this fashion by this government agency.
Perhaps complaints from different classes are handled by different people or organizations.
Whatever the reason, in this section we would like to build a multiclass classifier to identify these nine specific product classes.

We need to create a new split of the data using `initial_split()` on the unmodified `complaints` data set.


```r
set.seed(1234)

multicomplaints_split <- initial_split(complaints, strata = product)

multicomplaints_train <- training(multicomplaints_split)
multicomplaints_test <- testing(multicomplaints_split)
```

Before we continue, let us take a look at the number of cases in each of the classes.


```r
multicomplaints_train %>%
  count(product, sort = TRUE) %>%
  select(n, product)
```

```
#> # A tibble: 9 x 2
#>       n product                                                                 
#>   <int> <chr>                                                                   
#> 1 41628 Credit reporting, credit repair services, or other personal consumer re…
#> 2 16722 Debt collection                                                         
#> 3  8695 Credit card or prepaid card                                             
#> 4  7067 Mortgage                                                                
#> 5  5238 Checking or savings account                                             
#> 6  2960 Student loan                                                            
#> 7  2028 Vehicle loan or lease                                                   
#> 8  1926 Money transfer, virtual currency, or money service                      
#> 9  1647 Payday loan, title loan, or personal loan
```

There is significant imbalance between the classes that we must address, with over twenty times more cases of the majority class than there is of the smallest class.
This kind of imbalance is a common problem with multiclass classification, with few multiclass data sets in the real world exhibiting balance between classes.

Compared to binary classification, there are several additional issues to keep in mind when working with multiclass classification:

- Many machine learning algorithms do not handle imbalanced data well and are likely to have a hard time predicting minority classes.
- Not all machine learning algorithms are built for multiclass classification at all.
- Many evaluation metrics need to be reformulated to describe multiclass predictions.

When you have multiple classes in your data, it is possible to formulate the multiclass problem in two ways. With one approach, any given observation can belong to multiple classes. With the other approach, an observation can belong to one and only one class. We will be sticking to the second, "one class per observation" model formulation in this section.

There are many different ways to deal with imbalanced data.
We will demonstrate one of the simplest methods, downsampling, where observations from the majority classes are removed during training to achieve a balanced class distribution.
We will be using the [**themis**](https://themis.tidymodels.org) add-on package for recipes which provides the `step_downsample()` function to perform downsampling.

<div class="rmdpackage">
<p>The <strong>themis</strong> package provides many more algorithms to deal with imbalanced data during data preprocessing.</p>
</div>

We have to create a new recipe specification from scratch, since we are dealing with new training data this time.
The specification `multicomplaints_rec` is similar to what we created in Section \@ref(classfirstattemptlookatdata). The only changes are that different data is passed to the `data` argument in the `recipe()` function (it is now `multicomplaints_train`) and we have added `step_downsample(product)` to the end of the recipe specification to downsample after all the text preprocessing. We want to downsample last so that we still generate features on the full training data set. The downsampling will then _only_ affect the modeling step, not the preprocessing steps, with hopefully better results.


```r
library(themis)

multicomplaints_rec <-
  recipe(product ~ consumer_complaint_narrative,
         data = multicomplaints_train) %>%
  step_tokenize(consumer_complaint_narrative) %>%
  step_tokenfilter(consumer_complaint_narrative, max_tokens = 1e3) %>%
  step_tfidf(consumer_complaint_narrative) %>%
  step_downsample(product)
```

We also need a new cross-validation object since we are using a different data set.


```r
multicomplaints_folds <- vfold_cv(multicomplaints_train)
```

We cannot reuse the tuneable lasso classification specification from Section \@ref(tunelasso) because it only works for binary classification. Some model algorithms and computational engines (examples are most random forests and SVMs) automatically detect when we perform multiclass classification from the number of classes in the outcome variable and do not require any changes to our model specification. For lasso regularization, we need to create a new special model specification just for the multiclass class using `multinom_reg()`.


```r
multi_spec <- multinom_reg(penalty = tune(), mixture = 1) %>%
  set_mode("classification") %>%
  set_engine("glmnet")

multi_spec
```

```
#> Multinomial Regression Model Specification (classification)
#> 
#> Main Arguments:
#>   penalty = tune()
#>   mixture = 1
#> 
#> Computational engine: glmnet
```

We used the same arguments for `penalty` and `mixture` as in Section \@ref(tunelasso), as well as the same mode and engine, but this model specification is set up to handle more than just two classes. We can combine this model specification with our preprocessing recipe for multiclass data in a `workflow()`.


```r
multi_lasso_wf <- workflow() %>%
  add_recipe(multicomplaints_rec, blueprint = sparse_bp) %>%
  add_model(multi_spec)

multi_lasso_wf
```

```
#> ══ Workflow ════════════════════════════════════════════════════════════════════
#> Preprocessor: Recipe
#> Model: multinom_reg()
#> 
#> ── Preprocessor ────────────────────────────────────────────────────────────────
#> 4 Recipe Steps
#> 
#> ● step_tokenize()
#> ● step_tokenfilter()
#> ● step_tfidf()
#> ● step_downsample()
#> 
#> ── Model ───────────────────────────────────────────────────────────────────────
#> Multinomial Regression Model Specification (classification)
#> 
#> Main Arguments:
#>   penalty = tune()
#>   mixture = 1
#> 
#> Computational engine: glmnet
```

Now we have everything we need to tune the regularization penalty and find an appropriate value. Note that we specify `save_pred = TRUE`, so we can create ROC curves and a confusion matrix later. This is especially beneficial for multiclass classification.


```r
multi_lasso_rs <- tune_grid(
  multi_lasso_wf,
  multicomplaints_folds,
  grid = smaller_lambda,
  control = control_resamples(save_pred = TRUE)
)

multi_lasso_rs
```

```
#> # Tuning results
#> # 10-fold cross-validation 
#> # A tibble: 10 x 5
#>    splits             id     .metrics        .notes         .predictions        
#>    <list>             <chr>  <list>          <list>         <list>              
#>  1 <split [79119/879… Fold01 <tibble [40 × … <tibble [0 × … <tibble [175,840 × …
#>  2 <split [79120/879… Fold02 <tibble [40 × … <tibble [0 × … <tibble [175,820 × …
#>  3 <split [79120/879… Fold03 <tibble [40 × … <tibble [1 × … <tibble [175,820 × …
#>  4 <split [79120/879… Fold04 <tibble [40 × … <tibble [0 × … <tibble [175,820 × …
#>  5 <split [79120/879… Fold05 <tibble [40 × … <tibble [1 × … <tibble [175,820 × …
#>  6 <split [79120/879… Fold06 <tibble [40 × … <tibble [1 × … <tibble [175,820 × …
#>  7 <split [79120/879… Fold07 <tibble [40 × … <tibble [1 × … <tibble [175,820 × …
#>  8 <split [79120/879… Fold08 <tibble [40 × … <tibble [1 × … <tibble [175,820 × …
#>  9 <split [79120/879… Fold09 <tibble [40 × … <tibble [0 × … <tibble [175,820 × …
#> 10 <split [79120/879… Fold10 <tibble [40 × … <tibble [0 × … <tibble [175,820 × …
```

What do we see, in terms of performance metrics?


```r
best_acc <- multi_lasso_rs %>%
  show_best("accuracy")

best_acc
```

```
#> # A tibble: 5 x 7
#>    penalty .metric  .estimator  mean     n std_err .config              
#>      <dbl> <chr>    <chr>      <dbl> <int>   <dbl> <chr>                
#> 1 0.00234  accuracy multiclass 0.755    10 0.00220 Preprocessor1_Model10
#> 2 0.00428  accuracy multiclass 0.751    10 0.00238 Preprocessor1_Model11
#> 3 0.00127  accuracy multiclass 0.749    10 0.00273 Preprocessor1_Model09
#> 4 0.00785  accuracy multiclass 0.740    10 0.00219 Preprocessor1_Model12
#> 5 0.000695 accuracy multiclass 0.740    10 0.00451 Preprocessor1_Model08
```

The accuracy metric naturally extends to multiclass tasks, but even the very best value is quite low at 75.5%, significantly lower than for the binary case in Section \@ref(tunelasso). This is expected since multiclass classification is a harder task than binary classification. 

<div class="rmdwarning">
<p>In binary classification, there is one right answer and one wrong answer; in this case, there is one right answer and <em>eight</em> wrong answers.</p>
</div>

To get a more detailed view of how our classifier is performing, let us look at one of the confusion matrices in Figure \@ref(fig:multiheatmap).


```r
multi_lasso_rs %>%
  collect_predictions() %>%
  filter(penalty == best_acc$penalty) %>%
  filter(id == "Fold01") %>%
  conf_mat(product, .pred_class) %>%
  autoplot(type = "heatmap") +
  scale_y_discrete(labels = function(x) str_wrap(x, 20)) +
  scale_x_discrete(labels = function(x) str_wrap(x, 20))
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/multiheatmap-1.png" alt="Confusion matrix for multiclass lasso regularized classifier, with most of the classifications along the diagonal" width="960" />
<p class="caption">(\#fig:multiheatmap)Confusion matrix for multiclass lasso regularized classifier, with most of the classifications along the diagonal</p>
</div>

The diagonal is fairly well populated, which is a good sign. This means that the model generally predicted the right class.
The off-diagonals numbers are all the failures and where we should direct our focus.
It is a little hard to see these cases well since the majority class affects the scale.
A trick to deal with this problem is to remove all the correctly predicted observations.


```r
multi_lasso_rs %>%
  collect_predictions() %>%
  filter(penalty == best_acc$penalty) %>%
  filter(id == "Fold01") %>%
  filter(.pred_class != product) %>%
  conf_mat(product, .pred_class) %>%
  autoplot(type = "heatmap") +
  scale_y_discrete(labels = function(x) str_wrap(x, 20)) +
  scale_x_discrete(labels = function(x) str_wrap(x, 20))
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/multiheatmapminusdiag-1.png" alt="Confusion matrix for multiclass lasso regularized classifier without diagonal" width="960" />
<p class="caption">(\#fig:multiheatmapminusdiag)Confusion matrix for multiclass lasso regularized classifier without diagonal</p>
</div>

Now we can more clearly see where our model breaks down in Figure \@ref(fig:multiheatmapminusdiag). Some of the most common errors are "Credit reporting, credit repair services, or other personal consumer reports" complaints being wrongly being predicted as "Debt collection" or "Credit card of prepaid card" complaints. Those mistakes by the model are not hard to understand since all deal with credit and debt and do have overlap in vocabulary.
Knowing what the problem is helps us figure out how to improve our model.
The next step for improving our model is to revisit the data preprocessing steps and model selection.
We can look at different models or model engines that might be able to more easily separate the classes.

Now that we have an idea of where the model isn't working, we can look more closely at the data and attempt to create features that could distinguish between these classes. In Section \@ref(customfeatures) we will demonstrate how you can create your own custom features.

## Case study: including non-text data

We are building a model from a data set that includes more than text data alone. Annotations and labels have been added by the CFPB that we can use during modeling, but we need to ensure that only information that would be available at the time of prediction is included in the model.
Otherwise we we will be very disappointed once our model is used to predict on new data!
The variables we identify as available for use as predictors are:

- `date_received`
- `issue`
- `sub_issue`
- `consumer_complaint_narrative`
- `company`
- `state`
- `zip_code`
- `tags`
- `submitted_via`

Let's try including `date_received` in our modeling, along with the text variable we have already used `consumer_complaint_narrative` and a new variable `tags`.
The `submitted_via` variable could have been a viable candidate, but all the entries are "web".
The other variables like ZIP code could be of use too, but they are categorical variables with many values so we will exclude them for now.


```r
more_vars_rec <-
  recipe(product ~ date_received + tags + consumer_complaint_narrative,
         data = complaints_train)
```

How should we preprocess the `date_received` variable? We can use the `step_date()` function to extract the month and day of the week (`"dow"`). Then we remove the original date variable and convert the new month and day-of-the-week columns to indicator variables with `step_dummy()`.

<div class="rmdnote">
<p>Categorical variables like the month can be stored as strings or factors, but for some kinds of models, they must be converted to indicator or dummy variables. These are numeric binary variables for the levels of the original categorical variable. For example, a variable called <code>December</code> would be created that is all zeroes and ones specifying which complaints were submitted in December, plus a variable called <code>November</code>, a variable called <code>October</code>, and so on.</p>
</div>


```r
more_vars_rec <- more_vars_rec %>%
  step_date(date_received, features = c("month", "dow"), role = "dates") %>%
  step_rm(date_received) %>%
  step_dummy(has_role("dates"))
```

The `tags` variable has some missing data. We can deal with this by using `step_unknown()`, which adds a new level to this factor variable for cases of missing data. Then we "dummify" (create dummy/indicator variables) the variable with `step_dummy()`


```r
more_vars_rec <- more_vars_rec %>%
  step_unknown(tags) %>%
  step_dummy(tags)
```

Now we add steps to process the text of the complaints, as before.


```r
more_vars_rec <- more_vars_rec %>%
  step_tokenize(consumer_complaint_narrative) %>%
  step_tokenfilter(consumer_complaint_narrative, max_tokens = 1e3) %>%
  step_tfidf(consumer_complaint_narrative)
```

Let's combine this more extensive preprocessing recipe that handles more variables together with the tuneable lasso regularized classification model specification.


```r
more_vars_wf <- workflow() %>%
  add_recipe(more_vars_rec, blueprint = sparse_bp) %>%
  add_model(tune_spec)

more_vars_wf
```

```
#> ══ Workflow ════════════════════════════════════════════════════════════════════
#> Preprocessor: Recipe
#> Model: logistic_reg()
#> 
#> ── Preprocessor ────────────────────────────────────────────────────────────────
#> 8 Recipe Steps
#> 
#> ● step_date()
#> ● step_rm()
#> ● step_dummy()
#> ● step_unknown()
#> ● step_dummy()
#> ● step_tokenize()
#> ● step_tokenfilter()
#> ● step_tfidf()
#> 
#> ── Model ───────────────────────────────────────────────────────────────────────
#> Logistic Regression Model Specification (classification)
#> 
#> Main Arguments:
#>   penalty = tune()
#>   mixture = 1
#> 
#> Computational engine: glmnet
```

Let's tune this `workflow()` with our resampled data sets, find a good value for the regularization penalty, and estimate the model's performance.


```r
set.seed(123)
more_vars_rs <- tune_grid(
  more_vars_wf,
  complaints_folds,
  grid = smaller_lambda,
)
```

We can extract the metrics for the best-performing regularization penalties from these results with `show_best()` with an option like `"roc_auc"` or `"accuracy"` if we prefer. How did our chosen performance metric turn out for our model that included more than just the text data?


```r
more_vars_rs %>%
  show_best("roc_auc")
```

```
#> # A tibble: 5 x 7
#>     penalty .metric .estimator  mean     n  std_err .config              
#>       <dbl> <chr>   <chr>      <dbl> <int>    <dbl> <chr>                
#> 1 0.000695  roc_auc binary     0.953    10 0.000824 Preprocessor1_Model08
#> 2 0.000379  roc_auc binary     0.953    10 0.000818 Preprocessor1_Model07
#> 3 0.000207  roc_auc binary     0.953    10 0.000814 Preprocessor1_Model06
#> 4 0.000113  roc_auc binary     0.953    10 0.000813 Preprocessor1_Model05
#> 5 0.0000616 roc_auc binary     0.953    10 0.000812 Preprocessor1_Model04
```

We see here that including more predictors did not measurably improve our model performance but it did change the regularization a bit. With only text features in Section \@ref(casestudysparseencoding) and the same grid and sparse encoding, we achieved an accuracy of 0.953, the same as what we see now by including the features dealing with dates and tags as well. The best regularization penalty in Section \@ref(casestudysparseencoding) was 0.0007 but here it is a bit higher, indicating that our model learned to regularize more strongly once we added these extra features. This makes sense, and we can use `tidy()` and some **dplyr** manipulation to find at what rank (`term_rank`) any of the date or tag variables were included in the regularized results, by absolute value of the model coefficient.


```r
finalize_workflow(more_vars_wf, 
                  select_best(more_vars_rs, "roc_auc")) %>%
  fit(complaints_train) %>%
  pull_workflow_fit() %>%
  tidy() %>% 
  arrange(-abs(estimate)) %>% 
  mutate(term_rank = row_number()) %>% 
  filter(!str_detect(term, "tfidf"))
```

```
#> # A tibble: 21 x 4
#>    term                    estimate  penalty term_rank
#>    <chr>                      <dbl>    <dbl>     <int>
#>  1 date_received_month_Dec -0.319   0.000695       726
#>  2 (Intercept)              0.256   0.000695       734
#>  3 date_received_dow_Mon    0.129   0.000695       758
#>  4 date_received_month_Apr  0.101   0.000695       763
#>  5 date_received_month_Aug -0.0923  0.000695       768
#>  6 date_received_dow_Fri    0.0422  0.000695       782
#>  7 date_received_month_Jul -0.0302  0.000695       785
#>  8 date_received_month_Feb -0.0270  0.000695       787
#>  9 tags_Servicemember      -0.0176  0.000695       789
#> 10 date_received_dow_Wed   -0.00257 0.000695       795
#> # … with 11 more rows
```

In our example here, some of the non-text predictors are included in the model with non-zero coefficients but ranked down in the 700s of all model terms, with smaller coefficients than many text terms. They are not that important.

<div class="rmdnote">
<p>This whole book focuses on supervised machine learning for text data, but models can combine <em>both</em> text predictors and other kinds of predictors.</p>
</div>


## Case study: data censoring

The complaints data set already has sensitive information (PII) censored or protected using strings such as "XXXX" and "XX".
This data censoring can be viewed as data _annotation_; specific account numbers and birthdays are protected but we know they were there. These values would be mostly unique anyway, and likely filtered out in their original form.

Figure \@ref(fig:censoredtrigram) shows the most frequent trigrams (Section \@ref(tokenizingngrams)) in our training data set.


```r
library(tidytext)

complaints_train %>%
  slice(1:1000) %>%
  unnest_tokens(trigrams, 
                consumer_complaint_narrative, token = "ngrams",
                collapse = NULL) %>%
  count(trigrams, sort = TRUE) %>%
  mutate(censored = str_detect(trigrams, "xx")) %>%
  slice(1:20) %>%
  ggplot(aes(n, reorder(trigrams, n), fill = censored)) +
  geom_col() +
  scale_fill_manual(values = c("grey40", "firebrick")) +
  labs(y = "Trigrams", x = "Count")
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/censoredtrigram-1.png" alt="Many of the most frequent trigrams feature censored information" width="672" />
<p class="caption">(\#fig:censoredtrigram)Many of the most frequent trigrams feature censored information</p>
</div>

The vast majority of trigrams in Figure \@ref(fig:censoredtrigram) include one or more censored words.
Not only do the most used trigrams include some kind of censoring, 
but the censoring itself is informative as it is not used uniformly across the product classes.
In Figure \@ref(fig:trigram25), we take the top 25 most frequent trigrams that include censoring,
and plot the proportions for "Credit" and "Other".


```r
top_censored_trigrams <- complaints_train %>%
  slice(1:1000) %>%
  unnest_tokens(trigrams, 
                consumer_complaint_narrative, token = "ngrams",
                collapse = NULL) %>%
  count(trigrams, sort = TRUE) %>%
  filter(str_detect(trigrams, "xx")) %>%
  slice(1:25)

plot_data <- complaints_train %>%
  unnest_tokens(trigrams, 
                consumer_complaint_narrative, token = "ngrams",
                collapse = NULL) %>%
  right_join(top_censored_trigrams, by = "trigrams") %>%
  count(trigrams, product, .drop = FALSE)

plot_data %>%
  ggplot(aes(n, trigrams, fill = product)) +
  geom_col(position = "fill")
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/trigram25-1.png" alt="Many of the most frequent trigrams feature censored words, but there is a difference in how often they are used within each class" width="672" />
<p class="caption">(\#fig:trigram25)Many of the most frequent trigrams feature censored words, but there is a difference in how often they are used within each class</p>
</div>

There is a difference in these proportions across classes. Tokens like "on xx xx" and "of xx xx" are used when referencing a date, e.g., "we had a problem on 06/25/2018".
Remember that the current tokenization engine strips punctuation before tokenizing. 
This means that the above example will be turned into "we had a problem on 06 25 2018" before creating n-grams^[The censored trigrams that include "oh" seem mysterious but upon closer examination, they come from censored addresses, with "oh" representing the US state of Ohio. Most two-letter state abbreviations are censored but this one is not, since it is ambiguous. This highlights the real challenge of anonymizing text.].

To crudely simulate what the data might look like before it was censored, we can replace all cases of "XX" and "XXXX" with random integers. 
This isn't quite right since dates will be given values between `00` and `99` and we don't know for sure that only numerals have been censored, but it gives us a place to start.
Below is a simple function `uncensor_vec()` that locates all instances of `"XX"` and replaces them with a number between 11 and 99.
We don't need to handle the special case of `XXXX` as it automatically being handled.


```r
uncensor <- function(n) {
  as.character(sample(seq(10 ^ (n - 1), 10 ^ n - 1), 1))
}

uncensor_vec <- function(x) {
  locs <- str_locate_all(x, "XX")
  map2_chr(x, locs, ~ {
    for (i in seq_len(nrow(.y))) {
      str_sub(.x, .y[i, 1], .y[i, 2]) <- uncensor(2)
    }
    .x
  })
}
```

We can run a quick test to see how it works.


```r
uncensor_vec("In XX/XX/XXXX I leased a XXXX vehicle")
```

```
#> [1] "In 33/64/4458 I leased a 7595 vehicle"
```

Now we can produce the same visualization as Figure \@ref(fig:censoredtrigram) but also applying our uncensoring function to the text before tokenizing.


```r
complaints_train %>%
  slice(1:1000) %>%
  mutate(text = uncensor_vec(consumer_complaint_narrative)) %>%
  unnest_tokens(trigrams, text, token = "ngrams",
                collapse = NULL) %>%
  count(trigrams, sort = TRUE) %>%
  mutate(censored = str_detect(trigrams, "xx")) %>%
  slice(1:20) %>%
  ggplot(aes(n, reorder(trigrams, n), fill = censored)) +
  geom_col() +
  scale_fill_manual(values = c("grey40", "firebrick")) +
  labs(y = "Trigrams", x = "Count")
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/uncensoredtrigram-1.png" alt="Trigrams without numbers float to the top as the uncensored tokens are too spread out" width="672" />
<p class="caption">(\#fig:uncensoredtrigram)Trigrams without numbers float to the top as the uncensored tokens are too spread out</p>
</div>

Here in Figure \@ref(fig:uncensoredtrigram), we see the same trigrams that appeared in Figure \@ref(fig:censoredtrigram).
However, none of the uncensored words appear, because of our uncensoring function.
This is expected, because while `"xx xx 2019"` appears in the first plot indicating a date in the year 2019, after we uncensor it, it is split into 365 buckets (actually more, since we used numerical values between `00` and `99`).
Censoring the dates in these complaints gives more power to a date as a general construct.

<div class="rmdwarning">
<p>What happens when we use these censored dates as a feature in supervised machine learning? We have a higher chance of understanding if dates in the complaint text are important to predicting the class, but we are blinded to the possibility that certain dates and months are more important.</p>
</div>

Data censoring can be a form of preprocessing in your data pipeline.
For example, it is highly unlikely to be useful (or ethical/legal) to have any specific person's social security number, credit card number, or any other kind of PII embedded into your model. Such values appear rarely and are most likely highly correlated with other known variables in your data set.
More importantly, that information can become embedded in your model and begin to leak as demonstrated by @carlini2018secret, @Fredrikson2014, and @Fredrikson2015.
Both of these issues are important, and one of them could land you in a lot of legal trouble. 
Exposing such PII to modeling is an example of where we should all stop to ask, "Should we even be doing this?" as we discussed in the foreword to these chapters.

If you have social security numbers in text data, you should definitely not pass them on to your machine learning model, but you may consider the option of annotating the _presence_ of a social security number. 
Since a social security number has a very specific form, we can easily construct a regular expression (Appendix \@ref(regexp)) to locate them.

<div class="rmdnote">
<p>A social security number comes in the form <code>AAA-BB-CCCC</code> where <code>AAA</code> is a number between <code>001</code> and <code>899</code> excluding <code>666</code>, <code>BB</code> is a number between <code>01</code> and <code>99</code> and <code>CCCC</code> is a number between <code>0001</code> and <code>9999</code>. This gives us the following regex:</p>
<p><code>(?!000|666)[0-8][0-9]{2}-(?!00)[0-9]{2}-(?!0000)[0-9]{4}</code></p>
</div>

We can use a function to replace each social security number with an indicator that can be detected later by preprocessing steps. 
It's a good idea to use a "word" that won't be accidentally broken up by a tokenizer.


```r
ssn_text <- c("My social security number is 498-08-6333",
              "No way, mine is 362-60-9159",
              "My parents numbers are 575-32-6985 and 576-36-5202")

ssn_pattern <-  "(?!000|666)[0-8][0-9]{2}-(?!00)[0-9]{2}-(?!0000)[0-9]{4}"

str_replace_all(string = ssn_text,
                pattern = ssn_pattern,
                replacement = "ssnindicator")
```

```
#> [1] "My social security number is ssnindicator"           
#> [2] "No way, mine is ssnindicator"                        
#> [3] "My parents numbers are ssnindicator and ssnindicator"
```

This technique isn't useful only for personally identifiable information but can be used anytime you want to gather similar words in the same bucket; hashtags, email addresses, and usernames can sometimes benefit from being annotated in this way.


\BeginKnitrBlock{rmdwarning}<div class="rmdwarning">The practice of data re-identification or de-anonymization, where seemingly or partially "anonymized" data sets are mined to identify individuals, is out of scope for this section and our book. However, this is a significant and important issue for any data practitioner dealing with PII and we encourage readers to familiarize themselves with results such as @Sweeney2000, and current best practices to protect against such mining.</div>\EndKnitrBlock{rmdwarning}


## Case study: custom features {#customfeatures}

Most of what we have looked at so far has boiled down to counting tokens and weighting them in one way or another.
This approach is quite broad and domain agnostic, but you as a data practitioner often have specific knowledge about your data set that you should use in feature engineering.
Your domain knowledge allows you to build more predictive features than the naive search of simple tokens.
As long as you can reasonably formulate what you are trying to count, chances are you can write a function that can detect it.
This is where having a little bit of knowledge about regular expressions pays off.

\BeginKnitrBlock{rmdpackage}<div class="rmdpackage">The **textfeatures** [@R-textfeatures] package includes functions to extract useful features from text, from the number of digits to the number of second person pronouns and more. These features can be used in textrecipes data preprocessing with the `step_textfeature()` function.</div>\EndKnitrBlock{rmdpackage}

Your specific domain knowledge may provide specific guidance about feature engineering for text.
Such custom features can be simple such as the number of URLs or the number of punctuation marks.
They can also be more engineered such as the percentage of capitalization, whether the text ends with a hashtag, or whether two people's names are both mentioned in a document.

For our CFPB complaints data, certain patterns may not have adequately been picked up by our model so far, such as the data censoring and the curly bracket annotation for monetary amounts that we saw in Section \@ref(classfirstattemptlookatdata). Let's walk through how to create data preprocessing functions to build the features to:

- detect credit cards,
- calculate percentage censoring, and
- detect monetary amounts.

### Detect credit cards

A credit card number is represented as four groups of four capital Xs in this data set.
Since the data is fairly well processed we are fairly sure that spacing will not be an issue and all credit cards will be represented as "XXXX XXXX XXXX XXXX". 
A first naive attempt may be to use `str_detect()` with "XXXX XXXX XXXX XXXX" to find all the credit cards.

<div class="rmdnote">
<p>It is a good idea to create a small example regular expression where you know the answer, and then prototype your function before moving to the main data set.</p>
</div>

We start by creating a vector with two positives, one negative, and one potential false positive.
The last string is more tricky since it has the same shape as a credit card but has one too many groups.


```r
credit_cards <- c("my XXXX XXXX XXXX XXXX balance, and XXXX XXXX XXXX XXXX.",
                  "card with number XXXX XXXX XXXX XXXX.",
                  "at XX/XX 2019 my first",
                  "live at XXXX XXXX XXXX XXXX XXXX SC")


str_detect(credit_cards, "XXXX XXXX XXXX XXXX")
```

```
#> [1]  TRUE  TRUE FALSE  TRUE
```

As we feared, the last vector was falsely detected to be a credit card.
Sometimes you will have to accept a certain number of false positives and/or false negatives, depending on the data and what you are trying to detect. 
In this case, we can make the regex a little more complicated to avoid that specific false positive.
We need to make sure that the word coming before the X's doesn't end in a capital X and the word following the last X doesn't start with a capital X.
We place spaces around the credit card and use some negated character classes (Appendix \@ref(character-classes)) to detect anything BUT a capital X.


```r
str_detect(credit_cards, "[^X] XXXX XXXX XXXX XXXX [^X]")
```

```
#> [1]  TRUE FALSE FALSE FALSE
```

Hurray! This fixed the false positive. 
But it gave us a false negative in return.
Turns out that this regex doesn't allow the credit card to be followed by a period since it requires a space.
We can fix this with an alteration to match for a period or a space and a non-X.


```r
str_detect(credit_cards, "[^X] +XXXX XXXX XXXX XXXX(\\.| [^X])")
```

```
#> [1]  TRUE  TRUE FALSE FALSE
```

Now that we have a regular expression we are happy with we can wrap it up in a function we can use.
We can extract the presence of a credit card with `str_detect()` and the number of credit cards with `str_count()`.


```r
creditcard_indicator <- function(x) {
  str_detect(x, "[^X] +XXXX XXXX XXXX XXXX(\\.| [^X])")
}

creditcard_count <- function(x) {
  str_count(x, "[^X] +XXXX XXXX XXXX XXXX(\\.| [^X])")
}

creditcard_indicator(credit_cards)
```

```
#> [1]  TRUE  TRUE FALSE FALSE
```

```r
creditcard_count(credit_cards)
```

```
#> [1] 2 1 0 0
```

### Calculate percentage censoring

Some of the complaints contain a high proportion of censoring, and we can build a feature to measure the percentage of the text that is censored.

<div class="rmdwarning">
<p>There are often many ways to get to the same solution when working with regular expressions.</p>
</div>

Let's attack this problem by counting the number of X's in each string, then count the number of alphanumeric characters and divide the two to get a percentage.


```r
str_count(credit_cards, "X")
```

```
#> [1] 32 16  4 20
```

```r
str_count(credit_cards, "[:alnum:]")
```

```
#> [1] 44 30 17 28
```

```r
str_count(credit_cards, "X") / str_count(credit_cards, "[:alnum:]")
```

```
#> [1] 0.7272727 0.5333333 0.2352941 0.7142857
```

We can finish up by creating a function.


```r
percent_censoring <- function(x) {
  str_count(x, "X") / str_count(x, "[:alnum:]")
}

percent_censoring(credit_cards)
```

```
#> [1] 0.7272727 0.5333333 0.2352941 0.7142857
```

### Detect monetary amounts

We have already constructed a regular expression that detects the monetary amount from the text in Section \@ref(classfirstattemptlookatdata), so now we can look at how to use this information.
Let's start by creating a little example and see what we can extract.


```r
dollar_texts <- c("That will be {$20.00}",
                  "{$3.00}, {$2.00} and {$7.00}",
                  "I have no money")

str_extract_all(dollar_texts, "\\{\\$[0-9\\.]*\\}")
```

```
#> [[1]]
#> [1] "{$20.00}"
#> 
#> [[2]]
#> [1] "{$3.00}" "{$2.00}" "{$7.00}"
#> 
#> [[3]]
#> character(0)
```

We can create a function that simply detects the dollar amount, and we can count the number of times each amount appears.
Each occurrence also has a value, so it would be nice to include that information as well, such as the mean, minimum, or maximum.

First, let's extract the number from the strings. We could write a regular expression for this, but the `parse_number()` function from the readr package does a really good job of pulling out numbers.


```r
str_extract_all(dollar_texts, "\\{\\$[0-9\\.]*\\}") %>%
  map(readr::parse_number)
```

```
#> [[1]]
#> [1] 20
#> 
#> [[2]]
#> [1] 3 2 7
#> 
#> [[3]]
#> numeric(0)
```

Now that we have the numbers we can iterate over them with the function of our choice.
Since we are going to have texts with no monetary amounts, we need to handle the case with zero numbers. Defaults for some functions with vectors of length zero can be undesirable; we don't want `-Inf` to be a value. Let's extract the maximum value and give cases with no monetary amounts a maximum of zero.


```r
max_money <- function(x) {
  str_extract_all(x, "\\{\\$[0-9\\.]*\\}") %>%
    map(readr::parse_number) %>%
    map_dbl(~ ifelse(length(.x) == 0, 0, max(.x)))
}

max_money(dollar_texts)
```

```
#> [1] 20  7  0
```

Now that we have created some feature engineering functions, we can use them to (hopefully) make our classification model better.


## What evaluation metrics are appropriate?

We have focused on using accuracy and ROC AUC as metrics for our classification models so far. These are not the only classification metrics available and your choice will often depend on how much you care about false positives compared to false negatives.

If you know before you fit your model that you want to compute one or more metrics, you can specify them in a call to `metric_set()`. Let's set up a tuning grid for two new classification metrics, `recall` and `precision`, that focus not on the overall proportion of observations that are predicted correctly but instead on false positives and false negatives.


```r
nb_rs <- fit_resamples(
  nb_wf,
  complaints_folds,
  metrics = metric_set(recall, precision)
)
```

If you have already fit your model, you can still compute and explore non-default metrics as long as you saved the predictions for your resampled data sets using `control_resamples(save_pred = TRUE)`. 

Let's go back to the naive Bayes model we tuned in Section \@ref(classfirstmodel), with predictions stored in `nb_rs_predictions`. We can compute the overall recall.


```r
nb_rs_predictions %>%
  recall(product, .pred_class)
```

```
#> # A tibble: 1 x 3
#>   .metric .estimator .estimate
#>   <chr>   <chr>          <dbl>
#> 1 recall  binary         0.722
```

We can also compute the recall for each resample using `group_by()`.


```r
nb_rs_predictions %>%
  group_by(id) %>%
  recall(product, .pred_class)
```

```
#> # A tibble: 10 x 4
#>    id     .metric .estimator .estimate
#>    <chr>  <chr>   <chr>          <dbl>
#>  1 Fold01 recall  binary         0.791
#>  2 Fold02 recall  binary         0.690
#>  3 Fold03 recall  binary         0.674
#>  4 Fold04 recall  binary         0.8  
#>  5 Fold05 recall  binary         0.719
#>  6 Fold06 recall  binary         0.735
#>  7 Fold07 recall  binary         0.713
#>  8 Fold08 recall  binary         0.655
#>  9 Fold09 recall  binary         0.717
#> 10 Fold10 recall  binary         0.725
```

Many of the metrics used for classification are functions of the true positive, true negative, false positive, and false negative rates. 
The confusion matrix, a contingency table of observed classes and predicted classes, gives us information on these rates directly.


```r
conf_mat_resampled(nb_rs)
```

```
#>        Credit  Other
#> Credit 3009.5 1157.4
#> Other   549.1 4075.1
```

It is possible with many data sets to achieve high accuracy just by predicting the majority class all the time, but such a model is not useful in the real world. Accuracy alone is often not a good way to assess the performance of classification models.

<div class="rmdnote">
<p>For the full set of classification metric options, see the <a href="https://yardstick.tidymodels.org/reference/">yardstick documentation</a>.</p>
</div>


## The full game: classification {#mlclassificationfull}

We have come a long way from our first classification model in Section \@ref(classfirstmodel) and it is time to see how we can use what we have learned to improve it.
We started this chapter with a simple naive Bayes model and token counts.
Since then have we looked at different models, preprocessing techniques, and domain-specific feature engineering.
For our final model, let's use some of the domain-specific features we developed in Section \@ref(customfeatures) along with our lasso regularized classification model and tune both the regularization penalty as well as the number of tokens to include. For this final model we will:

- train on the same set of cross-validation resamples used throughout this chapter,
- include text (but not `tags` or date features, since those did not result in better performance),
- tune the number of tokens used in the model,
- include unigrams only,
- include custom-engineered features,
- finally evaluate on the testing set, which we have not touched at all yet.

### Feature selection

We start by creating a new preprocessing recipe, using only the text of the complaints for feature engineering. 


```r
complaints_rec_v2 <-
  recipe(product ~ consumer_complaint_narrative, data = complaints_train)
```

After exploring this text data more in Section \@ref(customfeatures), we want to add these custom features to our final model.
To do this, we use `step_textfeature()` to compute custom text features. 
We create a list of the custom text features and pass this list to `step_textfeature()` via the `extract_functions` argument. 
Note how we have to take a copy of `consumer_complaint_narrative` using `step_mutate()` as `step_textfeature()` consumes the column.


```r
extract_funs <- list(creditcard_count = creditcard_count,
                     percent_censoring = percent_censoring,
                     max_money = max_money)

complaints_rec_v2 <- complaints_rec_v2 %>%
  step_mutate(narrative_copy = consumer_complaint_narrative) %>%
  step_textfeature(narrative_copy, extract_functions = extract_funs)
```

The tokenization will be similar to the other models in this chapter.
In our original model, we only included 1000 tokens; for our final model, let's treat the number of tokens as a hyperparameter that we vary when we tune the final model.
Let's also set the `min_times` argument to 100, to throw away tokens that appear less than 100 times in the entire corpus.
We want our model to be robust and a token needs to appear enough times before we include it.

<div class="rmdnote">
<p>This data set has many more than 100 of even the most common 5000 or more tokens, but it can still be good practice to specify <code>min_times</code> to be safe. Your choice for <code>min_times</code> should depend on your data and how robust you need your model to be.</p>
</div>


```r
complaints_rec_v2 <- complaints_rec_v2 %>%
  step_tokenize(consumer_complaint_narrative) %>%
  step_tokenfilter(consumer_complaint_narrative,
                   max_tokens = tune(), min_times = 100) %>%
  step_tfidf(consumer_complaint_narrative)
```

### Specify the model

We use a lasso regularized classifier since it performed well throughout this chapter. We can reuse parts of the old workflow `sparse_wf` from Section \@ref(casestudysparseencoding) and update the recipe specification.


```r
sparse_wf_v2 <- sparse_wf %>%
  update_recipe(complaints_rec_v2, blueprint = sparse_bp)

sparse_wf_v2
```

```
#> ══ Workflow ════════════════════════════════════════════════════════════════════
#> Preprocessor: Recipe
#> Model: logistic_reg()
#> 
#> ── Preprocessor ────────────────────────────────────────────────────────────────
#> 5 Recipe Steps
#> 
#> ● step_mutate()
#> ● step_textfeature()
#> ● step_tokenize()
#> ● step_tokenfilter()
#> ● step_tfidf()
#> 
#> ── Model ───────────────────────────────────────────────────────────────────────
#> Logistic Regression Model Specification (classification)
#> 
#> Main Arguments:
#>   penalty = tune()
#>   mixture = 1
#> 
#> Computational engine: glmnet
```

Before we tune the model, we need to set up a set of possible parameter values to try. 

<div class="rmdwarning">
<p>There are <em>two</em> tunable parameters in this model, the regularization parameter and the maximum number of tokens included in the model.</p>
</div>

Let's include different possible values for each parameter, for a combination of 60 models.


```r
final_grid <- grid_regular(
  penalty(range = c(-4, 0)),
  max_tokens(range = c(1e3, 3e3)),
  levels = c(penalty = 20, max_tokens = 3)
)

final_grid
```

```
#> # A tibble: 60 x 2
#>     penalty max_tokens
#>       <dbl>      <int>
#>  1 0.0001         1000
#>  2 0.000162       1000
#>  3 0.000264       1000
#>  4 0.000428       1000
#>  5 0.000695       1000
#>  6 0.00113        1000
#>  7 0.00183        1000
#>  8 0.00298        1000
#>  9 0.00483        1000
#> 10 0.00785        1000
#> # … with 50 more rows
```

<div class="rmdpackage">
<p>We used <code>grid_regular()</code> here where we fit a model at every combination of parameters, but if you have a model with many tuning parameters, you may wish to try a space-filling grid instead, such as <code>grid_max_entropy()</code> or <code>grid_latin_hypercube()</code>. The <strong>tidymodels</strong> package for creating and handling tuning parameters and parameter grids is <strong>dials</strong>.</p>
</div>

Now it's time to set up our tuning grid. Let's save the predictions so we can explore them in more detail, and let's also set custom metrics instead of using the defaults. Let's compute accuracy, sensitivity, and specificity during tuning. Sensitivity and specificity are closely related to recall and precision.


```r
set.seed(2020)
tune_rs <- tune_grid(
  sparse_wf_v2,
  complaints_folds,
  grid = final_grid,
  metrics = metric_set(accuracy, sensitivity, specificity)
)
```

We have fitted these classification models!

### Evaluate the modeling {#classification-final-evaluation}

Now that all of the models with possible parameter values have been trained, we can compare their performance. Figure \@ref(fig:complaintsfinaltunevis) shows us the relationship between performance (as measured by the metrics we chose), the number of tokens, and regularization. 


```r
autoplot(tune_rs) +
  labs(
    color = "Number of tokens",
    title = "Model performance across regularization penalties and tokens",
    subtitle = paste("We can choose a simpler model with higher regularization")
  )
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/complaintsfinaltunevis-1.png" alt="Model performance is similar for the higher token options so we can choose a simpler model. Note the logarithmic scale on the x-axis for the regularization penalty." width="672" />
<p class="caption">(\#fig:complaintsfinaltunevis)Model performance is similar for the higher token options so we can choose a simpler model. Note the logarithmic scale on the x-axis for the regularization penalty.</p>
</div>

Since this is our final version of this model, we want to choose final parameters and update our model object so we can use it with new data. We have several options for choosing our final parameters, such as selecting the numerically best model. Instead, let's choose a simpler model within some limit around that numerically best result, with more regularization that gives close-to-best performance. Let's choose by percent loss compared to the best model (the default choice is 2% loss), and let's say we care most about overall accuracy (rather than sensitivity or specificity).


```r
choose_acc <- tune_rs %>%
  select_by_pct_loss(metric = "accuracy", -penalty)

choose_acc
```

```
#> # A tibble: 1 x 10
#>   penalty max_tokens .metric  .estimator  mean     n std_err .config .best .loss
#>     <dbl>      <int> <chr>    <chr>      <dbl> <int>   <dbl> <chr>   <dbl> <dbl>
#> 1 0.00483       1000 accuracy binary     0.882    10 9.44e-4 Prepro… 0.898  1.78
```

After we have those parameters, `penalty` and `max_tokens`, we can finalize our earlier tunable workflow, by updating it with this value.


```r
final_wf <- finalize_workflow(sparse_wf_v2, choose_acc)
final_wf
```

```
#> ══ Workflow ════════════════════════════════════════════════════════════════════
#> Preprocessor: Recipe
#> Model: logistic_reg()
#> 
#> ── Preprocessor ────────────────────────────────────────────────────────────────
#> 5 Recipe Steps
#> 
#> ● step_mutate()
#> ● step_textfeature()
#> ● step_tokenize()
#> ● step_tokenfilter()
#> ● step_tfidf()
#> 
#> ── Model ───────────────────────────────────────────────────────────────────────
#> Logistic Regression Model Specification (classification)
#> 
#> Main Arguments:
#>   penalty = 0.00483293023857175
#>   mixture = 1
#> 
#> Computational engine: glmnet
```

The `final_wf` workflow now has finalized values for `max_tokens` and `penalty`.

We can now fit this finalized workflow on training data and _finally_ return to our testing data. 

<div class="rmdwarning">
<p>Notice that this is the first time we have used our testing data during this entire chapter; we tuned and compared models using resampled data sets instead of touching the testing set.</p>
</div>

We can use the function `last_fit()` to **fit** our model one last time on our training data and **evaluate** it on our testing data. We only have to pass this function our finalized model/workflow and our data split.


```r
final_fitted <- last_fit(final_wf, complaints_split)

collect_metrics(final_fitted)
```

```
#> # A tibble: 2 x 4
#>   .metric  .estimator .estimate .config             
#>   <chr>    <chr>          <dbl> <chr>               
#> 1 accuracy binary         0.885 Preprocessor1_Model1
#> 2 roc_auc  binary         0.949 Preprocessor1_Model1
```

The metrics for the test set look about the same as the resampled training data and indicate we did not overfit during tuning. The accuracy of our final model has improved compared to our earlier models, both because we are combining multiple preprocessing steps and because we have tuned the number of tokens.

The confusion matrix on the testing data in Figure \@ref(fig:finalheatmap) also yields pleasing results. It appears symmetric with a strong presence on the diagonal, showing that there isn't any strong bias towards either of the classes. 


```r
collect_predictions(final_fitted) %>%
  conf_mat(truth = product, estimate = .pred_class) %>%
  autoplot(type = "heatmap")
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/finalheatmap-1.png" alt="Confusion matrix on the test set for final lasso regularized classifier" width="672" />
<p class="caption">(\#fig:finalheatmap)Confusion matrix on the test set for final lasso regularized classifier</p>
</div>

Figure \@ref(fig:finalroccurve) shows the ROC curve for testing set, to demonstrate how well this final classification model can distinguish between the two classes.


```r
collect_predictions(final_fitted)  %>%
  roc_curve(truth = product, .pred_Credit) %>%
  autoplot() +
  labs(
    color = NULL,
    title = "ROC curve for US Consumer Finance Complaints",
    subtitle = "With final tuned lasso regularized classifier on the test set"
  )
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/finalroccurve-1.png" alt="ROC curve with the test set for final lasso regularized classifier" width="672" />
<p class="caption">(\#fig:finalroccurve)ROC curve with the test set for final lasso regularized classifier</p>
</div>


The output of `last_fit()` also contains a fitted model (a `workflow`, to be more specific), that has been trained on the _training_ data. We can use the vip package to understand what the most important variables are in the predictions, shown in Figure \@ref(fig:complaintsvip).


```r
library(vip)

complaints_imp <- pull_workflow_fit(final_fitted$.workflow[[1]]) %>%
  vi(lambda = choose_acc$penalty)

complaints_imp %>%
  mutate(
    Sign = case_when(Sign == "POS" ~ "Less about credit reporting",
                     Sign == "NEG" ~ "More about credit reporting"),
    Importance = abs(Importance),
    Variable = str_remove_all(Variable, "tfidf_consumer_complaint_narrative_"),
    Variable = str_remove_all(Variable, "textfeature_narrative_copy_")
  ) %>%
  group_by(Sign) %>%
  top_n(20, Importance) %>%
  ungroup %>%
  ggplot(aes(x = Importance,
             y = fct_reorder(Variable, Importance),
             fill = Sign)) +
  geom_col(show.legend = FALSE) +
  scale_x_continuous(expand = c(0, 0)) +
  facet_wrap(~Sign, scales = "free") +
  labs(
    y = NULL,
    title = "Variable importance for predicting the topic of a CFPB complaint",
    subtitle = paste("These features are the most important in predicting",
                     "whether a complaint is about credit or not")
  )
```

<div class="figure" style="text-align: center">
<img src="07_ml_classification_files/figure-html/complaintsvip-1.png" alt="Some words increase a CFPB complaint's probability of being about credit reporting while some decrease that probability" width="672" />
<p class="caption">(\#fig:complaintsvip)Some words increase a CFPB complaint's probability of being about credit reporting while some decrease that probability</p>
</div>

Tokens like "interest", "bank", and "escrow" contribute in this model away from a classification as about credit reporting, while tokens like the names of the credit reporting agencies, "reporting", and "report" and  contribute in this model _toward_ classification as about credit reporting.

<div class="rmdnote">
<p>The top features we see here are all tokens learned directly from the text. None of our hand-crafted custom features, like <code>percent_censoring</code> or <code>max_money</code> are top features in terms of variable importance. In many cases, it can be difficult to create features from text that perform better than the tokens themselves.</p>
</div>

We can gain some final insight into our model by looking at observations from the test set that it *misclassified*. Let's bind together the predictions on the test set with the original `complaints_test` data. Then let's look at complaints that were labeled as about credit reporting in the original data but that our final model thought had a low probability of being about credit reporting.


```r
complaints_bind <- collect_predictions(final_fitted) %>%
  bind_cols(complaints_test %>% select(-product))

complaints_bind %>%
  filter(product == "Credit", .pred_Credit < 0.2) %>%
  select(consumer_complaint_narrative) %>%
  slice_sample(n = 10)
```

```
#> # A tibble: 10 x 1
#>    consumer_complaint_narrative                                                 
#>    <chr>                                                                        
#>  1 "Spoke with Mr XXXX today at XXXX. Inquired about medical bill and date. Inf…
#>  2 "I opened a credit card account with GE Financial to finance an air conditio…
#>  3 "I lost my debit card and had to use my checks until I received my new card.…
#>  4 "Loan was for {$2500.00} balance is showing {$4200.00} because they included…
#>  5 "Chase XXXX Reward card was activated in my name without my consent. Card # …
#>  6 "I have had this service for more the seven years, the more I use them the m…
#>  7 "Ive had severe issues with the student loan process for at least 10 years. …
#>  8 "Chase Card Address : XXXX XXXX XXXX City/ State/ Zip : XXXX, DE XXXX Date :…
#>  9 "I received a notice that stated that I was currently in debt in the amount …
#> 10 "Hi About 2 months ago ( XXXX ) I received an email from a company represent…
```

We can see why some of these would be difficult for our model to classify as about credit reporting, since some are about other topics as well. The original label may also be incorrect in some cases.

What about misclassifications in the other direction, observations in the test set that were *not* labeled as about credit reporting but that our final model gave a high probability of being about credit reporting?


```r
complaints_bind %>%
  filter(product == "Other", .pred_Credit > 0.8) %>%
  select(consumer_complaint_narrative) %>%
  slice_sample(n = 10)
```

```
#> # A tibble: 10 x 1
#>    consumer_complaint_narrative                                                 
#>    <chr>                                                                        
#>  1 "Paid in full collections to CBE Group amount of {$360.00} paid in XXXX of 2…
#>  2 "Back in 2013, my purse was stolen containing all of my personal belongings.…
#>  3 "I Have contacted Credit Bureaus on numerous occasions to have incorrect or …
#>  4 "I contacted the company, I reported to police department, sent in police re…
#>  5 "FCRA states information reporting has to be 100 % verifiable and 100 % accu…
#>  6 "I have attempted on numerous times to dispute an account that has ERRORS. E…
#>  7 "XXXX  OF XXXX FLORIDA IS TAKING ADVANTAGE OF THEIR ABILITY TO REPORT TO THE…
#>  8 "Late payment reported ( {$16.00} ) to my credit report that caused my credi…
#>  9 "MIDLAND FUNDING XXXX as of XX/XX/2019 reporting for identity fraud, was rep…
#> 10 "I have contacted this company through the credit bureaus multiple times to …
```

Again, these are "mistakes" on the part of the model that we can understand based on the content of these complaints. The original labeling on the complaints looks to be not entirely correct or consistent, typical of real data from the real world.

## Summary {#mlclassificationsummary}

You can use classification modeling to predict labels or categorical variables from a data set, including data sets that include text.
Naive Bayes models can perform well with text data since each feature is handled independently and thus large numbers of features are computational feasible.
This is important as bag-of-word text models can involve thousands of tokens.
We also saw that regularized linear models, such as lasso, often work well for text data sets.
Your own domain knowledge about your text data is valuable, and using that knowledge in careful engineering of custom features can improve your model in some cases.

### In this chapter, you learned:

- how text data can be used in a classification model
- to tune hyperparameters of a model
- how to compare different model types
- that models can combine both text and non-text predictors
- about engineering custom features for machine learning
- about performance metrics for classification models

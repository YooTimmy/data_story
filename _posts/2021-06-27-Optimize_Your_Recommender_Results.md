---
title:  "How to Optimize Recommendation Results with Genetic Algorithm"
date:   2021-06-27 12:39:59 +0800
categories:
  - Recommendation
  - Optimization
tags:
  - ALS
  - Pyspark
  - DEAP
  - Multi-Objective Optimization
toc: true
toc_sticky: true
---
## 1. Simple Movie Recommender with ALS

![Collaborative_filtering](https://upload.wikimedia.org/wikipedia/commons/5/52/Collaborative_filtering.gif)
Recommender systems have been applied in various industries nowadays, including e-commerce, marketing, video streaming, financial industries and so on. There are different types of algorithms out there,
including collaborative filtering, content-based filtering, and reinforcement learning based recommender. However, sometimes the implementation of recommender algorithm is only a starting point -
there are always requirements to evaluate and further optimize the results based on business needs. In this post, we will be using a small subset of the classic dataset for recommendation study - [movielens dataset](https://grouplens.org/datasets/movielens/),
to demonstrate how to use genetic algorithm to further optimize the recommendation results.

In terms of the recommendation algorithm, we will use the widely used collaborative filtering method - ALS (Alternative Least Squares), which is provided by Spark MLlib. This approach is especially preferred when dealing with large datasets, although in our case
study we are only using a small dataset for illustration purpose. The sample code of a basic ALS based recommender is as follows:

```python
spark = SparkSession.builder.appName('Recommender').getOrCreate()
data = spark.read.csv('movielens_ratings.csv',inferSchema=True,header=True)
# To use a 80-20 train/test split
(training, test) = data.randomSplit([0.8, 0.2],seed=42)
# Using the ALS algorithm provided by pyspark
als = ALS(maxIter=5, regParam=0.01, userCol="userId", itemCol="movieId", ratingCol="rating", seed=42)
model = als.fit(training)
# Make predictions on the test dataset, which would be used to evaluate the model performances later.
predictions = model.transform(test)
```

With just a few lines of code, we have a simple movie recommender model established. The next question is, how do we evaluate the performance of the recommender?
The answer for this question really depends on how to frame the problem, as well as the business context behind this model.
For instance, if we are just building the recommender for learning purpose, then we can simply evaluate recommender output with some built-in regression evaluator function as follows:

```python
from pyspark.ml.evaluation import RegressionEvaluator
evaluator = RegressionEvaluator(metricName="rmse", labelCol="rating",predictionCol="prediction")
prediction_rmse = evaluator.evaluate(predictions)
print("The RMSE score for recommender is : " + str(prediction_rmse))
```

The RMSE value we obtained is ~1.6, which is quite high considering the ratings from customers are ranging from 1-5. Nonetheless, it is still expected given we are only using a very small dataset for training.

## 2. Considering Business Objectives for Evaluations

In certain use cases, the final recommendations given to the customers might not be directly based on the output from the ALS algorithm.
After all, when applying the recommender to production, the more relevant KPIs for business would be the actual revenue lift from the model,
instead of only looking at the modelling KPIs such as RMSE scores.

Now imagine a online streaming start-up is planning to deploy our movie recommender. Before doing that, business team wants to know what is revenue lift from the model to make justifications to business.
To make this calculation, we need to know the revenue associated with each individual movie. For instance, we assume some older movies require a lower acquisition cost, whereas the latest movies have higher costs.
As a result, the net revenue associated with each individual movie also varies.

Now with the revenue part of the overall consideration, we need to consider both objectives in the final recommendations: the likelihood of customers purchasing the movie, which is mainly decided by the recommender,
as well as the total revenue to the business. In other words, it is a multi-objective optimization problem we are looking at right now.

With these in mind, now we need to re-think how to evaluate the recommender. Though customer's potential ratings for a movie can be considered as a continuous variable from 0 to 5,
in the end whether they will purchase a particular movie or not is still a binary selection - either "yes" or "no". For simplicity, let's assume as long as the customer gives a rating larger than 2, then he/she would purchase the movie for viewing.
The code sample to transform the ratings dataframe is as follows:

```python
df_val2 = predictions.groupby('userId').pivot('movieId').max('rating')
df_val2 = df_val2.withColumn("userId",df_val2['userId'].cast("string"))
#Convert to pandas df for further manipulations
dfp_actual = df_val2.toPandas()
dfp_actual[dfp_actual.iloc[:,1:] < 2] = 0
dfp_actual[dfp_actual.iloc[:,1:] > 0] = 1
```

Now we have adjusted the actual ratings from customers into binary values. In terms of the predicted rating generated by recommender, usually what we can do is for each customers,
rank the predicted scores for different movies from high to low, and then choose the top K movies as the final recommendation to that particular customer. The code block to implement this is as follows:

```python
#Pivoting the predictions into user-movie matrix
df_val = predictions.groupby('userId').pivot('movieId').max('prediction')
df_val = df_val.withColumn("userId",df_val['userId'].cast("string"))
dfp_pred = df_val.toPandas()
dfp_pred.set_index('userId',inplace = True)
rank_df = dfp_pred.rank(1, ascending = False, method = 'first')
rank_df[rank_df > K] = 0
rank_df[rank_df > 0] = 1
```

With the above data manipulations done, we can now evaluate the recommendation performance with some typical classification KPIs, such as precision, recall and F1 scores.
Regarding to the revenue associated with each product, we will just randomly generate numbers for illustration purpose.

## 3. GA Multi-Objective Optimization with DEAP

Similar to the recommender algorithms, there are various solutions for multi-objective optimizations as well. In this post, we are going to explore the usage of Genetic Algorithm(GA) for multi-objective optimization.

Unlike other optimization methods like gradient descend, GA does not assume any special properties for the target function, which gives more flexibility in usage.

The general work flow for GA is as follows: first of all, users need to initialize a population of potential candidates, and then evaluate the fitness for each individual (fitness can be considered as the objective function outputs).
After this, the best individuals are selected as the parents for the next generation. Subsequently, crossover and mutation will then take place in the algorithm, and fitness values would be calculated for selecting the parents for next generation.
This whole process will continue on until a satisfied solution is obtained.

[DEAP](https://deap.readthedocs.io/en/master/) is a python library that can be used to implement GA effectively. It provides necessary functions required in GA, including fitness functions creation, individual and populations generation, etc.
Although DEAP has done a good job to help users use GA for optimization effectively, it could be still a bit challenging in the beginning to get to understand and apply all the relevant functions and parameters.
Luckily, there are some [sample projects](https://github.com/lmarti/evolutionary-computation-course/blob/master/AEC.06%20-%20Evolutionary%20Multi-Objective%20Optimization.ipynb) available that we can use for reference purpose.

Essentially, what we are trying to achieve is to adjust the weight metrics for each individual movie, which will then be multiplied with the scores generated from ALS model to get the adjusted scores. With the adjusted scores,
the top 3 movies to be recommended to users would be updated accordingly. The GA will run through multiple iterations to adjust the weight metrics in order to maximize both objectives.

I would not go to much details on the specific functionality of DEAP library here, you can always refer to the [official website](https://deap.readthedocs.io/en/master/) for more information.
Just want to highlight the section where we need to define the "evaluate" function to be used for the GA optimization.
As observed from the [example notebook mentioned](https://github.com/lmarti/evolutionary-computation-course/blob/master/AEC.06%20-%20Evolutionary%20Multi-Objective%20Optimization.ipynb), the input for this evaluate function need to be the weight metrics,
whereas the output need to be a tuple of the two objective functions to be optimized, which are F1 score and total revenue in our case. The sample code is as follows:

```python
def evaluate_recommender(dfp_actual,dfp_pred,w_lst,K,rev):
    movie_ids = dfp_actual.columns.tolist()
    revenue = {k:i for k,i in zip(movie_ids, rev)}
    weights = {k:i for k,i in zip(movie_ids, w_lst)}
    weights_normalized = {i:weights[i]/sum(weights.values()) for i in movie_ids}
    dfp_actual = dfp_actual.astype('float')
    dfp_pred = dfp_pred.astype('float')
    for i in movie_ids:
        dfp_pred[i] = dfp_pred[i].apply(lambda x: x*weights_normalized[i])
    dfp_pred.fillna(0,inplace = True)
    #Consider the top K matchings
    rank_df = dfp_pred.rank(1, ascending = False, method = 'first')
    rank_df[rank_df > K] = 0
    rank_df[rank_df > 0] = 1
    val_df = rank_df * dfp_actual
    ##Here just to calculate some relevant eval metrics
    val_res = {"Predicted":rank_df.sum(axis = 0),
            "Actual": dfp_actual.sum(axis = 0),
            "Match": val_df.sum(axis = 0),
            "Precision": val_df.sum(axis = 0)/rank_df.sum(axis = 0),
            "Recall": val_df.sum(axis = 0)/rank_df.sum(axis = 0),
            "Weights": pd.Series(weights),
            "Weights_Normalized": pd.Series(weights_normalized),
            "Base_Revenue": pd.Series(revenue),
            "Total_Revenue": (pd.Series(revenue)*val_df.sum(axis = 0))
        }
    val_res = pd.DataFrame(val_res)
    val_res.fillna(0,inplace = True)
    recall = val_res.Match.sum()/val_res.Actual.sum()
    precision = val_res.Match.sum()/val_res.Predicted.sum()
    f1_score = 2*(precision * recall)/(precision + recall)
    total_rev = val_res.Total_Revenue.sum()
    return val_res, recall, precision, f1_score, total_rev

def evaluate(weights_lst):
    res_df,recall,precision,f1_score, total_rev = evaluate_recommender(dfp_actual,dfp_pred,weights_lst,3,rev)
    return (f1_score,total_rev)
```

## 4. Understanding the Optimization Results

While running optimizations with DEAP, we can observe from the logs that the total revenue and F1 score keep improving in the beginning of iterations and gradually get stabilized in later stages.
The final optimized F1 score and revenue is 0.67 and 75204, which corresponding to 35% and 46% lift comparing the scores before optimization. Looks like we get good improvements from the optimization!

![GA_Iterations](https://raw.githubusercontent.com/YooTimmy/data_story/gh-pages/assets/images/Recommender/GA_Iterations.png)

Just one side note here: when talking about multi-objective optimizations, in a lot of cases the objectives may have trade-offs between each other. For instance, while increasing the total revenue generated, we might see the decrease in F1 score.
However in our case, this type pf trade-off was not observed - both F1 score and total revenue are trending into the positive direction.

Why the objectives trade-off were not observed in our case study? I believe the main reason is that we are looking at a simple optimization with a very small dataset, thus the total rev and F1 scores optimization is actually not conflicting with each other.
Taking a look at the final recommendations outputs post weights optimization, most actual movie purchases, if not all are already covered by the top-3 recommendations from recommender.
In fact, the final optimized solution we obtained may be called as a dominated solution for a optimization problem, i.e. the optimized solution is better than all the rest intermediate solutions for both objectives. More details on this could be found in this
[article](https://communities.bentley.com/products/products_generativecomponents/w/generative_components_community_wiki/37421/b---non-dominated-solutions-and-the-pareto-frontier).

For the full code used for this post, you may find it from my [Github Repo](https://github.com/YooTimmy/Recommender_with_Optimization) here. Thanks for reading!



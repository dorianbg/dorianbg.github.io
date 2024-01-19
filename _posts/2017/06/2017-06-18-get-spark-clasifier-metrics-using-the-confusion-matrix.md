---
title: "Get Spark Clasifier metrics using the Confusion Matrix"
date: "2017-06-18"
categories: 
  - "data-engineering"
  - "data-science"
---

When using Apache Spark specifically for "binary" classification (ie. the labels are either 0 or 1) it is possible to use the Confusion Matrix for getting some metrics such as the number of true positives, false negatives or true negatives.  
  
To get that data you can use [MulticlassMetrics class](https://spark.apache.org/docs/1.6.3/api/java/index.html?org/apache/spark/mllib/evaluation/MulticlassMetrics.html) containing those useful metrics.  
  
**Please note that the code in this tutorial will focus on the RDD API.**  
  
An example of getting those metrics is this:  
  
**0. Create the training and testing dataset**  
-> I will skip this part,  
but assume the output are two  JavaRDD objects: training and test.  
  
**1. Train your model**   
  
```java
LogisticRegressionClassifier.model = new LogisticRegressionWithLBFGS()  
                .setNumClasses(2)  
                .run(training.rdd());  
```

where training is of type JavaRDD training. But I will not go into specifics of that now.  

  

**2. Test your model**

Then you can get the classifier metrics by using your testing data set (also of type JavaRDD)  

```java
JavaRDD<Tuple2> predictionAndLabels = test.map(new Function<LabeledPoint, Tuple2>() {  
        public Tuple2 call(LabeledPoint p) {  
            Double prediction = model.predict(p.features());  
            return new Tuple2(prediction, p.label());  
        }  
    };  
MulticlassMetrics metrics = new MulticlassMetrics(predictionAndLabels.rdd());  
```

**3. Profit** 

This part is actually easy if your metrics are covered by the _MulticlassMetrics_ API.  
For instance you could easily do:
```java
logger.info("Precision = " + metrics.precision());  
logger.info("Recall = " + metrics.recall());  
```

But I needed to get very specific measurement which could not be easily digged out with the API,  
specifically I needed exact numbers for True Positives, False Positives, False Negatives and True Negatives.  
  
You can get that data from the  
  
      metrics.confusionMatrix()  
  
which will be printed out with (in my specific case):  
  
     logger.info("n" + metrics.confusionMatrix());  
And the final output is:
```
41.0  30.0  
26.0  48.0  
```

But now the question is how are the metrics alligned, and that took me a good 30 minutes to figure out. The answer is that they are structured as this:  

```
            Predicted  
Actual     0      1  
    0      TN    FN  
    1      FP    TP  
```
  

So to get the actual values use this "one liner":  

```java
double[] confusionMatrix = metrics.confusionMatrix().toArray();  
double tn = confusionMatrix[0];  
double fp = confusionMatrix[1];  
double fn = confusionMatrix[2];  
double tp = confusionMatrix[3];
```

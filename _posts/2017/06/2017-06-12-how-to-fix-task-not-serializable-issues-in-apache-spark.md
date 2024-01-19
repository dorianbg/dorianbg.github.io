---
title: "How to fix 'Task not serializable' issues in Apache Spark"
date: "2017-06-12"
categories:
  - "data-engineering"
---

When using the RDD API, you can write Map functions which can serve as complex closures.

Because each Map function is executed in parallel on one of the executors, the functionality inside the Map phase, (ie.
the code) is sent to each executor in a serialized form.

An issue that often occurs is that the classes and their respective methods/fields/etc are not serializable. An occurs
in those cases as your code cannot be serialized and shipped to the executors.

The relevant error is: "org.apache.spark.SparkException: Job aborted:Task not serializable:
java.io.NotSerializableException_"_

Several remedies are proposed to solve this issue:

- Serialize the class/methods you want executed -> really hard to implement if you are referencing a library
- Use Kryo serialization -> I couldn't get it to work
- ...

**But my favorite solution is to create the closure in a static function, because invoking some static function doesn't
pass the reference to the closure and hence no need to make serializable this way.**

So I will give you an example of that:

PROBLEMATIC EXAMPLE:

Say you have the following "test" function where the WaveletTransform is some class from an external library which is
not serialized.

```java
public class FeatureExtractionTest {

  @Test

  public void test() {

    try {
      String[] files = {"/user/digitalAssistanceSystem/data/numbers/infoTrain.txt"};
      OffLineDataProvider odp = new OffLineDataProvider(files);
      odp.loadData();
      List rawEpochs = odp.getTrainingData();
      List rawTargets = odp.getTrainingDataLabels();
      JavaRDD epochs = SparkInitializer.javaSparkContext.parallelize(rawEpochs);
      JavaRDD targets = SparkInitializer.javaSparkContext.parallelize(rawTargets);
      epochs.map(new Function {
        public double[] call ( double[][] epoch) throws Exception {
          WaveletTransform waveletTransformer = new WaveletTransform(8, 512, 175, 16);
          return waveletTransformer.extractFeatures(epoch);
        }
      } ;)

    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

SOLUTION:

```java
public class FeatureExtractionTest {
  private static Function mapFunc = new Function() {
    public double[] call(double[][] epoch) {
      WaveletTransform waveletTransformer = new WaveletTransform(8, 512, 175, 16);
      return waveletTransformer.extractFeatures(epoch);
    }
  };

  @Test
  public void test() {
    try {
      String[] files = {"/user/digitalAssistanceSystem/data/numbers/infoTrain.txt"};
      OffLineDataProvider odp = new OffLineDataProvider(files);
      odp.loadData();
      List rawEpochs = odp.getTrainingData();
      List rawTargets = odp.getTrainingDataLabels();
      JavaRDD epochs = SparkInitializer.javaSparkContext.parallelize(rawEpochs);
      JavaRDD targets = SparkInitializer.javaSparkContext.parallelize(rawTargets);
      epochs.map(mapFunc);
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

}
```

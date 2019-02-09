# Posterior Shape Models

In this tutorial we will use Gaussian processes for regression tasks and experiment with the concept of posterior shape models.
This will form the basics for the next tutorial, where we will see how these tools can be applied to construct a
reconstruction of partial shapes.

##### Related resources

The following resources from our [online course](https://www.futurelearn.com/courses/statistical-shape-modelling) may provide
some helpful context for this tutorial:

- The regression problem [(Article)](https://www.futurelearn.com/courses/statistical-shape-modelling/3/steps/250360)
- Gaussian process regression [(Video)](https://www.futurelearn.com/courses/statistical-shape-modelling/3/steps/250361)
- Posterior models for different kernels [(Article)](https://www.futurelearn.com/courses/statistical-shape-modelling/3/steps/250362)

##### Preparation

As in the previous tutorials, we start by importing some commonly used objects and initializing the system.

```scala
import scalismo.geometry._
import scalismo.common._
import scalismo.ui.api._
import scalismo.mesh._
import scalismo.io.{StatisticalModelIO, MeshIO}
import scalismo.statisticalmodel._
import scalismo.numerics.UniformMeshSampler3D
import scalismo.kernels._
import breeze.linalg.{DenseMatrix, DenseVector}

scalismo.initialize()
implicit val rng = scalismo.utils.Random(42)

val ui = ScalismoUI()
```


We also load and visualize the face model:

```scala
val model = StatisticalModelIO.readStatisticalMeshModel(new java.io.File("datasets/bfm.h5")).get

val modelGroup = ui.createGroup("modelGroup")
val ssmView = ui.show(modelGroup, model, "model")
```


## Fitting observed data using Gaussian process regression

The reason we build statistical models, is that we want to use them
for explaining data. More precisely, given some observed data, we fit the model
to the data and get as a result a distribution over the model parameters.
In our case, the model is a Gaussian process model of shape deformations, and the data are observed shape deformations; I.e. deformation vectors from the reference surface.

To illustrate this process, we simulate some data. We generate  
a deformation vector at the tip of the nose, which corresponds ot a really long
nose:

```scala
val idNoseTip = PointId(8156)
val noseTipReference = model.referenceMesh.pointSet.point(idNoseTip)
val noseTipMean = model.mean.pointSet.point(idNoseTip)
val noseTipDeformation = (noseTipMean - noseTipReference) * 2.0
```

To visualize this deformation, we need to define a ```DiscreteField```, which can then be passed to the show
method of our ```ui``` object.

```scala
val noseTipDomain = UnstructuredPointsDomain(IndexedSeq(noseTipReference))
val noseTipDeformationAsSeq = IndexedSeq(noseTipDeformation)
val noseTipDeformationField = DiscreteField[_3D, UnstructuredPointsDomain[_3D], EuclideanVector[_3D]](noseTipDomain, noseTipDeformationAsSeq)

val observationGroup = ui.createGroup("observation")
ui.show(observationGroup, noseTipDeformationField, "noseTip")
```

In the next step we set up the regression. The Gaussian process model assumes, that the deformation is always only observed with some uncertainty,
which can be modelled using a normal distribution.

```scala
val noise = MultivariateNormalDistribution(DenseVector.zeros[Double](3), DenseMatrix.eye[Double](3))
```

The data for the regression is specified by a sequence of triples, consisting of the point of the reference, the
corresponding deformation vector, as well as the noise at that point:

```scala
val regressionData = IndexedSeq((noseTipReference, noseTipDeformation, noise))
```

We can now obtain the regression result by feeding this data to the method ```regression``` of the ```GaussianProcess``` object:

```scala
val gp : LowRankGaussianProcess[_3D, EuclideanVector[_3D]] = model.gp.interpolate(NearestNeighborInterpolator())
val posteriorGP : LowRankGaussianProcess[_3D, EuclideanVector[_3D]] = LowRankGaussianProcess.regression(gp, regressionData)
```

Note that the result of the regression is again a Gaussian process, over the same domain as the original process. We call this the *posterior process*.
This construction is very important in Scalismo. Therefore, we have a convenience method defined directly on the Gaussian process object. We could write the same in
the more succinctly:

```scala
gp.posterior(regressionData)
```

Independently of how you call the method, the returned type is a continuous (low rank) Gaussian Process from which we can now sample deformations at any set of points:

```scala
val posteriorSample : DiscreteField[_3D, UnstructuredPointsDomain[_3D], EuclideanVector[_3D]] 
    = posteriorGP.sampleAtPoints(model.referenceMesh.pointSet)
val posteriorSampleGroup = ui.createGroup("posteriorSamples")
for (i <- 0 until 10) {
    ui.show(posteriorSampleGroup, posteriorSample, "posteriorSample")
}
```


### Posterior of a StatisticalMeshModel:

Given that the StatisticalMeshModel is merely a wrapper around a GP, the same posterior functionality is available for statistical mesh models:

```scala
val littleNoise = MultivariateNormalDistribution(DenseVector.zeros[Double](3), DenseMatrix.eye[Double](3) * 0.01)
val pointOnLargeNose = noseTipReference + noseTipDeformation
val discreteTrainingData = IndexedSeq((PointId(8156), pointOnLargeNose, littleNoise))
val meshModelPosterior : StatisticalMeshModel = model.posterior(discreteTrainingData)
```

Notice in this case, since we are working with a discrete Gaussian process, the observed data is specified in terms of the *point identifier* of the nose tip point instead of its 3D coordinates.

Let's visualize the obtained posterior model:

```scala
val posteriorModelGroup = ui.createGroup("posteriorModel")
ui.show(posteriorModelGroup, meshModelPosterior, "NoseyModel")
```

##### Exercise: sample a few random faces from the graphical interface using the random button. Notice how all faces display large noses :) with the tip of the nose remaining close to the selected landmark.


Here again we obtain much more than just a single face instance fitting the input data: we get a full normal distribution of shapes fitting the observation. The **most probable** shape, and hence our best fit, is the **mean** of the posterior.

We notice by sampling from the posterior model that we tend to get faces with rather large noses. This is since we chose our observation to be twice the length of the
average (mean) deformation at the tip of the nose.


#### Landmark uncertainty:

As you could see previously, when specifying the training data for the posterior GP computation, we model the uncertainty of the input data.
This can allow to tune the fitting results.

Below, we perform the posterior computation again with, this time, a 5 times bigger noise variance.


```scala
val largeNoise = MultivariateNormalDistribution(DenseVector.zeros[Double](3), DenseMatrix.eye[Double](3) * 5.0)
val discreteTrainingDataLargeNoise = IndexedSeq((PointId(8156), pointOnLargeNose, largeNoise))
val discretePosteriorLargeNoise = model.posterior(discreteTrainingDataLargeNoise)
val posteriorGroupLargeNoise = ui.createGroup("posteriorLargeNoise")
ui.show(posteriorGroupLargeNoise, discretePosteriorLargeNoise, "NoisyNoseyModel")
```

##### Exercise: sample a few faces from this noisier posterior model. How flexible is the model compared to the previous posterior? How well are the sample tips fitting to the indicated landmark when compared with the previous posterior?


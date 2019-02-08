{% include head.html %}

# Active Shape Model fitting

In this tutorial we show how we can perform active shape model fitting in Scalismo.

##### Related resources

The following resources from our [online course](https://www.futurelearn.com/courses/statistical-shape-modelling) may provide
some helpful context for this tutorial:

- Fitting models to images [(Video)](https://www.futurelearn.com/courses/statistical-shape-modelling/3/steps/250379)

##### Preparation

As in the previous tutorials, we start by importing some commonly used objects and initializing the system.

```scala
  import scalismo.geometry._
  import scalismo.ui.api._
  import scalismo.registration._
  import scalismo.mesh.TriangleMesh
  import scalismo.statisticalmodel.asm._
  import scalismo.io.{ActiveShapeModelIO, ImageIO}
  import breeze.linalg.{DenseVector}


scalismo.initialize()
implicit val rng = scalismo.utils.Random(42)

val ui = ScalismoUI()
```


## Active Shape models in Scalismo

Scalismo provides full support for Active Shape models. This means we can use it to learn active shape models from
a set of images and corresponding contour, we can save these models, and we can use them to fit images. In this tutorial
we will assume that the model has already been built and will only concentrate on model fitting.


We can load an Active Sahpe Model as follows:

```scala
val asm = ActiveShapeModelIO.readActiveShapeModel(new java.io.File("datasets/femur-asm.h5")).get
```

An ActiveShapeModel instance in Scalismo is a combination of a statistical shape model and an intensity model.
Using the method ```statisticalModel```, we can obtain the shape model part. Let's visualize this model:

```scala
val modelGroup = ui.createGroup("modelGroup") 
val modelView = ui.show(modelGroup, asm.statisticalModel, "shapeModel")
```

The second part of the model is the intensity model. This model consists of a set of profiles,
which are attached to specific vertices of the shape model, indicated by the ```pointId```.
For each of the profile points, an probability distribution, which represent the intensity model for this profile.

The following code shows how this information can be accessed:

```scala
val profiles = asm.profiles
profiles.map(profile => {
    val pointId = profile.pointId
    val distribution = profile.distribution
})
```

#### Finding likely model correspondences in an image

The main usage of this information is the points of the image, which are most likely to correspond to the given "profile points* in the model.
More precisely, let $$p_i$$ denote the i-th profile in the model. We can use the information to evaluate for any set of points
$(x_1, \ldots, x_n)$, how likely it is that a point $x_j$ corresponds to the profile point $p_i$, based on the image intensity patterns
$$(\rho(x_1), \ldots, \rho(x_n))$$ we find at these points in an image.

To illustrate this, we first load an image:

```scala
val image = ImageIO.read3DScalarImage[Short](new java.io.File("datasets/femur-image.nii")).get.map(_.toFloat)
val targetGroup = ui.createGroup("target")

val imageView = ui.show(targetGroup, image, "image")
```

The ASM implementation in Scalismo, is not restricted to work with the raw intensities, but the active shape model may first apply some preprocessing,
such as smooth, applying a gradient transform, etc.  Thus in a first step we obtain this preprocess iamge uing the prepocessor method of the ```asm``` object:

```scala
val preprocessedImage = asm.preprocessor(image)
```

We can now extract features at a given point:

```scala
val point1 = image.domain.origin + Vector(10.0, 10.0, 10.0)
val profile = asm.profiles.head
val feature1 : DenseVector[Double] = asm.featureExtractor(preprocessedImage, point1, asm.statisticalModel.mean, profile.pointId).get
```

Here we specified the preprocessed image, a point in the image where whe want the evaluate the feature vector, a mesh instance and a point id for the mesh.
The mesh instance and point id are needed, since a feature extractor might choose to extract the feature based on mesh information, such as the normal direction
of a line at this point.

We can retrieve the likelihood, that each corresponding point corresponds to a given profile point:

```scala
val point2 = image.domain.origin + Vector(20.0, 10.0, 10.0)
val featureVec1 = asm.featureExtractor(preprocessedImage, point1, asm.statisticalModel.mean, profile.pointId).get
val featureVec2 = asm.featureExtractor(preprocessedImage, point2, asm.statisticalModel.mean, profile.pointId).get

val probabilityPoint1 = profile.distribution.logpdf(featureVec1)
val probabilityPoint2 = profile.distribution.logpdf(featureVec2)
```

#### Evaluating the likelihood of a model instance overlapping with an image

We can extend the method used above to compute the likelihood for all profile points of an Active Shape Model,
in order to determine whether a given model instance nicely fits to the image.

Given the model instance, we will get the position of each profile point in the current instance,
evaluate its likelihood and then compute the joint likelihood for all profiles.

Assuming independence, the joint probability is just the product of the probability at the individual profile points.
In order not to get too extreme values, we use log probabilities here (and consequently the product becomes a sum).

```scala
def likelihoodForMesh(asm : ActiveShapeModel, mesh : TriangleMesh[_3D], preprocessedImage: PreprocessedImage) : Double = {

    val ids = asm.profiles.ids

    val likelihoods = for (id <- ids) yield {
        val profile = asm.profiles(id)
        val profilePointOnMesh = mesh.pointSet.point(profile.pointId)
        val featureAtPoint = asm.featureExtractor(preprocessedImage, profilePointOnMesh, mesh, profile.pointId).get
        profile.distribution.logpdf(featureAtPoint)
    }
    likelihoods.sum
}
```

This method allows us to compute for each mesh, represented by the model, how likely it is to correspond
to the given image.

```scala
val sampleMesh1 = asm.statisticalModel.sample 
val sampleMesh2 = asm.statisticalModel.sample
println("Likelihood for mesh 1 = " + likelihoodForMesh(asm, sampleMesh1, preprocessedImage))
println("Likelihood for mesh 2 = " + likelihoodForMesh(asm, sampleMesh2, preprocessedImage))
```

This information is all that is need to write probabilistic fitting methods methods using Markov Chain Monte Carlo
methods, which will be discussed in the next tutorial.

### The original Active Shape Model Fitting

Although in Scalismo we prefer to follow a probabilistic approach and use the ASM objects
only to evaluate the likelyhood of a shape,
we have also implemented the original active shape model fitting algorithm, as proposed by [Cootes and Taylor](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.141.3089&rep=rep1&type=pdf).

To configure the fitting process, we need to set up a search method, which searches points in the image as candidates
for corresponding points. One search strategy that is already implemented in Scalismo is to search along
the normal direction of a model point. This behavior is provided by the ```NormalDirectionSearchPointSampler```

```scala
val searchSampler = NormalDirectionSearchPointSampler(numberOfPoints = 100, searchDistance = 3)
```

We also need to pass some configuration parameters to the fitting process:

```scala
val config = FittingConfiguration(featureDistanceThreshold = 3, pointDistanceThreshold = 5, modelCoefficientBounds = 3)
```

The first determines how far away (as measured by the mahalanobis distance) an intensity feature can be, such that it is still
chosen as corresponding. The ```pointDistanceThreshold``` does the same for the distance of the points; I.e. in this  case points which are
more than 5 standard deviations aways are not chosen as corresponding points. The last parameters determines how
large coefficients of the model can become in the fitting process. Whenver a model parameter is larger than this threshold,
it will be set back to this maximal value. This introduces a regularization into the fitting, which prevents the shape
from becoming too unlikely.

The ASM fitting algorithm optimizes both the pose (as defined by a RigidTransformation) and the shape.
In order to allow it to optimize the rotation, it is important that we choose a rotation center, which is approximately
the center of mass of the model:

```scala
// make sure we rotate around a reasonable center point
val modelBoundingBox = asm.statisticalModel.referenceMesh.boundingBox
val rotationCenter = modelBoundingBox.origin + modelBoundingBox.extent * 0.5    
```

We set up the initial transformations

```scala
// we start with the identity transform
val translationTransformation = TranslationTransform(Vector(0, 0, 0))
val rotationTransformation = RotationTransform(0, 0, 0, rotationCenter)
val initialRigidTransformation = RigidTransformation(translationTransformation, rotationTransformation)
val initialModelCoefficients = DenseVector.zeros[Double](asm.statisticalModel.rank)
val initialTransformation = ModelTransformations(initialModelCoefficients, initialRigidTransformation)
```

To start the fitting, we obtain an iterator, which we subsequently use to drive the iteration.

```scala
   val numberOfIterations = 20
   val asmIterator = asm.fitIterator(image, searchSampler, numberOfIterations, config, initialTransformation)
```

Especially in a debugging phase, we want to visualize the result in every iteration. The following code shows,
how we can obtain a new iterator, which updates the pose transformation and model coefficients in the ```ui```
in every iteration:

```scala
val asmIteratorWithVisualization = asmIterator.map(it => {
  it match {
    case scala.util.Success(iterationResult) => {
      modelView.shapeModelTransformationView.poseTransformationView.transformation = iterationResult.transformations.rigidTransform
      modelView.shapeModelTransformationView.shapeTransformationView.coefficients = iterationResult.transformations.coefficients
    }
    case scala.util.Failure(error) => System.out.println(error.getMessage)
  }
  it
})   
```

To run the model, and get the result, we finally consume the iterator:

```scala
val result = asmIteratorWithVisualization.toIndexedSeq.last
val finalMesh = result.get.mesh
```


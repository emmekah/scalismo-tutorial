{% include head.html %}

# Model fitting using MCMC - Fitting a shape model

In this tutorial we show how the MCMC framework, which was introduced in the previous
tutorial, can be used for shape model fitting.

We will illustrate it by computing a posterior of a shape model,
given a set of corresponding landmark pairs. This is the same setup that we have
discussed in the tutorial about Gaussian process regression. The difference is,
that here we will also allow for rotation and translation of the model. In this setting,
it is not possible anymore to compute the posterior analytically. Rather, our only hope are approximation methods, such as
using Markov-chain monte carlo methods.

In this tutorial we show not only a working example, but also how to make it
computationally efficient. Making the individual parts as efficient as possible is
important in sampling approaches, as we need to produce many samples to get accurate
estimates.

##### Preparation

As in the previous tutorials, we start by importing some commonly used objects and
initializing the system.

```scala
import breeze.linalg.{DenseMatrix, DenseVector}
import scalismo.common.PointId
import scalismo.geometry._
import scalismo.io.{LandmarkIO, MeshIO, StatisticalModelIO}
import scalismo.mesh.TriangleMesh
import scalismo.registration.{RigidTransformation, RigidTransformationSpace, RotationTransform, TranslationTransform}
import scalismo.sampling.algorithms.MetropolisHastings
import scalismo.sampling.evaluators.ProductEvaluator
import scalismo.sampling.proposals.MixtureProposal
import scalismo.sampling.loggers.AcceptRejectLogger
import scalismo.sampling.{DistributionEvaluator, ProposalGenerator, TransitionProbability}
import scalismo.statisticalmodel.{MultivariateNormalDistribution, StatisticalMeshModel}
import scalismo.ui.api.ScalismoUI
import scalismo.utils.Memoize

implicit val rng = scalismo.utils.Random(42)
scalismo.initialize()

val ui = ScalismoUI()
```

### Loading and visualizing the data

In a first step, we load and visualize all the data that we need.
First, we load the statistical model:

```scala
  val model = StatisticalModelIO.readStatisticalMeshModel(new java.io.File("datasets/bfm.h5")).get

  val modelGroup = ui.createGroup("model")
  val modelView = ui.show(modelGroup, model, "model")
  modelView.meshView.opacity = 0.5
```

In this example, we will fit the model such that a set of model landmarks, coincide
with a set of landmark points defined on a target face. We load and visualize the corresponding landmark data:

```scala
  val modelLms = LandmarkIO.readLandmarksJson[_3D](new java.io.File("datasets/modelLM_mcmc.json")).get
  val modelLmViews = ui.show(modelGroup, modelLms, "modelLandmarks")
  modelLmViews.foreach(lmView => lmView.color = java.awt.Color.BLUE)

  val targetGroup = ui.createGroup("target")
  
  val targetLms = LandmarkIO.readLandmarksJson[_3D](new java.io.File("datasets/targetLM_mcmc.json")).get
  val targetLmViews = ui.show(targetGroup, targetLms, "targetLandmarks")
  modelLmViews.foreach(lmView => lmView.color = java.awt.Color.RED)
```

In the following, we will refer to the points on the model using their point id, while the target
position is represented as physical points. The reason why we use the point id for the model is that the model instances,
and therefore the points, which are represented by the point id, are changing as we fit the model.

```scala
val modelLmIds =  modelLms.map(l => model.mean.pointSet.pointId(l.point).get)
val targetPoints = targetLms.map(l => l.point)
```

We assume that the target points we observe are subject to
noise, which we model as a normal distribution with $$3 mm$$ standard deviation (hence $$9 mm$$ variance):

```scala
val landmarkNoiseVariance = 9.0 
val uncertainty = MultivariateNormalDistribution(
  DenseVector.zeros[Double](3),
  DenseMatrix.eye[Double](3) * landmarkNoiseVariance
)
```

We summarize the correspondences as a triple, consisting of model id, target position
and uncertainty.

```scala
val correspondences = modelLmIds.zip(targetPoints).map(modelIdWithTargetPoint => {
  val (modelId, targetPoint) =  modelIdWithTargetPoint
  (modelId, targetPoint, uncertainty)
})
  
```

### The parameter class

In this example, we want to model the posterior $$p(\theta | D)$$, where
the parameters $$\theta =( t, r, \alpha)$$ consist of the translation parameters
$$t=(t_x, t_y, t_z)$$, the rotation parameters $$r = (\phi, \psi, \omega)$$,
represented as Euler angles as well a shape model coefficients $$\alpha = (\alpha_1, \ldots, \alpha_n)$$.

```scala
case class Parameters(translationParameters: EuclideanVector[_3D],
                     rotationParameters: (Double, Double, Double),
                     modelCoefficients: DenseVector[Double])
```

As in the previous tutorial, we wrap this into a class representing the sample, which can keep track by whom
it was generated. Furthermore, we will add convenience method,
which builds a ```RigidTransformation``` from the parameters. As a rigid transformation
is not completely determined by the translation and rotation parameters, we need to
store also the center of rotation.

```scala

  case class Sample(generatedBy : String, parameters : Parameters, rotationCenter: Point[_3D]) {
    def poseTransformation : RigidTransformation[_3D] = {

      val translation = TranslationTransform(parameters.translationParameters)
      val rotation = RotationTransform(
        parameters.rotationParameters._1,
        parameters.rotationParameters._2,
        parameters.rotationParameters._3,
        rotationCenter
      )
      RigidTransformation(translation, rotation)
    }
  }

```


### Evaluators: Modelling the target density

As in the previous tutorial, we represent the unnormalized posterior distribution
as the product of prior and likelihood:
$$p(\theta | D) \propto p(\theta) p(D | \theta)$$,
where $$D$$ denotes the data (i.e. the corresponding landmark points) and $$\theta$$
are our parameters.

As a prior over the shape parameters is given by the shape model. For the
translation and rotation, we assume a zero-mean normal distribution:

```scala
case class PriorEvaluator(model: StatisticalMeshModel)
    extends DistributionEvaluator[Sample] {

    val translationPrior = breeze.stats.distributions.Gaussian(0.0, 5.0)
    val rotationPrior = breeze.stats.distributions.Gaussian(0, 0.1)

    override def logValue(sample: Sample): Double = {
      model.gp.logpdf(sample.parameters.modelCoefficients) +
      translationPrior.logPdf(sample.parameters.translationParameters.x) +
      translationPrior.logPdf(sample.parameters.translationParameters.y) +
      translationPrior.logPdf(sample.parameters.translationParameters.z) +
      rotationPrior.logPdf(sample.parameters.rotationParameters._1) +
      rotationPrior.logPdf(sample.parameters.rotationParameters._2) +
      rotationPrior.logPdf(sample.parameters.rotationParameters._3)
    }
  }
```

To compute the likelihood $$p(D | \theta)$$ we first compute
the current model instance as determined by the shape and pose parameters.
From this model instance, the points at the given points id are extracted and
the distance to their target position is computed. This distance is what was
modelled by the uncertainty of the observations. We can therefore directly use
the modelled uncertainty to compute the likelihood of our model given the data:

```scala
case class SimpleCorrespondenceEvaluator(model: StatisticalMeshModel,
                                   correspondences: Seq[(PointId, Point[_3D], MultivariateNormalDistribution)])
  extends DistributionEvaluator[Sample] {
   
  override def logValue(sample: Sample): Double = {

    val currModelInstance = model.instance(sample.parameters.modelCoefficients).transform(sample.poseTransformation)
  
    val likelihoods = correspondences.map( correspondence => {
      val (id, targetPoint, uncertainty) = correspondence
      val modelInstancePoint = currModelInstance.pointSet.point(id)
      val observedDeformation = targetPoint - modelInstancePoint
  
      uncertainty.logpdf(observedDeformation.toBreezeVector)
    })
  
  
    val loglikelihood = likelihoods.sum
    loglikelihood
  }
}
```

Conceptually, this is all that needed to be done to specify the target distribution.
In practice, we are interested to make these evaluators as efficient as possible,
as they are usually called thousands of times.

##### Performance improvements

In the above implementation, we compute a full model instance (the new position of all the mesh points
represented by the shape model), although we are only interested in the position of the landmark points.
This is rather inefficient. A more efficient version would first marginalize the model to the
points of interest. Since marginalization changes the point ids, we need to map the
ids given as```correspondences``` to their new ids. This is achieved by the following helper function:

```scala
def marginalizeModelForCorrespondences(model: StatisticalMeshModel,
                                         correspondences: Seq[(PointId, Point[_3D], MultivariateNormalDistribution)])
    : (StatisticalMeshModel, Seq[(PointId, Point[_3D], MultivariateNormalDistribution)]) = {

    val (modelIds, _, _) = correspondences.unzip3
    val marginalizedModel = model.marginal(modelIds.toIndexedSeq)
    val newCorrespondences = correspondences.map(idWithTargetPoint => {
      val (id, targetPoint, uncertainty) = idWithTargetPoint
      val modelPoint = model.referenceMesh.pointSet.point(id)
      val newId = marginalizedModel.referenceMesh.pointSet.findClosestPoint(modelPoint).id
      (newId, targetPoint, uncertainty)
    })
    (marginalizedModel, newCorrespondences)
  }
```

The more efficient version of the evaluator uses now the marginalized model to evaluate the likelihood:

```scala
 case class CorrespondenceEvaluator(model: StatisticalMeshModel,
                                     correspondences: Seq[(PointId, Point[_3D], MultivariateNormalDistribution)])
    extends DistributionEvaluator[Sample] {

    val (marginalizedModel, newCorrespondences) = marginalizeModelForCorrespondences(model, correspondences)

    override def logValue(sample: Sample): Double = {

      val currModelInstance = marginalizedModel.instance(sample.parameters.modelCoefficients).transform(sample.poseTransformation)

      val likelihoods = newCorrespondences.map( correspondence => {
        val (id, targetPoint, uncertainty) = correspondence
        val modelInstancePoint = currModelInstance.pointSet.point(id)
        val observedDeformation = targetPoint - modelInstancePoint

        uncertainty.logpdf(observedDeformation.toBreezeVector)
      })


      val loglikelihood = likelihoods.sum
      loglikelihood
    }
  }
```

In order for the Metropolis-Hastings algorithm to decide if a new sample is accepted,
the likelihood needs to be computed several times for each set of parameters. To further
increase the efficiency, we should therefore cache the computations, such that when
an evaluator is used the second time with the same parameters, the ```logValue``` is
not recomputed, but simply taken from cache. Using the following utility class,
we can obtain for any evaluator a new evaluator, which performs such caching:

```scala
 case class CachedEvaluator[A](evaluator: DistributionEvaluator[A]) extends DistributionEvaluator[A] {
    val memoizedLogValue = Memoize(evaluator.logValue, 10)

    override def logValue(sample: A): Double = {
      memoizedLogValue(sample)
    }
  }

```

#### The posterior evaluator

Given these evaluators, we can now build the computationally efficient version of
our target density $$p(\theta | D)$$

```scala
val likelihoodEvaluator = CachedEvaluator(CorrespondenceEvaluator(model, correspondences))
val priorEvaluator = CachedEvaluator(PriorEvaluator(model))

val posteriorEvaluator = ProductEvaluator(priorEvaluator, likelihoodEvaluator)
```

### The proposal generator

As in the previous tutorials, we will use simple random walk proposals.
We will define separate proposals for shape, translation and rotation.
On one hand, this lets us set the step length (i.e. stddev of the distribution from which we
sample the next step) individually for each group, and thus to incorporate our knowledge
that changes in rotation will be much smaller than the shape changes. On the other hand,
splitting the parameter updates in blocks will increase our chance for the random updates
to be accepted. The reason for this is that when many parameters are updated at one,
chances are high that some of the proposed changes make the new state more unlikely,
and hence increase the chance of the new state being rejected.

The definition of the proposals are straight-forward.

We start with the shape update proposal:

```scala
  case class ShapeUpdateProposal(paramVectorSize : Int, stddev: Double)
    extends ProposalGenerator[Sample]  with TransitionProbability[Sample] {

    val perturbationDistr = new MultivariateNormalDistribution(
      DenseVector.zeros(paramVectorSize),
      DenseMatrix.eye[Double](paramVectorSize) * stddev * stddev
    )


    override def propose(sample: Sample): Sample = {
      val perturbation = perturbationDistr.sample()
      val newParameters = sample.parameters.copy(modelCoefficients = sample.parameters.modelCoefficients + perturbationDistr.sample)
      sample.copy(generatedBy = s"ShapeUpdateProposal ($stddev)", parameters = newParameters)
    }

    override def logTransitionProbability(from: Sample, to: Sample) = {
      val residual = to.parameters.modelCoefficients - from.parameters.modelCoefficients
      perturbationDistr.logpdf(residual)
    }
  }
```

The update of the roation parameters is very similar. Note that we only update the
rotation parameters, but keep the center of rotation unchanged.

```scala
 case class RotationUpdateProposal(stddev: Double) extends
    ProposalGenerator[Sample]  with TransitionProbability[Sample] {
    val perturbationDistr = new MultivariateNormalDistribution(
      DenseVector.zeros[Double](3),
      DenseMatrix.eye[Double](3) * stddev * stddev)
    def propose(sample: Sample): Sample= {
      val perturbation = perturbationDistr.sample
      val newRotationParameters = (
        sample.parameters.rotationParameters._1 + perturbation(0),
        sample.parameters.rotationParameters._2 + perturbation(1),
        sample.parameters.rotationParameters._3 + perturbation(2)
      )
      val newParameters = sample.parameters.copy(rotationParameters = newRotationParameters)
      sample.copy(generatedBy = s"RotationUpdateProposal ($stddev)", parameters = newParameters)
    }
    override def logTransitionProbability(from: Sample, to: Sample) = {
      val residual = DenseVector(
        to.parameters.rotationParameters._1 - from.parameters.rotationParameters._1,
        to.parameters.rotationParameters._2 - from.parameters.rotationParameters._2,
        to.parameters.rotationParameters._3 - from.parameters.rotationParameters._3
      )
      perturbationDistr.logpdf(residual)
    }
  }
```

Finally, we define the update proposal for the translation:

```scala
  case class TranslationUpdateProposal(stddev: Double) extends
     ProposalGenerator[Sample]  with TransitionProbability[Sample] {
 
     val perturbationDistr = new MultivariateNormalDistribution( DenseVector.zeros(3),
       DenseMatrix.eye[Double](3) * stddev * stddev)
 
     def propose(sample: Sample): Sample= {
       val newTranslationParameters = sample.parameters.translationParameters + EuclideanVector.fromBreezeVector(perturbationDistr.sample())
       val newParameters = sample.parameters.copy(translationParameters = newTranslationParameters)
       sample.copy(generatedBy = s"TranlationUpdateProposal ($stddev)", parameters = newParameters)
     }
 
     override def logTransitionProbability(from: Sample, to: Sample) = {
       val residual = to.parameters.translationParameters - from.parameters.translationParameters
       perturbationDistr.logpdf(residual.toBreezeVector)
     }
   }

```

The final proposal is a mixture of the three proposals we defined above.
We choose to update the shape more often than the translation and rotation parameters,
as we expect most changes to be shape changes.

```scala
val shapeUpdateProposal = ShapeUpdateProposal(model.rank, 0.1)
val rotationUpdateProposal = RotationUpdateProposal(0.01)
val translationUpdateProposal = TranslationUpdateProposal(1.0)
val generator = MixtureProposal.fromProposalsWithTransition(
                                   (0.6, shapeUpdateProposal),
                                   (0.2, rotationUpdateProposal),
                                   (0.2, translationUpdateProposal)
                               )
  
```


#### Building the Markov Chain

For running the Markov Chain, we proceed exactly as in the previous tutorial. We start by defining the logger,
to compute the accept/reject ratios of the individual generators

```scala
class Logger extends AcceptRejectLogger[Sample] {
    private val numAccepted = collection.mutable.Map[String, Int]()
    private val numRejected = collection.mutable.Map[String, Int]()

    override def accept(current: Sample,
                        sample: Sample,
                        generator: ProposalGenerator[Sample],
                        evaluator: DistributionEvaluator[Sample]
                       ): Unit = {
      val numAcceptedSoFar = numAccepted.getOrElseUpdate(sample.generatedBy, 0)
      numAccepted.update(sample.generatedBy, numAcceptedSoFar + 1)
    }

    override def reject(current: Sample,
                          sample: Sample,
                          generator: ProposalGenerator[Sample],
                          evaluator: DistributionEvaluator[Sample]
                         ): Unit = {
      val numRejectedSoFar = numRejected.getOrElseUpdate(sample.generatedBy, 0)
      numRejected.update(sample.generatedBy, numRejectedSoFar + 1)
    }


    def acceptanceRatios() : Map[String, Double] = {
      val generatorNames = numRejected.keys.toSet.union(numAccepted.keys.toSet)
      val acceptanceRatios = for (generatorName <- generatorNames ) yield {
        val total = (numAccepted.getOrElse(generatorName, 0)
                     + numRejected.getOrElse(generatorName, 0)).toDouble
        (generatorName, numAccepted.getOrElse(generatorName, 0) / total)
      }
      acceptanceRatios.toMap
    }
  }
```

We then create the initial sample, where we choose here the center of mass of the model mean as the
rotation center.

```scala
def computeCenterOfMass(mesh : TriangleMesh[_3D]) : Point[_3D] = {
    val normFactor = 1.0 / mesh.pointSet.numberOfPoints
    mesh.pointSet.points.foldLeft(Point(0, 0, 0))((sum, point) => sum + point.toVector * normFactor)
}


val initialParameters = Parameters(
    EuclideanVector(0, 0, 0),
    (0.0, 0.0, 0.0),
    DenseVector.zeros[Double](model.rank)
    )

val initialSample = Sample("initial", initialParameters, computeCenterOfMass(model.mean))

```

*Remark: Setting the rotation center correctly is very important for the rotation proposal to work as expected.
Fortunately, most of the time this error is easy to diagnose, as the acceptance ratio of the rotation proposal will be unexpectedly low.*

Next we set up the chain and obtain an iterator.

```scala
val chain = MetropolisHastings(generator, posteriorEvaluator)
val logger = new Logger()    
val mhIterator = chain.iterator(initialSample, logger)
```

In this example we are interested to visualize some samples from the posterior as we run the chain. This can be done
by augmenting the iterator with visualization code:

```scala
val samplingIterator = for((sample, iteration) <- mhIterator.zipWithIndex) yield {
   println("iteration " + iteration)
   if (iteration % 500 == 0) {
     modelView.shapeModelTransformationView.shapeTransformationView.coefficients = sample.parameters.modelCoefficients
     modelView.shapeModelTransformationView.poseTransformationView.transformation = sample.poseTransformation
   }
   sample
 }   
```

Finally, we draw the samples using the chain by consuming the iterator. We drop the first 1000 iterations, as the
chain needs some burn-in time to converge to a equilibrium solution:

```scala
val samples = samplingIterator.drop(1000).take(10000).toIndexedSeq
```


Before working with the results, we check the acceptance ratios to verify that all the proposals work as expected:

```scala
println(logger.acceptanceRatios())
// Map(RotationUpdateProposal (0.01) -> 0.6971894832275612, TranlationUpdateProposal (1.0) -> 0.5043859649122807, ShapeUpdateProposal (0.1) -> 0.7907262398280362)
```

### Analyzing the results

Once we have the samples, we can now use them to analyze our fit.
For example, we can select the best fit from these samples and visualize it

```scala
val bestSample = samples.maxBy(posteriorEvaluator.logValue)
val bestFit = model.instance(bestSample.parameters.modelCoefficients).transform(bestSample.poseTransformation)
val resultGroup = ui.createGroup("result")
ui.show(resultGroup, bestFit, "best fit")
```

The samples allow us to infer much more about the distribution. For example, we can estimate the expected position of
any point in the model and the variance from the samples:

```scala
  
 def computeMean(model : StatisticalMeshModel, id: PointId): Point[_3D] = {
    var mean = EuclideanVector(0, 0, 0)
    for (sample <- samples) yield {
      val modelInstance = model.instance(sample.parameters.modelCoefficients)
      val pointForInstance = modelInstance.transform(sample.poseTransformation).pointSet.point(id)
      mean += pointForInstance.toVector
    }
    (mean * 1.0 / samples.size).toPoint
  }

  def computeCovarianceFromSamples(model : StatisticalMeshModel, id: PointId, mean: Point[_3D]): SquareMatrix[_3D] = {
    var cov = SquareMatrix.zeros[_3D]
    for (sample <- samples) yield {
      val modelInstance = model.instance(sample.parameters.modelCoefficients)
      val pointForInstance = modelInstance.transform(sample.poseTransformation).pointSet.point(id)
      val v = pointForInstance - mean
      cov += v.outer(v)
    }
    cov * (1.0 / samples.size)
  }

```

For efficiency reasons, we do the computations here only for the landmark points, using again the marginalized model:

```scala
val (marginalizedModel, newCorrespondences) = marginalizeModelForCorrespondences(model, correspondences)    
// marginalizedModel: StatisticalMeshModel = StatisticalMeshModel(
//   TriangleMesh3D(
//     scalismo.common.UnstructuredPointsDomain3D@16b01f4d,
//     TriangleList(Vector())
//   ),
//   DiscreteLowRankGaussianProcess(
//     scalismo.common.UnstructuredPointsDomain3D@16b01f4d,
//     DenseVector(-2.4434967041015625, 2.350008010864258, 0.442657470703125, -2.9081268310546875, 1.451995849609375, 0.908905029296875, -1.6051483154296875, 0.8872489929199219, 0.2577056884765625, -1.0640869140625, 1.5610084533691406, 1.529266357421875, 7.3378448486328125, -1.0377426147460938, -0.8446807861328125, 4.6402587890625, -1.0488052368164062, -0.2399749755859375),
//     DenseVector(1448832.5, 366419.90625, 272218.9375, 174594.765625, 118833.6953125, 96322.5703125, 65858.7265625, 32242.50390625, 22365.66796875, 2.256714282111716E-8),
//     -0.0020904541015625    -0.00250244140625     ... (10 total)
// -0.00147247314453125   9.765625E-4           ...
// 0.002288818359375      -0.00201416015625     ...
// -0.001922607421875     -0.0016021728515625   ...
// -9.11712646484375E-4   1.6021728515625E-4    ...
// 0.002197265625         -0.001373291015625    ...
// -0.0019683837890625    -0.002044677734375    ...
// -8.029937744140625E-4  -1.52587890625E-4     ...
// -0.0016937255859375    0.001495361328125     ...
// -0.002166748046875     -0.00250244140625     ...
// -9.307861328125E-4     8.335113525390625E-4  ...
// -0.001861572265625     0.002166748046875     ...
// -0.003875732421875     -6.103515625E-5       ...
// 4.425048828125E-4      -7.5531005859375E-4   ...
// 3.96728515625E-4       3.204345703125E-4     ...
// -0.0030364990234375    -0.0048828125         ...
// 0.00316619873046875    -3.8909912109375E-4   ...
// 3.662109375E-4         1.52587890625E-5      ...
//   )
// )
// newCorrespondences: Seq[(PointId, Point[_3D], MultivariateNormalDistribution)] = List(
//   (
//     PointId(0),
//     Point3D(148.41268920898438, -5.294463634490967, 290.7681884765625),
//     MultivariateNormalDistribution(
//       DenseVector(0.0, 0.0, 0.0),
//       9.0  0.0  0.0  
// 0.0  9.0  0.0  
// 0.0  0.0  9.0  
//     )
//   ),
//   (
//     PointId(1),
//     Point3D(142.35382080078125, -5.664807319641113, 268.0462951660156),
//     MultivariateNormalDistribution(
//       DenseVector(0.0, 0.0, 0.0),
//       9.0  0.0  0.0  
// 0.0  9.0  0.0  
// 0.0  0.0  9.0  
//     )
//   ),
//   (
//     PointId(2),
//     Point3D(140.85040283203125, -5.128785610198975, 225.04928588867188),
//     MultivariateNormalDistribution(
//       DenseVector(0.0, 0.0, 0.0),
//       9.0  0.0  0.0  
// 0.0  9.0  0.0  
// 0.0  0.0  9.0  
//     )
//   ),
//   (
//     PointId(3),
//     Point3D(143.13357543945312, -4.769620895385742, 202.04534912109375),
//     MultivariateNormalDistribution(
//       DenseVector(0.0, 0.0, 0.0),
//       9.0  0.0  0.0  
// 0.0  9.0  0.0  
// 0.0  0.0  9.0  
//     )
//   ),
//   (
//     PointId(4),
//     Point3D(102.81741333007812, 32.58876037597656, 247.52294921875),
//     MultivariateNormalDistribution(
//       DenseVector(0.0, 0.0, 0.0),
//       9.0  0.0  0.0  
// 0.0  9.0  0.0  
// 0.0  0.0  9.0  
// ...    
for ((id, _, _) <- newCorrespondences) {
    val meanPointPosition = computeMean(marginalizedModel, id)
    println(s"expected position for point at id $id  = $meanPointPosition")
    val cov = computeCovarianceFromSamples(marginalizedModel, id, meanPointPosition)
    println(s"posterior variance computed  for point at id (shape and pose) $id  = ${cov(0,0)}, ${cov(1,1)}, ${cov(2,2)}")
}
// expected position for point at id PointId(0)  = Point3D(148.10137201095068,-7.151235826279037,291.3929897650882)
// posterior variance computed  for point at id (shape and pose) PointId(0)  = 4.254464918957609, 3.939366851939424, 3.32790002246177
// expected position for point at id PointId(1)  = Point3D(142.24949688108646,-4.4064122402505745,266.34241057439533)
// posterior variance computed  for point at id (shape and pose) PointId(1)  = 2.918968756161676, 2.159129807301085, 2.915190937982501
// expected position for point at id PointId(2)  = Point3D(141.3051915247215,-4.2897512164448095,226.94538458273826)
// posterior variance computed  for point at id (shape and pose) PointId(2)  = 2.203212589533782, 1.9443312683620244, 3.558990457373749
// expected position for point at id PointId(3)  = Point3D(143.33662067133574,-4.885817375124605,201.61254954128444)
// posterior variance computed  for point at id (shape and pose) PointId(3)  = 4.211176241740335, 3.5805967352101438, 3.8214742761066214
// expected position for point at id PointId(4)  = Point3D(102.9147937651206,32.77448813851955,248.52670379739345)
// posterior variance computed  for point at id (shape and pose) PointId(4)  = 3.670091389108584, 4.998066354925429, 3.3640625099189827
// expected position for point at id PointId(5)  = Point3D(137.63389362946234,101.3277589516202,248.64874726494665)
// posterior variance computed  for point at id (shape and pose) PointId(5)  = 8.479207875196545, 6.924214739937887, 5.29092402298062
```

It is interesting to compare this posterior variance, with the variance we would have obtained by using a Gaussian process regression,
where we would have to assume the pose to be fixed:

```scala
val posteriorFixedPose = model.posterior(correspondences.toIndexedSeq)
```

Not surprisingly, computing the variance of this models reveals, that the points vary less in this model:

```scala
for ((id, _, _) <- newCorrespondences) {
  val cov = posteriorFixedPose.cov(id, id)
  println(s"posterior variance computed by analytic posterior (shape only) for point with id $id = ${cov(0,0)}, ${cov(1,1)}, ${cov(2,2)}")
}
// posterior variance computed by analytic posterior (shape only) for point with id PointId(0) = 1.7493438462816924, 2.3773798978463545, 2.4529203603071132
// posterior variance computed by analytic posterior (shape only) for point with id PointId(1) = 1.7425234299448356, 2.347569081897583, 2.484181513654568
// posterior variance computed by analytic posterior (shape only) for point with id PointId(2) = 1.728312680395977, 2.311983615096077, 2.4760881301654973
// posterior variance computed by analytic posterior (shape only) for point with id PointId(3) = 1.7362509228394625, 2.3504558671936087, 2.4497130086130414
// posterior variance computed by analytic posterior (shape only) for point with id PointId(4) = 1.74656771644886, 2.315887303162952, 2.494671632799402
// posterior variance computed by analytic posterior (shape only) for point with id PointId(5) = 1.7511296018408746, 2.2802845887546255, 2.505733093422708
```

The reason for the reduced flexibility is that the model cannot adjust the pose, and hence has less freedom to explain the observed data.

### Beyond landmark fitting

We have shown above how Scalismo can be used to perform Bayesian model fitting on the example of fitting 3D landmarks. This example
can easily be extended to other fitting tasks, such as fitting the model to points with unkown correspondences, fitting shapes in surfaces
of fitting a model to an image using an Active Shape Model as a likelihood function. In principle, all that is required is to
change the likelihood function and rerun the fit.
In practice, however, as a change in the likelihood function can dramatically change the posterior density, it is often required
to tune the proposals, such that good convergence can be achieved. Indeed, finding good proposal distributions is the key to
applying this method successfully. The more prior knowledge about the target distribution we can incorporate into the proposals,
the faster will the chain converge to the equilibrium distribution.

For more complicated use-cases of this method in image analysis , we refer the interested reader is referred to the paper by S. Schönborn et al.
and references therein:

* Schönborn, Sandro, et al. "Markov chain monte carlo for automated face image analysis." International Journal of Computer Vision 123.2 (2017): 160-183.


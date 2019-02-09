# Building a shape model from data

The goal in this tutorial is to learn how to build a Statistical Shape Model from data in correspondence and assess the importance of rigid alignment while doing so.

##### Related resources

The following resources from our [online course](https://www.futurelearn.com/courses/statistical-shape-modelling) may provide
some helpful context for this tutorial:

- Learning a model from example data [(Video)](https://www.futurelearn.com/courses/statistical-shape-modelling/3/steps/250329)  

##### Preparation

As in the previous tutorials, we start by importing some commonly used objects and initializing the system. 

```scala mdoc:silent
import scalismo.geometry._
import scalismo.common._
import scalismo.ui.api._
import scalismo.mesh._
import scalismo.io.{StatisticalModelIO, MeshIO}
import scalismo.statisticalmodel._
import scalismo.registration._
import scalismo.statisticalmodel.dataset._

scalismo.initialize()
implicit val rng = scalismo.utils.Random(42)

val ui = ScalismoUI()
```

### Loading and preprocessing a dataset:

Let us load (and visualize )a set of face meshes based on which we would like to model shape variation:

```scala mdoc:silent
val dsGroup = ui.createGroup("datasets")

val meshFiles = new java.io.File("datasets/nonAlignedFaces/").listFiles

val (meshes, meshViews) = meshFiles.map(meshFile => {
  val mesh = MeshIO.readMesh(meshFile).get 
  val meshView = ui.show(dsGroup, mesh, "mesh")
  (mesh, meshView) // return a tuple of the mesh and the associated view
}) .unzip // take the tuples apart, to get a sequence of meshes and one of meshViews 
```

We notice two things about this dataset:

1. The shapes are not aligned
2. The shapes **are in correspondence**. This means that for every point on one of the face meshes (corner of eye, tip of nose, ...), one can identify the corresponding point on other meshes. Using the same identifiers for corresponding points ensures this.

##### Exercise: verify that the meshes are indeed in correspondence by displaying a few corresponding points.

#### Rigidly aligning the data:

In order to study shape variations, we need to eliminate variations due to relative spatial displacement of the shapes (rotation and translation).
This can be achieved by selecting one of the meshes as a reference, wo which the rest of the datasets are aligned.
In this example here, we simply take the first mesh in the list as a reference and align all the others. 

```scala mdoc:silent
val reference = meshes.head
val toAlign : IndexedSeq[TriangleMesh[_3D]] = meshes.tail
```

Given that our dataset is in correspondence, we can specify a set of point identifiers that we use to locate corresponding points on the reference mesh and on each of the meshes to be aligned to the reference. 

```scala mdoc:silent
val pointIds = IndexedSeq(2214, 6341, 10008, 14129, 8156, 47775)
val refLandmarks = pointIds.map{id => Landmark(s"L_$id", reference.pointSet.point(PointId(id))) }
```
After locating the landmark positions on the reference, we iterate on each remaining data item, identify the corresponding landmark points and then rigidly align the mesh to the reference.

```scala mdoc:silent
val alignedMeshes = toAlign.map { mesh =>    
     val landmarks = pointIds.map{id => Landmark("L_"+id, mesh.pointSet.point(PointId(id)))}
     val rigidTrans = LandmarkRegistration.rigid3DLandmarkRegistration(landmarks, refLandmarks, center = Point(0,0,0))
     mesh.transform(rigidTrans)
}
```

Now, the IndexedSeq of triangle meshes *alignedMeshes* contains the faces that are aligned to the reference mesh.

##### Exercise: verify visually that at least the first element of the aligned dataset is indeed aligned to the reference.

### Building a discrete Gaussian process from data

Now that we have our set of meshes that are in correspondence and aligned to our reference, we can turn the dataset into a set of deformation fields, as we saw previously:

```scala mdoc:silent 
val defFields = alignedMeshes.map{ m => 
    val deformationVectors = reference.pointSet.pointIds.map{ id : PointId =>  
    m.pointSet.point(id) - reference.pointSet.point(id)
  }.toIndexedSeq

  DiscreteField[_3D, UnstructuredPointsDomain[_3D], EuclideanVector[_3D]](reference.pointSet, deformationVectors)
}
```

Finally, we can compute a ```DiscreteGaussianProcess``` from the data.

```scala mdoc:silent
val interpolator = NearestNeighborInterpolator[_3D, EuclideanVector[_3D]]()
val continuousFields = defFields.map(f => f.interpolate(interpolator) )
val gp = DiscreteLowRankGaussianProcess.createUsingPCA(reference.pointSet, continuousFields)
```

##### Exercise: display the mean deformation field of the returned Gaussian Process.

##### Exercise: sample and display a few deformation fields from this GP.

##### Exercise: using the GP's *cov* method, evaluate the sample covariance between two close points on the right cheek first, and a point on the nose and one on the cheek second. What does the data tell you?

By combining this Gaussian process over deformation fields with the reference mesh, we obtain a Statistical Mesh Model:

```scala mdoc:silent
val model = StatisticalMeshModel(reference, gp.interpolate(interpolator))
```

Notice that when we visualize this mesh model in Scalismo-ui, it generates a GaussianProcessTransformation and the reference mesh in the Scene Tree.

```scala mdoc:silent
val modelGroup = ui.createGroup("model")
val ssmView = ui.show(modelGroup, model, "model")
```

### An easier way to build a model.

Performing all the operations above every time we want to build a PCA model from a set of files containing meshes in correspondence can be tedious. Therefore, Scalismo provides a handier implementation via the *DataCollection* data structure:


```scala mdoc:silent
val dc = DataCollection.fromMeshDirectory(reference, new java.io.File("datasets/nonAlignedFaces/"))._1.get
```

The *DataCollection* class in Scalismo allows grouping together a dataset of meshes in correspondence, in order to make collective operations on such sets easier.

In the code above, we created a *DataCollection* from all the meshes in correspondence that are stored in the indicated directory.

When creating the collection, we need to indicate the reference mesh.
In the case of *dc*, we chose the reference to be the *reference* mesh we previously defined: 

```scala mdoc:silent
dc.reference == reference
```

In addition to the reference, the data collection holds for each data item the transformation to apply to the reference to obtain the item:

```scala mdoc:silent
val item0 :DataItem[_3D] = dc.dataItems(0)
val transform : Transformation[_3D] = item0.transformation
```
Note that this transformation simply consists in moving each reference point by its associated deformation vector according to the vector field deforming the reference mesh into the item mesh.

Now that we have our data collection, we can build a shape model as follows: 

```scala mdoc:silent
val modelNonAligned = StatisticalMeshModel.createUsingPCA(dc).get

val modelGroup2 = ui.createGroup("modelGroup2")
ui.show(modelGroup2, modelNonAligned, "nonAligned")
```

Here again, a PCA is performed based the deformation fields retrieved from the data in correspondence.

Notice that, in this case, we built a model from **misaligned** meshes in correspondence.

##### Exercise: sample a few faces from the second model. How does the quality of the obtained shapes compare to the model built from aligned data?

##### Exercise: using the GUI, change the coefficient of the first principal component of the *nonAligned* shape model. What is the main shape variation encoded in the model?



```scala mdoc:invisible
ui.close();
```
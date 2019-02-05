
# Gaussian processes and Point Distribution Models

The goal in this tutorial is to understand the relation between Point Distribution Models (PDM) and Gaussian Processes.

As in the previous tutorials, we start by importing some commonly used objects and initializing the system. 

```scala mdoc
import scalismo.geometry.{_, Vector=>SpatialVector}
import scalismo.common._
import scalismo.ui.api._

scalismo.initialize()
implicit val rng = scalismo.utils.Random(42)

val ui = ScalismoUI()
```



# Gaussian Processes and PDMs 

We start by loading a shape model (or PDM) of faces : 


```scala mdoc
val faceModel = StatisticalModelIO.readStatisticalMeshModel(new File("datasets/bfm.h5")).get
val modelGroup = ui.createGroup("model")
```

This models a **probability distribution of face meshes**. We can obtain concrete face meshes by sampling from it:

```scala mdoc
val sampledFace : TriangleMesh[_3D] = faceModel.sample

val sampleGroup = ui.createGroup("samples")
ui.show(sampleGroup, sampledFace, "randomFace")
```

or we can obtain its mean:
 
```scala mdoc
val meanFace : TriangleMesh[_3D] = faceModel.mean
ui.show(sampleGroup, meanFace, "meanFace")
```


##### The GP behind the PDM: 

A PDM can be seen as a triangle Mesh (called the reference mesh) on which a Gaussian Process over deformation fields
is defined. Consequently, we can obtain from the face model the reference mesh:
```scala mdoc
val reference : TriangleMesh[_3D] = faceModel.reference
```
as well as the Gaussian Process (GP)
```scala mdoc
val faceGP : DiscreteLowRankGaussianProcess[_3D, UnstructuredPointsDomain[_3D], SpatialVector[_3D]] = faceModel.gp
```
The type signature of the GP looks slightly scary. If we recall that a Gaussian process is a distribution over functions, 
we can, however, rather easily make sense of the individual bits. 
The type signature tells us that:
- It is a DiscreteGaussianProcess. This means, the function, which the process models are defined on a discrete, finite set of points)
- It is defined in 3D Space (indicated by the type parameter ```_3D```)
- Its domain of the modeled functions is a ```UnstructuredPointsDomain``` (namely the points of the reference mesh)
- The values of the modeled functions are vectors (more precisely, they are of type ```SpatialVectors```).
- It is represented using a low-rank approximation. This is a technicality, which we will come back to later.  
    
Consequently, when we draw samples or obtain the mean from the Gaussian process, we expect to obtain functions with a matching 
signature. This is indeed the case  
  
```scala mdoc
val meanDeformation : DiscreteField[_3D, UnstructuredPointsDomain[_3D], SpatialVector[_3D]] = faceGP.mean
val sampleDeformation : DiscreteField[_3D, UnstructuredPointsDomain[_3D], SpatialVector[_3D]] = faceGP.sample
```

Let's visualize the mean:
```scala mdoc
ui.show(sampleGroup, meanDeformation, "meanField)
``` 

##### Exercise : make everything invisible in the 3D scene, except for "meanField" and "meanFace". Now zoom in (right click and drag) on the vector field. Where are the tips of the vectors ending?

As you hopefully see, all the tips of the mean deformation vectors end on points of the mean face. 

To find out where they start from, let's display the face model's reference mesh : 
```tut:silent
ui.show(modelGroup, referenceFace, "referenceFace")
```

##### Exercise : Zoom in on the scene and observe the deformation field. Where are the vectors starting

As you can see, the mean deformation field of the Gaussian Process contained in our face model 
is a deformation from the reference mesh of the model into the mean face mesh.

Hence when calling *faceModel.mean*, what is really happening is 

1. the mean deformation field is obtained (by calling *faceModel.gp.mean*) 
2. the mean deformation field is then used to deform the reference mesh (*faceModel.referenceMesh*) 
into the triangle Mesh representing the mean face

The same is happening when randomly sampling from the face model :

1. a random deformation field is sampled (*faceModel.gp.sample*)
2. the deformation field is applied to the reference mesh to obtain a random face mesh


##### Exercise : Perform the 2 steps above in order to sample a random face (that is sample a random deformation first, then use it to warp the reference mesh).

```scala mdoc:invisible
gui.close
```
# Rigid alignment

In this tutorial we show how rigid alignment of shapes can be performed in Scalismo.

##### Related resources

The following resources from our [online course](https://www.futurelearn.com/courses/statistical-shape-modelling) may provide
some helpful context for this tutorial:

- Learning a model from example data [(Video)](https://www.futurelearn.com/courses/statistical-shape-modelling/3/steps/250329)
- Superimposing shapes [(Article)](https://www.futurelearn.com/courses/statistical-shape-modelling/3/steps/250330)  

##### Preparation
As in the previous tutorials, we start by importing some commonly used objects and initializing the system. 

```scala mdoc:silent
import scalismo.geometry._
import scalismo.common._
import scalismo.ui.api._
import scalismo.mesh.TriangleMesh
import scalismo.io.MeshIO

scalismo.initialize()
implicit val rng = scalismo.utils.Random(42)
```

### Quick view on Transformations

Let's start by loading and showing Paola's mesh again:

```scala mdoc:silent
val ui = ScalismoUI()
val paolaGroup = ui.createGroup("paola")
val mesh : TriangleMesh[_3D] = MeshIO.readMesh(new java.io.File("datasets/Paola.stl")).get
val meshView = ui.show(paolaGroup, mesh, "Paola")
``` 

Scalismo allows us to perform geometric transformations on meshes.

Transformations are *functions* that map a given point, into a new *transformed* point.
We find the transformations in the package ```scalismo.registration```. 
Lets import the classes in this package
```scala mdoc:silent
import scalismo.registration.{Transformation, RotationTransform, TranslationTransform, RigidTransformation}
```


The most general way to define a transformation is by specifying the function, which is applied directly.
The following example shows a transformation, which simply flips the point along the x axis. 

```scala mdoc:silent
val flipTransform = Transformation((p : Point[_3D]) => Point(-p.x, p.y, p.z))
```

When given a point as an argument, the defined transform will then simply return a new point:

```scala mdoc:silent
val pt : Point[_3D] = flipTransform(Point(1.0, 1.0, 1.0))
```

An important class of Transformations are the rigid transformation, i.e. a rotation followed by a translation. Due to their 
importance, these transformations are readily defined in scalismo. 

To perform a  Translation, we define the following transformation:

```scala mdoc:silent
val translation = TranslationTransform[_3D](Vector(100,0,0))
```

For defining a rotation, we define the 3 [Euler angles](https://en.wikipedia.org/wiki/Euler_angles) , as well as the center of rotation.
```scala mdoc:silent
val rotationCenter = Point(0.0, 0.0, 0.0)
val rotation : RotationTransform[_3D] = RotationTransform(0f,3.14f,0f, rotationCenter)
```
This transformation rotates every point with approximately 180 degrees around the Y axis (centered at the origin of the space). 

```scala mdoc:silent
val pt2 : Point[_3D] = rotation(Point(1,1,1))
```

In Scalismo, such transformations can be applied not only to single points, but most collections of points such as triangle meshes, can be 
transformed by invoking the method ```transform``` on the respective object.

```scala mdoc:silent
val translatedPaola : TriangleMesh[_3D] = mesh.transform(translation)
val paolaMeshTranslatedView = ui.show(paolaGroup, translatedPaola, "translatedPaola")
```

Here, we used the *transform* method of the TriangleMesh class. This method takes as input a transformation function that maps a 3D point into another 3D point and applies it to all the points of the mesh while maintaining the same triangulation. As a result, we obtain a new translated version of Paola

### Composing transformations

Simple transformations can be composed to more complicated ones using the ```compose``` method. For example, we can define a rigid 
tranformation as a composition of translation and rotation:
```scala mdoc:silent
val rigidTransform1 = translation.compose(rotation)
```

In Scalismo, rigid transformations are also already predefined. We could have written instead:

```scala mdoc:silent
val rigidTransform2 : RigidTransformation[_3D] = RigidTransformation[_3D](translation, rotation)
```


##### Exercise: Apply the rotation transform to the original mesh of Paola and show the result
##### Note: since the rotation is around the origin, you might have to zoom out (hold right click and drag down) to see the result.


### Rigid alignment

A task that we need to perform in any shape modelling pipeline, is the rigid alignment of objects; I.e. normalizing the pose of 
an object with respect to some reference. 

To illustrate this procedure, we consider the mesh of Paola, which we loaded, and translate it rigidly using the rigid transformation defined above. 
```scala mdoc:silent
val paolaTransformedGroup = ui.createGroup("paolaTransformed")
val paolaTransformed = mesh.transform(rigidTransform2)
ui.show(paolaTransformedGroup, paolaTransformed, "paolaTransformed")
```

In the following we assume that we do not know the parameters of the translation and rotation that led to *rigidPaola*.

**How can we retrieve those parameters and obtain a transformation from the original mesh to *rigidPaola* ?**

Rigid alignment is easiest if we already know some corresponding points in both shapes. Assume for the moment, that we 
have identified a few corresponding points and marked them using landmarks. We can then apply *Procrustes Analysis*. 
Usually, these landmarks would need to be clicked manually in a GUI framework. To simplify this tutorial, we exploit that the two meshes
are the same and hence have the same point ids. We can thus define landmarks programmatically:

```scala mdoc:silent
val ptIds = Seq(PointId(2213), PointId(14727), PointId(8320), PointId(48182))
val paolaLandmarks = ptIds.map(pId => Landmark(s"lm-${pId.id}", mesh.pointSet.point(pId)))
val paolaTransformedLandmarks = ptIds.map(pId => Landmark(s"lm-${pId.id}", paolaTransformed.pointSet.point(pId)))

val paolaLandmarkViews = paolaLandmarks.map(lm => ui.show(paolaGroup, lm, s"${lm.id}"))
val paolaTransformedLandmarkViews = paolaTransformedLandmarks.map(lm => ui.show(paolaTransformedGroup, lm, lm.id))
```

Given this lists of landmarks, we can apply Procrustes analysis to retrieve the best rigid transformation from the original set of landmarks to the new one as follows:

```scala mdoc:silent
import scalismo.registration.LandmarkRegistration

val bestTransform : RigidTransformation[_3D] = LandmarkRegistration.rigid3DLandmarkRegistration(paolaLandmarks, paolaTransformedLandmarks, center = Point(0, 0, 0))
```

The resulting transformation is the best possible rigid transformation (with rotation center ```Point(0,0,0)```) from ```paolaLandmarks``` to ```paolaTransformedLandmarks```.
Best here means, that it minimizes the mean squared error over the landmark points. 

Let's now apply it to the original set of landmarks, to see how well they are transformed : 

```scala mdoc:silent
val transformedLms = paolaLandmarks.map(lm => lm.transform(bestTransform))
val landmarkViews = ui.show(paolaGroup, transformedLms, "transformedLMs")
```

##### Question: Are the transformed landmarks matching *perfectly* the set of *rigidLms*? If not, why not?

Let's now also apply the transform to the entire mesh : 

```scala mdoc:silent
val alignedPaola = mesh.transform(bestTransform)
val alignedPaolaView = ui.show(paolaGroup, alignedPaola, "alignedPaola") 
alignedPaolaView.color = java.awt.Color.RED
```

##### Question: Is the transformed mesh matching *perfectly rigidPaola*? If not, why not?


##### Exercise: Load the mesh file named "datasets/323.stl" and perform a similar alignment of Paola to this mesh. Feel free to pick any set of landmarks you wish.

```scala mdoc:invisible
ui.close()
```
---
layout: post
title:  "vtkProgrammableFilter ain't so bad"
---

When I started preparing for this blog, my goal was to write about how awful `vtkProgrammableFilter`
is to use in Python and how the new vtkPythonAlgorithm is superior. Funny enough, as I worked on
a few examples, I discovered that `vtkProgrammableFilter` is not so bad if you know a few tricks. It
does what it is supposed to fairly well. `vtkPythonAlgorithm` can do a lot more but more on that
later.

In summary, `vtkProgrammableFilter` enables one to execute Python code within a pipeline. It is
designed to perform relatively simple tasks without having to compile C++ classes. Actually, when
combined with the `numpy_interface` module that I previously wrote about, it can do fairly complex
and computationally intensive things too. Let's start with a very simple example.

{% highlight python %}
1   import time, vtk
2   from vtk.numpy_interface import dataset_adapter as dsa
3   from vtk.numpy_interface import algorithms as alg
4   
5   def create_scene(flt):
6       m = vtk.vtkPolyDataMapper()
7       m.SetInputConnection(flt.GetOutputPort())
8   
9       a = vtk.vtkActor()
10      a.SetMapper(m)
11  
12      ren = vtk.vtkRenderer()
13      ren.AddActor(a)
14  
15      renWin = vtk.vtkRenderWindow()
16      renWin.AddRenderer(ren)
17      renWin.SetSize(600, 600)
18  
19      return renWin
20  
21  s = vtk.vtkSphereSource()
22  
23  def execute():
24      inp = dsa.WrapDataObject(pf.GetInput())
25      opt = dsa.WrapDataObject(pf.GetOutput())
26      opt.ShallowCopy(inp.VTKObject)
27      opt.PointData.append(inp.PointData['Normals'][:, 0], 'Normals-x')
28      opt.PointData.SetActiveScalars('Normals-x')
29  
30  pf = vtk.vtkProgrammableFilter()
31  pf.SetInputConnection(s.GetOutputPort())
32  pf.SetExecuteMethod(execute)
33  
34  renWin = create_scene(pf)
35  renWin.Render()
36  time.sleep(3)
{% endhighlight %}

Lines 5 - 19 should be pretty obvious to a VTK initiate. They simply create a rendering scene. Line
21 - 32 create a simple pipeline consisting of a `vtkSphereSource` and a `vtkProgrammableFilter`.
The Programmable Filter creates a new scalar array called `Normals-x` by extracting the first
component of the `Normals` vector generated by the Sphere Source (lines 27-28). This function is
assigned to the Programmable Filter on line 32. Pretty standard stuff. Still quite powerful if you
consider that you have almost the entire VTK API plus the `numpy_interface` module.

What I really dislike about this is that on lines 24 and 25, we have to refer to the pf object that
is in the global namespace. So, if we add something like this:

{% highlight python %}
39  pf = None
{% endhighlight %}

We will get the following error during execution:

{% highlight python %}
Traceback (most recent call last):
  File "lame.py", line 34, in execute
    inp = dsa.WrapDataObject(pf.GetInput())
AttributeError: 'NoneType' object has no attribute 'GetInput'
{% endhighlight %}

Since the method passed to `SetExecuteMethod` cannot take any arguments, this seems like the only
solution. It is kind of yucky. What is worse, if you want to re-use this method and tweak it by
setting some parameters, you have to do it by adjusting global variables or write a new function for
each new combination. Ugh.

We can get around the first problem if we can use a closure as follows.

{% highlight python %}
def create_execute(filter):
    pf = filter
    def execute():
        inp = dsa.WrapDataObject(pf.GetInput())
        opt = dsa.WrapDataObject(pf.GetOutput())
        opt.ShallowCopy(inp.VTKObject)
        opt.PointData.append(inp.PointData['Normals'][:, 0], 'Normals-x')
        opt.PointData.SetActiveScalars('Normals-x')
    return execute

pf = vtk.vtkProgrammableFilter()
pf.SetInputConnection(s.GetOutputPort())
pf.SetExecuteMethod(create_execute(pf))

renWin = create_scene(pf)

pf = None

renWin.Render()
{% endhighlight %}

This works fine because `create_execute()` returns a function object which has its argument bound
within the function object's scope (a closure). It still doesn't address the configurability issue.
I didn't think that this problem could be solved. I was wrong. `SetExecuteMethod` accepts any
_callable_ object. So the following works very nicely.

{% highlight python %}
s = vtk.vtkSphereSource()

class MyAlgorithm(object):
    def __init__(self, algorithm):
        import weakref
        self.__Algorithm = weakref.ref(algorithm)

    def __call__(self):
        inp = dsa.WrapDataObject(self.__Algorithm().GetInput())
        opt = dsa.WrapDataObject(self.__Algorithm().GetOutput())
        opt.ShallowCopy(inp.VTKObject)
        opt.PointData.append(inp.PointData['Normals'][:, 0], 'Normals-x')
        opt.PointData.SetActiveScalars('Normals-x')


pf = vtk.vtkProgrammableFilter()
pf.SetInputConnection(s.GetOutputPort())
pf.SetExecuteMethod(MyAlgorithm(pf))
{% endhighlight %}

An instance of a class that implements the `__call__()` method is callable. So we can call a
`MyAlgorithm` as if it is a function:

{% highlight python %}
o = MyAlgorithm(pf)
o()
{% endhighlight %}

Note that I used a weak reference to avoid a reference loop between the Programmable Filter and
`MyAlgorithm`. We can now customize the programmable filter using the class interface.

Consider another example.

{% highlight python %}
1   class MyAlgorithm(object):
2       def __init__(self, algorithm):
3           import weakref
4           self.__Algorithm = weakref.ref(algorithm)
5           self.__Factor = 1
6   
7       def __call__(self):
8           inp = dsa.WrapDataObject(self.__Algorithm().GetInput())
9           opt = dsa.WrapDataObject(self.__Algorithm().GetOutput())
10          opt.ShallowCopy(inp.VTKObject)
11          print self.Factor
12          opt.PointData.append(inp.PointData['Normals'][:, 0] * self.Factor, 'Normals-x')
13          opt.PointData.SetActiveScalars('Normals-x')
14  
15      def SetFactor(self, factor):
16          self.__Factor = factor
17          self.__Algorithm().Modified()
18  
19      def GetFactor(self):
20          return self.__Factor
21  
22      Factor = property(GetFactor, SetFactor)
23  
24  pf = vtk.vtkProgrammableFilter()
25  pf.SetInputConnection(s.GetOutputPort())
26  alg = MyAlgorithm(pf)
27  pf.SetExecuteMethod(alg)
28  pf.Update()
29  
30  alg.Factor = 2
31  
32  renWin = create_scene(pf)
33  renWin.Render()
{% endhighlight %}

Here I added a Factor property to allow the configuration of the filter. In this example, `pf` will
execute twice: the first when Update() is called (with Factor == 1), second during render (with
Factor == 2). Note that `Modified()`  on line 17 is necessary for this to work.

`vtkProgrammableFilter` works great for many use cases but has some limitations:

* Its output type is always the same as its input type. So one cannot implement filters such as a
contour filter (which accepts a `vtkDataSet` and outputs a `vtkPolyData`) using it.
* It is not possible to properly manipulate pipeline execution by setting keys. We'll discuss how
this can be done using `vtkPythonAlgorithm` in a future blog. 
* The source counterpart of `vtkProgrammableFilter`, `vtkProgrammableSource`, is very difficult
to use and is very limited. So if you want to create sources (readers for example), I strongly
recommend using `vtkPythonAlgorithm`.

I'll follow up with one or more blogs on `vtkPythonAlgorithm`.

One final note. The callable objects also work nicely as observers. See below.

{% highlight python %}
import vtk

class MyObserver(object):
    def __call__(self, obj, event):
        print obj.GetClassName(), event

s = vtk.vtkSphereSource()
s.AddObserver('ModifiedEvent', MyObserver())
s.Modified()
{% endhighlight %}

will print

{% highlight python %}
vtkSphereSource ModifiedEvent
{% endhighlight %}

_Note: This article was originally published on the [Kitware blog](http://www.kitware.com/blog/home/post/733).
Please see the [Kitware web site](http://www.kitware.com), the [VTK web site](http://www.vtk.org) and the
[ParaView web site](http://www.paraview.org) for more information._
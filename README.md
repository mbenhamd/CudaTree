CudaTree
==================
**Forked from the original repository written for python 2.7 and support python 3.6 now.<br/>**
**I decide to stop maintaining CudaTree since scikit-learn made a sigificant improvements on its random forest and it's well maintained.**

CudaTree is an implementation of Leo Breiman's [Random Forests](http://www.stat.berkeley.edu/~breiman/RandomForests/cc_home.htm)
adapted to run on the GPU. 
A random forest is an ensemble of randomized decision trees which  vote together to predict new labels.
CudaTree parallelizes the construction of each individual tree in the ensemble and thus is able to train faster than 
the latest version of [scikits-learn](http://scikit-learn.org/stable/modules/tree.html). 

We've also implemented a hybrid version of random forest which uses multiple GPU and multicore CPU to fully utilize all the 
resource your system has. For the multicore version, we use scikits-learn random forest as default, you can also supply other 
multicore implementations such as WiseRF.

### Usage


```python
  import numpy as np
  from cudatree import load_data, RandomForestClassifier

  x_train, y_train = load_data("digits")
  forest = RandomForestClassifier(n_estimators = 50, verbose = True, bootstrap = False)
  forest.fit(x_train, y_train, bfs_threshold = 4196)
  forest.predict(x_train)
```

For hybrid version:
```python
  import numpy as np
  from cudatree import load_data
  from hybridforest import RandomForestClassifier
  from PyWiseRF import WiseRF

  x_train, y_train = load_data("digits")
  #We gonna build random forest on two GPU and 6 CPU core. 
  #For the GPU version we use CudaTree, CPU version we use WiseRF
  forest = RandomForestClassifier(n_estimators=50, 
                                    n_gpus = 2, 
                                    n_jobs = 6, 
                                    bootstrap=False, 
                                    cpu_classifier = WiseRF)
  
  forest.fit(x_train, y_train)
  forest.predict(x_train)
```


### Install 
You should be able to install CudaTree from its [PyPI package](https://pypi.python.org/pypi/cudatree) by running:

    pip install cudatree
    

### Dependencies 

CudaTree is writen for Python 2.7 and depends on:

* [Scikit-learn](http://scikit-learn.org/stable/)
* [Numpy](http://www.scipy.org/install.html)
* [PyCUDA](http://documen.tician.de/pycuda/#)
* [Parakeet](https://pypi.python.org/pypi/parakeet/0.16.2)


### Limitations:

It's important to remember that a dataset which fits into your computer's main memory may not necessarily fit on a GPU's smaller memory. 
Furthermore, CudaTree uses several temporary arrays during tree construction which will limit how much space is available. 
A formula for the total number of bytes required to fit a decision tree for a given dataset is given below. If less than this quantity is available 
on your GPU, then CudaTree will fail. 




<!-- 
\mathrm{GPU}\;\mathrm{memory}\;\mathrm{in}\;\mathrm{bytes} = \mathit{DatasetSize} + 2\cdot \mathit{Samples} \cdot \mathit{Features} \cdot \left\lceil \frac{\log_2 \mathit{Samples}}{8} \right\rceil + \mathit{Features} \cdot \mathit{Samples}
-->
  <div align="center">
  ![gpu memory = dataset + 2*samples*features*ceil(log2(samples)/8) + samples*features](https://raw.github.com/EasonLiao/CudaTree/master/doc/gpumem.png) 
  </div>

<!--     
  <i>(n_bytes_per_idx is 1 when the number of samples <= 256
  <br />
  n_bytes_per_idx is 2 when the number of samples <= 65536
  <br />
  n_bytes_per_idx is 4 when the number of samples <= 4294967296 
  <br />
  n_bytes_per_idx is 8 when the number of samples > 4294967296)</i>
  <br/>
 --> 
 
  For example, let's assume you have a training dataset which takes up 200MB, and the number of samples = 10000 and 
  the number of features is 3000, then the total GPU memory required will be: <br>
  <div align="center" style="font-style:italic;">
  200MB + (2 * 3000 * 10000 * 2 + 3000 * 10000) / 1024 / 1024 = 314MB
  </div>

In addition to memory requirement, there are several other limitations hard-coded into CudaTree: 

* The maximum number of features allowed is 65,536.
* The maximum number of categories allowed is 5000(CudaTree performs best when the number of categories is <=100).
* Your NVIDIA GPU must have compute capability >= 1.3.
* Currently, the only splitting criterion is GINI impurity, which means CudaTree can't yet do regression (splitting by variance for continuous outputs is planned)

Since scikit-learn changed their impelementation of random forest. So we may be slower than scikit-learn on some of the dataset. But any way. You can always get the performance boost if you use the hybrid mode.


### Implementation Details 

Trees are first constructed in depth-first order, with a separate kernel launch for each node's subset of the data. 
Eventually the data gets split into very small subsets and at that point CudaTree switches to breadth-first grouping
of multiple subsets for each kernel launch. 



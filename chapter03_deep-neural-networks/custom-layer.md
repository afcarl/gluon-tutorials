# Designing a custom layer with ``gluon`` 

Now that we've peeled back some of the syntactic sugar conferred by ``nn.Sequential()`` and given you a feeling for how ``gluon`` works under the hood, you might feel more comfortable when writing your high-level code. But the real reason to get to know ``gluon`` more intimately is so that we can mess around with it and write our own Blocks. 

Up until now, we've presented two versions of each tutorial. One from scratch and one in ``gluon``. Empowered with such independence, you might be wondering, "if I wanted to write my own layer, why wouldn't I just do it from scratch?" 

In reality, writing every model completely from scratch can be cumbersome.  Just like there's only so many times a developer can code up a blog from scratch without hating life, there's only so many times that you'll want to write out a convolutional layer, or define the stochastic gradient descent updates. Even in a pure research environment, we usually want to customize one part of the model. For example, we might want to implement a new layer, but still rely on other common layers, loss functions, optimizers, etc. In some cases it might be nontrivial to compute the gradient efficiently and the automatic differentiation subsystem might need some help: When was the last time you performed backprop through a log-determinant, a Cholesky factorization, or a matrix exponential? In other cases things might not be numerically very stable when calculated straightforwardly (e.g. taking logs of exponentials of some arguments). 

By hacking ``gluon``, we can get the desired flexibility in one part of our model, without screwing up everything else that makes our life easy.

```{.python .input}
from __future__ import print_function
import mxnet as mx
import numpy as np
from mxnet import nd, autograd, gluon
from mxnet.gluon import nn, Block
mx.random.seed(1)
import sys
sys.path.append('..')
import utils

###########################
#  Speficy the context we'll be using
###########################
ctx = mx.cpu()

###########################
#  Load up our dataset
###########################
batch_size = 64
train_data, test_data = utils.load_mnist(batch_size)
```

## Defining a (toy) custom layer

To start, let's pretend that we want to use ``gluon`` for its optimizer, serialization, etc, but that we need a new layer. Specifically, we want a layer that centers its input about 0 by subtracting its mean. We'll go ahead and define the simplest possible ``Block``. Remember from the last tutorial that in ``gluon`` a layer is called a ``Block`` (after all, we might compose multiple blocks into a larger block, etc.).

```{.python .input}
class CenteredLayer(Block):
    def __init__(self, **kwargs):
        super(CenteredLayer, self).__init__(**kwargs)
        
    def forward(self, x):
        return x - nd.mean(x)
```

That's it. We can just instantiate this block and make a forward pass. 
Note that this layer doesn't actually care 
what its input or output dimensions are. 
So we can just feed in an arbitrary array 
and should expect appropriately transformed output. Whenever we are happy with whatever the automatic differentiation generates, this is all we need.

```{.python .input}
net = CenteredLayer()
net(nd.array([1,2,3,4,5]))
```

We can also incorporate this layer into a more complicated network, as by using ``nn.Sequential()``.

```{.python .input}
net2 = nn.Sequential()
net2.add(nn.Dense(128))
net2.add(nn.Dense(10))
net2.add(CenteredLayer())
```

This network contains Blocks (Dense) that contain parameters and thus require initialization

```{.python .input}
net2.collect_params().initialize(mx.init.Xavier(magnitude=2.24), ctx=ctx)
```

Now we can pass some data through it, say the first image from our MNIST dataset.

```{.python .input}
for data, _ in train_data:
    data = data.as_in_context(ctx)
    break
output = net2(data[0:1])
print(output)
```

And we can verify that as expected, the resulting vector has mean 0.

```{.python .input}
nd.mean(output)
```

There's a good chance you'll see something other than 0. When I ran this code, I got ``2.68220894e-08``. 
That's roughly ``.000000027``. This is due to the fact that MXNet often uses low precision arithmetics. 
For deep learning research, this is often a compromise that we make.
In exchange for giving up a few significant digits, we get tremendous speedups on modern hardware.
And it turns out that most deep learning algorithms don't suffer too much from the loss of precision.

## Custom layers with parameters

While ``CenteredLayer`` should give you some sense of how to implement a custom layer, it's missing a few important pieces. Most importantly, ``CenteredLayer`` doesn't care about the dimensions of its input or output, and it doesn't contain any trainable parameters. Since you already know how to implement a fully-connected layer from scratch, let's learn how to make parametric ``Block`` by implementing MyDense, our own version of a fully-connected (Dense) layer.

## Parameters

Before we can add parameters to our custom ``Block``, we should get to know how ``gluon`` deals with parameters generally. Instead of working with NDArrays directly, each ``Block`` is associated with some number (as few as zero) of ``Parameter`` (groups). 

At a high level, you can think of a ``Parameter`` as a wrapper on an ``NDArray``. However, the ``Parameter`` can be instantiated before the corresponding NDArray is. For example, when we instantiate a ``Block`` but the shapes of each parameter still need to be inferred, the Parameter will wait for the shape to be inferred before allocating memory. 

To get a hands-on feel for mxnet.Parameter, let's just instantiate one outside of a ``Block``:

```{.python .input}
my_param = gluon.Parameter("exciting_parameter_yay", grad_req='write', shape=(5,5))
print(my_param)
```

Here we've instantiated a parameter, giving it the name "exciting_parameter_yay". We've also specified that we'll want to capture gradients for this Parameter. Under the hood, that lets ``gluon`` know that it has to call ``.attach_grad()`` on the underlying NDArray. We also specified the shape. Now that we have a Parameter, we can initialize its values via ``.initialize()`` and extract its data by calling ``.data()``.

```{.python .input}
my_param.initialize(mx.init.Xavier(magnitude=2.24), ctx=ctx)
print(my_param.data())
```

For data parallelism, a Parameter can also be initialized on multiple contexts. The Parameter will then keep a copy of its value on each context. Keep in mind that you need to maintain consistency about the copies when updating the Parameter (usually `gluon.Trainer` does this for you).

Note that you need at least two GPUs to run this section.

```{.python .input}
# my_param = gluon.Parameter("exciting_parameter_yay", grad_req='write', shape=(5,5))
# my_param.initialize(mx.init.Xavier(magnitude=2.24), ctx=[mx.gpu(0), mx.gpu(1)])
# print(my_param.data(mx.gpu(0)), my_param.data(mx.gpu(1)))
```

## Parameter dictionaries (introducing ``ParameterDict``)

Rather than directly store references to each of its ``Parameters``, ``Block``s typicaly contain a parameter dictionary (``ParameterDict``). In practice, we'll rarely instantiate our own ``ParameterDict``. That's because whenever we call the ``Block`` constructor it's generated automatically. For pedagogical purposes, we'll do it from scratch this one time.

```{.python .input}
pd = gluon.ParameterDict(prefix="block1_")
```

MXNet's ``ParameterDict`` does a few cool things for us. First, we can instantiate a new Parameter by calling ``pd.get()``

```{.python .input}
pd.get("exciting_parameter_yay", grad_req='write', shape=(5,5))
```

Note that the new parameter is (i) contained in the ParameterDict and (ii) appends the prefix to its name. This naming convention helps us to know which parameters belong to which ``Block`` or sub-``Block``. It's especially useful when we want to write parameters to disc (i.e. serialize), or read them from disc (i.e. deserialize).

Like a regular Python dictionary, we can get the names of all parameters with ``.keys()`` and can access parameters with:

```{.python .input}
pd["block1_exciting_parameter_yay"]
```

## Craft a bespoke fully-connected ``gluon`` layer

Now that we know how parameters work, we're ready to create our very own fully-connected layer. We'll use the familiar relu activation from previous tutorials.

```{.python .input}
def relu(X):
    return nd.maximum(X, 0)
```

Now we can define our ``Block``.

```{.python .input}
class MyDense(Block):
    ####################
    # We add arguments to our constructor (__init__)
    # to indicate the number of input units (``in_units``) 
    # and output units (``units``)
    ####################
    def __init__(self, units, in_units=0, **kwargs):
        super(MyDense, self).__init__(**kwargs)
        with self.name_scope():
            self.units = units
            self._in_units = in_units
            #################
            # We add the required parameters to the ``Block``'s ParameterDict , 
            # indicating the desired shape
            #################
            self.weight = self.params.get(
                'weight', init=mx.init.Xavier(magnitude=2.24), 
                shape=(in_units, units))
            self.bias = self.params.get('bias', shape=(units,))        

    #################
    #  Now we just have to write the forward pass. 
    #  We could rely upong the FullyConnected primitative in NDArray, 
    #  but it's better to get our hands dirty and write it out
    #  so you'll know how to compose arbitrary functions
    #################
    def forward(self, x):
        with x.context:
            linear = nd.dot(x, self.weight.data()) + self.bias.data()
            activation = relu(linear)
            return activation
```

Recall that every Block can be run just as if it were an entire network. 
In fact, linear models are nothing more than neural networks 
consisting of a single layer as a network.
    
So let's go ahead and run some data though our bespoke layer.
We'll want to first instantiate the layer and initialize its parameters.

```{.python .input}
dense = MyDense(20, in_units=10)
dense.collect_params().initialize(ctx=ctx)
```

```{.python .input}
dense.params
```

Now we can run through some dummy data.

```{.python .input}
dense(nd.ones(shape=(2,10)))
```

## Using our layer to build an MLP

While it's a good sanity check to run some data though the layer, the real proof that it works will be if we can compose a network entirely out of ``MyDense`` layers and achieve respectable accuracy on a real task. So we'll revisit the MNIST digit classification task, and use the familiar ``nn.Sequential()`` syntax to build our net.

```{.python .input}
net = gluon.nn.Sequential()
with net.name_scope():
    net.add(MyDense(128, in_units=784))
    net.add(MyDense(64, in_units=128))
    net.add(MyDense(10, in_units=64))
```

## Initialize Parameters

```{.python .input}
net.collect_params().initialize(ctx=ctx)
```

## Instantiate a loss

```{.python .input}
loss = gluon.loss.SoftmaxCrossEntropyLoss()
```

## Optimizer

```{.python .input}
trainer = gluon.Trainer(net.collect_params(), 'sgd', {'learning_rate': .1})
```

## Evaluation Metric

```{.python .input}
metric = mx.metric.Accuracy()

def evaluate_accuracy(data_iterator, net):
    numerator = 0.
    denominator = 0.
    
    for i, (data, label) in enumerate(data_iterator):
        with autograd.record():
            data = data.as_in_context(ctx).reshape((-1,784))
            label = label.as_in_context(ctx)
            label_one_hot = nd.one_hot(label, 10)
            output = net(data)
        
        metric.update([label], [output])
    return metric.get()[1]
```

## Training loop

```{.python .input}
epochs = 2  # Low number for testing, set higher when you run!
moving_loss = 0.

for e in range(epochs):
    for i, (data, label) in enumerate(train_data):
        data = data.as_in_context(ctx).reshape((-1,784))
        label = label.as_in_context(ctx)
        with autograd.record():
            output = net(data)
            cross_entropy = loss(output, label)
            cross_entropy.backward()
        trainer.step(data.shape[0])
            
    test_accuracy = evaluate_accuracy(test_data, net)
    train_accuracy = evaluate_accuracy(train_data, net)
    print("Epoch %s. Train_acc %s, Test_acc %s" % (e, train_accuracy, test_accuracy))
```

## Conclusion

It works! There's a lot of other cool things you can do. In more advanced chapters, we'll show how you can make a layer that takes in multiple inputs, or one that cleverly calls down to MXNet's symbolic API to squeeze out extra performance without screwing up your convenient imperative workflow. 

## Next
[Serialization: saving your models and parameters for later re-use](../chapter03_deep-neural-networks/serialization.ipynb)

For whinges or inquiries, [open an issue on  GitHub.](https://github.com/zackchase/mxnet-the-straight-dope)

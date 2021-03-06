# libnn

Tests: [![CircleCI](https://circleci.com/gh/mrpossoms/libnn.svg?style=svg)](https://circleci.com/gh/mrpossoms/libnn)

libnn is an intuitive feed-forward neural network library designed with embedded linux systems in mind. It's intended to instantiate a trained model for performing predictions in the field.

## Requirements
* POSIX compliant OS
* Python 2.7+ (for running tests)
* gcc
* gnu make

## Installation

```
$ make install
```
Will build a static library, and copy it and the header file to `/usr/local/lib` and `/usr/local/include` respectively.

## Usage

libnn defines two primary struct types to be aware of, _mat_t_ and _nn_layer_t_. _mat_t_ serves as a description and container for either and NxM matrix or a tensor. Allocating a zero filled matrix is as easy as the following.

```C
mat_t M = {
	.dims = { 4, 4 }
};

nn_mat_init(&M); // returns 0 on success
```

### _Network Declaration_

A libnn neural network is composed of _nn_layer_t_ instances, which in turn contain a handful of matrices. When defining a network architecture there are only a few that you need to be concerned with. `w` the connection weights, `b` the biases, and `A`, a pointer to the vector of activations output for that layer. The network should be defined as an array of _nn_layer_t_ instances, with the final layer being empty to act as a terminator. The flow of activations follows the order of the layers defined in the array. Each _nn_layer_t_ contains an function pointer called `activation` which you can customize per layer. libnn contains several builtin activation functions _nn_act_sigmoid_, _nn_act_relu_ and _nn_act_softmax_. 

Here's an example of a two layer fully connected network. Note: the net in this example would be untrained, and useless. See 'Loading a Trained Model' below.

```C
mat_t x = {
	.dims = { 1, 768 },
};
nn_mat_init(&x);

nn_layer_t L[] = {
	{ // Layer 0
		.w = { .dims = { 256, 768 } }, // shape of layer 0's weight matrix
		.b = { .dims = { 256, 1 } },   // shape of layer 0's bias matrix
		.activation = nn_act_relu      // pointer to layer 0's activation function
	},
	{ // Layer 1 (output layer)
		.w = { .dims = { 3, 256 } }, // shape of layer 1's weight matrix
		.b = { .dims = { 3, 1 } },   // shape of layer 1's bias matrix
		.activation = nn_act_softmax // pointer to layer 1's activation function
	},
	{} // terminator
};

nn_init(L, &x); // returns 0 on success
```

_nn_layer_t_ can represent either a fully connected layer, or a 2d-convolutional layer. When defining a convolutional layer, you must also set the `pool` and `filter` structs. `filter` is a _conv_op_t_ struct, and needs the following values set.
```C
.filter = {
	.kernel = {
		w, h // integers
	},

	.stride = {
		row, col // integers
	},

	.padding = // PADDING_VALID or PADDING_SAME
}
```

`pool` is an anonymous struct and can be defined optionally if a pooling operation is desired.

```C
.pool = {
	.op = {
		.kernel = {
			w, h // integers
		},

		.stride = {
			row, col // integers
		},
	},
	.type = POOLING_MAX,
},
```


### _Loading a Trained Model_

In practice, however, you would want to load stored weights and biases from files using `nn_mat_load`. Below is an example of instantiating a fully connected network.

```c
nn_layer_t L[] = {
	{
		.w = nn_mat_load("data/dense.kernel"),
		.b = nn_mat_load("data/dense.bias"),
		.activation = nn_act_relu
	},
	{
		.w = nn_mat_load("data/dense_1.kernel"),
		.b = nn_mat_load("data/dense_1.bias"),
		.activation = nn_act_softmax
	},
	{} // NOTE: don't forget the terminator
};
```

And here's the equivalent for a simple CNN operating on a 9x9x1 image.

```c
mat_t x = {
	.dims = { 9, 9, 1 },
};
nn_mat_init(&x);

nn_layer_t L[] = {
	{
		.w = nn_mat_load("data/model1/c0.kernel"),
		.b = nn_mat_load("data/model1/c0.bias"),
		.activation = nn_act_relu,
		.filter = {
			.kernel = { 3, 3 },
			.stride = { 1, 1 },
			.padding = PADDING_VALID,
		},
	},
	{
		.w = nn_mat_load("data/model1/c1.kernel"),
		.b = nn_mat_load("data/model1/c1.bias"),
		.activation = nn_act_softmax,
		.filter = {
			.kernel = { 7, 7 },
			.stride = { 1, 1 },
			.padding = PADDING_VALID,
		},
	},
	{} // NOTE: don't forget the terminator
};

assert(nn_init(L, &x) == 0);
```


Matrices loaded by `nn_mat_load` are stored in a simple binary format. With a header starting with a 1 byte integer stating the number of dimensions, and the equivalent number of 4 byte integers following it.

```
[ ui8 num_dimensions | ui32 dim 0 | ... | ui32 dim num_dimensions - 1 ]

```
After the header, the remainder of the matrix consists of a number of 32 bit floats equivalent to the product of the dimensions in the header. The default matrix indexer assumes they are stored in row major order.

If you happen to be a Tensorflow user, you could use the following function to store the weights and biases of a trained Estimator in the format described above.

```Python
def export_model(model):
    for param_name in model.get_variable_names():
        comps = param_name.split('/')

        if len(comps) < 2: continue
        if comps[-1] in ['kernel', 'bias']:
            with open('/var/model/' + param_name.replace('/', '.'), mode='wb') as file:
                param = model.get_variable_value(param_name)
                shape = param.shape

                file.write(struct.pack('b', len(shape)))
                for d in shape:
                    file.write(struct.pack('i', d))

                if len(param.shape) == 4:
                    for f in range(param.shape[3]):
                        filter = param[:,:,:,f].T

                        for w in filter.flatten():
                            file.write(struct.pack('f', w))
                else:
                    for w in param.flatten():
                        file.write(struct.pack('f', w))
```
If you are instead using the TensorFlow low-level Python API, serialization can be performed with this function for each evaluated parameter tensor. E.g. `m = sess.run(tensor, feed_dict={...})`

```Python

def serialize_matrix(m, fp):
    """
    Writes a numpy array into fp in the simple format that
    libnn's nn_mat_load() function understands
    :param m: numpy matrix
    :param fp: file stream
    :return: void
    """
    import struct

    # write the header
    fp.write(struct.pack('b', len(m.shape)))
    for d in m.shape:
        fp.write(struct.pack('i', d))

    # followed by each element
    for e in m.flatten():
        fp.write(struct.pack('f', e))
```

### _Making Predictions_

A prediction can be carried out by the network with a call to `nn_predict` like so. `nn_predict` returns a pointer to the set of activations produced by the final layer, which is the output of the network.

```C
mat_t* y = nn_predict(L, &x);

float* p = y->data.f;
printf("predictions: %f %f %f\n", p[0], p[1], p[2])
```

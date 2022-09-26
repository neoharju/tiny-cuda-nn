# Tiny CUDA Neural Networks ![](https://github.com/NVlabs/tiny-cuda-nn/workflows/CI/badge.svg)

This is a small, self-contained framework for training and querying neural networks. Most notably, it contains a lightning fast ["fully fused" multi-layer perceptron](https://raw.githubusercontent.com/NVlabs/tiny-cuda-nn/master/data/readme/fully-fused-mlp-diagram.png) ([technical paper](https://tom94.net/data/publications/mueller21realtime/mueller21realtime.pdf)), a versatile [multiresolution hash encoding](https://raw.githubusercontent.com/NVlabs/tiny-cuda-nn/master/data/readme/multiresolution-hash-encoding-diagram.png) ([technical paper](https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.pdf)), as well as support for various other input encodings, losses, and optimizers.

## Performance

![Image](data/readme/fully-fused-vs-tensorflow.png)
_Fully fused networks vs. TensorFlow v2.5.0 w/ XLA. Measured on 64 (solid line) and 128 (dashed line) neurons wide multi-layer perceptrons on an RTX 3090. Generated by `benchmarks/bench_ours.cu` and `benchmarks/bench_tensorflow.py` using `data/config_oneblob.json`._


## Usage

Tiny CUDA neural networks have a simple C++/CUDA API:

```cpp
#include <tiny-cuda-nn/common.h>

// Configure the model
nlohmann::json config = {
	{"loss", {
		{"otype", "L2"}
	}},
	{"optimizer", {
		{"otype", "Adam"},
		{"learning_rate", 1e-3},
	}},
	{"encoding", {
		{"otype", "HashGrid"},
		{"n_levels", 16},
		{"n_features_per_level", 2},
		{"log2_hashmap_size", 19},
		{"base_resolution", 16},
		{"per_level_scale", 2.0},
	}},
	{"network", {
		{"otype", "FullyFusedMLP"},
		{"activation", "ReLU"},
		{"output_activation", "None"},
		{"n_neurons", 64},
		{"n_hidden_layers", 2},
	}},
};

using namespace tcnn;

auto model = create_from_config(n_input_dims, n_output_dims, config);

// Train the model
GPUMatrix<float> training_batch_inputs(n_input_dims, batch_size);
GPUMatrix<float> training_batch_targets(n_output_dims, batch_size);

for (int i = 0; i < n_training_steps; ++i) {
	generate_training_batch(&training_batch_inputs, &training_batch_targets); // <-- your code

	float loss;
	model.trainer->training_step(training_batch_inputs, training_batch_targets, &loss);
	std::cout << "iteration=" << i << " loss=" << loss << std::endl;
}

// Use the model
GPUMatrix<float> inference_inputs(n_input_dims, batch_size);
generate_inputs(&inference_inputs); // <-- your code

GPUMatrix<float> inference_outputs(n_output_dims, batch_size);
model.network->inference(inference_inputs, inference_outputs);
```


## Example: learning a 2D image

We provide a sample application where an image function _(x,y) -> (R,G,B)_ is learned. It can be run via
```sh
tiny-cuda-nn/build$ ./mlp_learning_an_image ../data/images/albert.jpg ../data/config_hash.json
```
producing an image every 1000 training steps. Each 1000 steps should take roughly 0.42 seconds with the default configuration on an RTX 3090.

| 10 steps (4.2 ms) | 100 steps (42 ms) | 1000 steps (420 ms) | Reference image |
|:---:|:---:|:---:|:---:|
| ![10steps](data/readme/10.jpg) | ![100steps](data/readme/100.jpg) | ![1000steps](data/readme/1000.jpg) | ![reference](data/images/albert.jpg) |



## Requirements

- An __NVIDIA GPU__; tensor cores increase performance when available. All shown results come from an RTX 3090.
- A __C++14__ capable compiler. The following choices are recommended and have been tested:
  - __Windows:__ Visual Studio 2019
  - __Linux:__ GCC/G++ 7.5 or higher
- __[CUDA](https://developer.nvidia.com/cuda-toolkit) v10.2 or higher__ and __[CMake](https://cmake.org/) v3.21 or higher__.
- The fully fused MLP component of this framework requires a __very large__ amount of shared memory in its default configuration. It will likely only work on an RTX 3090, an RTX 2080 Ti, or high-end enterprise GPUs. Lower end cards must reduce the `n_neurons` parameter or use the `CutlassMLP` (better compatibility but slower) instead.

If you are using Linux, install the following packages
```sh
sudo apt-get install build-essential git
```

We also recommend installing [CUDA](https://developer.nvidia.com/cuda-toolkit) in `/usr/local/` and adding the CUDA installation to your PATH.
For example, if you have CUDA 11.4, add the following to your `~/.bashrc`
```sh
export PATH="/usr/local/cuda-11.4/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda-11.4/lib64:$LD_LIBRARY_PATH"
```


## Compilation (Windows & Linux)

Begin by cloning this repository and all its submodules using the following command:
```sh
$ git clone --recursive https://github.com/nvlabs/tiny-cuda-nn
$ cd tiny-cuda-nn
```

Then, use CMake to build the project: (on Windows, this must be in a [developer command prompt](https://docs.microsoft.com/en-us/cpp/build/building-on-the-command-line?view=msvc-160#developer_command_prompt))
```sh
tiny-cuda-nn$ cmake . -B build
tiny-cuda-nn$ cmake --build build --config RelWithDebInfo -j
```


## PyTorch extension

__tiny-cuda-nn__ comes with a [PyTorch](https://github.com/pytorch/pytorch) extension that allows using the fast MLPs and input encodings from within a [Python](https://www.python.org/) context.
These bindings can be significantly faster than full Python implementations; in particular for the [multiresolution hash encoding](https://raw.githubusercontent.com/NVlabs/tiny-cuda-nn/master/data/readme/multiresolution-hash-encoding-diagram.png).

> The overheads of Python/PyTorch can nonetheless be extensive.
> For example, the bundled `mlp_learning_an_image` example is __~2x slower__ through PyTorch than native CUDA.

Begin by setting up a Python 3.X environment with a recent, CUDA-enabled version of PyTorch. Then, invoke
```sh
pip install git+https://github.com/NVlabs/tiny-cuda-nn/#subdirectory=bindings/torch
```

Alternatively, if you would like to install from a local clone of __tiny-cuda-nn__, invoke
```sh
tiny-cuda-nn$ cd bindings/torch
tiny-cuda-nn/bindings/torch$ python setup.py install
```

Upon success, you can use __tiny-cuda-nn__ models as in the following example:
```py
import commentjson as json
import tinycudann as tcnn
import torch

with open("data/config_hash.json") as f:
	config = json.load(f)

# Option 1: efficient Encoding+Network combo.
model = tcnn.NetworkWithInputEncoding(
	n_input_dims, n_output_dims,
	config["encoding"], config["network"]
)

# Option 2: separate modules. Slower but more flexible.
encoding = tcnn.Encoding(n_input_dims, config["encoding"])
network = tcnn.Network(encoding.n_output_dims, n_output_dims, config["network"])
model = torch.nn.Sequential(encoding, network)
```

See `samples/mlp_learning_an_image_pytorch.py` for an example.



## Components

Following is a summary of the components of this framework. [The JSON documentation](DOCUMENTATION.md) lists configuration options.


| Networks | &nbsp; | &nbsp;
| :--- | :---------- | :-----
| Fully fused MLP | `src/fully_fused_mlp.cu` | Lightning fast implementation of small multi-layer perceptrons (MLPs).
| CUTLASS MLP     | `src/cutlass_mlp.cu`     | MLP based on [CUTLASS](https://github.com/NVIDIA/cutlass)' GEMM routines. Slower than fully-fused, but handles larger networks and still is reasonably fast.

| Input encodings | &nbsp; | &nbsp;
| :--- | :---------- | :-----
| Composite | `include/tiny-cuda-nn/encodings/composite.h` | Allows composing multiple encodings. Can be, for example, used to assemble the Neural Radiance Caching encoding [[Müller et al. 2021]](https://tom94.net/).
| Frequency | `include/tiny-cuda-nn/encodings/frequency.h` | NeRF's [[Mildenhall et al. 2020]](https://www.matthewtancik.com/nerf) positional encoding applied equally to all dimensions.
| Grid | `include/tiny-cuda-nn/encodings/grid.h` | Encoding based on trainable multiresolution grids. Used for [Instant Neural Graphics Primitives [Müller et al. 2022]](https://nvlabs.github.io/instant-ngp/). The grids can be backed by hashtables, dense storage, or tiled storage.
| Identity | `include/tiny-cuda-nn/encodings/identity.h` | Leaves values untouched.
| Oneblob | `include/tiny-cuda-nn/encodings/oneblob.h` | From Neural Importance Sampling [[Müller et al. 2019]](https://tom94.net/data/publications/mueller18neural/mueller18neural-v4.pdf) and Neural Control Variates [[Müller et al. 2020]](https://tom94.net/data/publications/mueller20neural/mueller20neural.pdf).
| SphericalHarmonics | `include/tiny-cuda-nn/encodings/spherical_harmonics.h` | A frequency-space encoding that is more suitable to direction vectors than component-wise ones.
| TriangleWave | `include/tiny-cuda-nn/encodings/triangle_wave.h` | Low-cost alternative to the NeRF's encoding. Used in Neural Radiance Caching [[Müller et al. 2021]](https://tom94.net/).

| Losses | &nbsp; | &nbsp;
| :--- | :---------- | :-----
| L1 | `include/tiny-cuda-nn/losses/l1.h` | Standard L1 loss.
| Relative L1 | `include/tiny-cuda-nn/losses/l1.h` | Relative L1 loss normalized by the network prediction.
| MAPE | `include/tiny-cuda-nn/losses/mape.h` | Mean absolute percentage error (MAPE). The same as Relative L1, but normalized by the target.
| SMAPE | `include/tiny-cuda-nn/losses/smape.h` | Symmetric mean absolute percentage error (SMAPE). The same as Relative L1, but normalized by the mean of the prediction and the target.
| L2 | `include/tiny-cuda-nn/losses/l2.h` | Standard L2 loss.
| Relative L2 | `include/tiny-cuda-nn/losses/relative_l2.h` | Relative L2 loss normalized by the network prediction [[Lehtinen et al. 2018]](https://github.com/NVlabs/noise2noise).
| Relative L2 Luminance | `include/tiny-cuda-nn/losses/relative_l2_luminance.h` | Same as above, but normalized by the luminance of the network prediction. Only applicable when network prediction is RGB. Used in Neural Radiance Caching [[Müller et al. 2021]](https://tom94.net/).
| Cross Entropy | `include/tiny-cuda-nn/losses/cross_entropy.h` | Standard cross entropy loss. Only applicable when the network prediction is a PDF.
| Variance | `include/tiny-cuda-nn/losses/variance_is.h` | Standard variance loss. Only applicable when the network prediction is a PDF.

| Optimizers | &nbsp; | &nbsp;
| :--- | :---------- | :-----
| Adam | `include/tiny-cuda-nn/optimizers/adam.h` | Implementation of Adam [[Kingma and Ba 2014]](https://arxiv.org/abs/1412.6980), generalized to AdaBound [[Luo et al. 2019]](https://github.com/Luolc/AdaBound).
| Novograd | `include/tiny-cuda-nn/optimizers/lookahead.h` | Implementation of Novograd [[Ginsburg et al. 2019]](https://arxiv.org/abs/1905.11286).
| SGD | `include/tiny-cuda-nn/optimizers/sgd.h` | Standard stochastic gradient descent (SGD).
| Shampoo | `include/tiny-cuda-nn/optimizers/shampoo.h` | Implementation of the 2nd order Shampoo optimizer [[Gupta et al. 2018]](https://arxiv.org/abs/1802.09568) with home-grown optimizations as well as those by [Anil et al. [2020]](https://arxiv.org/abs/2002.09018).
| Average | `include/tiny-cuda-nn/optimizers/average.h` | Wraps another optimizer and computes a linear average of the weights over the last N iterations. The average is used for inference only (does not feed back into training).
| Batched | `include/tiny-cuda-nn/optimizers/batched.h` | Wraps another optimizer, invoking the nested optimizer once every N steps on the averaged gradient. Has the same effect as increasing the batch size but requires only a constant amount of memory. |
| EMA | `include/tiny-cuda-nn/optimizers/average.h` | Wraps another optimizer and computes an exponential moving average of the weights. The average is used for inference only (does not feed back into training).
| Exponential Decay | `include/tiny-cuda-nn/optimizers/exponential_decay.h` | Wraps another optimizer and performs piecewise-constant exponential learning-rate decay.
| Lookahead | `include/tiny-cuda-nn/optimizers/lookahead.h` | Wraps another optimizer, implementing the lookahead algorithm [[Zhang et al. 2019]](https://arxiv.org/abs/1907.08610).

| Composite | `include/tiny-cuda-nn/optimizers/composite.h` | Allows using several optimizers on different parameters 


## License and Citation

This framework is licensed under the BSD 3-clause license. Please see `LICENSE.txt` for details.

If you use it in your research, we would appreciate a citation via
```bibtex
@software{tiny-cuda-nn,
	author = {M\"uller, Thomas},
	license = {BSD-3-Clause},
	month = {4},
	title = {{tiny-cuda-nn}},
	url = {https://github.com/NVlabs/tiny-cuda-nn},
	version = {1.6},
	year = {2021}
}
```

For business inquiries, please visit our website and submit the form: [NVIDIA Research Licensing](https://www.nvidia.com/en-us/research/inquiries/)


## Publications

This framework powers the following publications:

> __Instant Neural Graphics Primitives with a Multiresolution Hash Encoding__  
> [Thomas Müller](https://tom94.net), [Alex Evans](https://research.nvidia.com/person/alex-evans), [Christoph Schied](https://research.nvidia.com/person/christoph-schied), [Alexander Keller](https://research.nvidia.com/person/alex-keller)  
> _ACM Transactions on Graphics (__SIGGRAPH__), July 2022_  
> __[Website](https://nvlabs.github.io/instant-ngp/)&nbsp;/ [Paper](https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.pdf)&nbsp;/ [Code](https://github.com/NVlabs/instant-ngp)&nbsp;/ [Video](https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.mp4)&nbsp;/ [BibTeX](https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.bib)__

> __Extracting Triangular 3D Models, Materials, and Lighting From Images__  
> [Jacob Munkberg](https://research.nvidia.com/person/jacob-munkberg), [Jon Hasselgren](https://research.nvidia.com/person/jon-hasselgren), [Tianchang Shen](http://www.cs.toronto.edu/~shenti11/), [Jun Gao](http://www.cs.toronto.edu/~jungao/), [Wenzheng Chen](http://www.cs.toronto.edu/~wenzheng/), [Alex Evans](https://research.nvidia.com/person/alex-evans), [Thomas Müller](https://tom94.net), [Sanja Fidler](https://www.cs.toronto.edu/~fidler/)  
> __CVPR (Oral)__, June 2022  
> __[Website](https://nvlabs.github.io/nvdiffrec/)&nbsp;/ [Paper](https://nvlabs.github.io/nvdiffrec/assets/paper.pdf)&nbsp;/ [Video](https://nvlabs.github.io/nvdiffrec/assets/video.mp4)&nbsp;/ [BibTeX](https://nvlabs.github.io/nvdiffrec/assets/bib.txt)__

> __Real-time Neural Radiance Caching for Path Tracing__  
> [Thomas Müller](https://tom94.net), [Fabrice Rousselle](https://research.nvidia.com/person/fabrice-rousselle), [Jan Novák](http://jannovak.info), [Alexander Keller](https://research.nvidia.com/person/alex-keller)  
> _ACM Transactions on Graphics (__SIGGRAPH__), August 2021_  
> __[Paper](https://tom94.net/data/publications/mueller21realtime/mueller21realtime.pdf)&nbsp;/ [GTC talk](https://gtc21.event.nvidia.com/media/Fully%20Fused%20Neural%20Network%20for%20Radiance%20Caching%20in%20Real%20Time%20Rendering%20%5BE31307%5D/1_liqy6k1c)&nbsp;/ [Video](https://tom94.net/data/publications/mueller21realtime/mueller21realtime.mp4)&nbsp;/ [Interactive results viewer](https://tom94.net/data/publications/mueller21realtime/interactive-viewer/)&nbsp;/ [BibTeX](https://tom94.net/data/publications/mueller21realtime/mueller21realtime.bib)__


## Acknowledgments

Special thanks go to the NRC authors for helpful discussions and to [Nikolaus Binder](https://research.nvidia.com/person/nikolaus-binder) for providing part of the infrastructure of this framework, as well as for help with utilizing TensorCores from within CUDA.

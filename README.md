# Image Deduplicator (imagededup)

[![Build Status](https://dev.azure.com/axelspringerai/Public/_apis/build/status/idealo.imagededup?branchName=master)](https://dev.azure.com/axelspringerai/Public/_build/latest?definitionId=1&branchName=master)
[![Build Status](https://travis-ci.org/idealo/imagededup.svg?branch=master)](https://travis-ci.org/idealo/imagededup)
[![Docs](https://img.shields.io/badge/docs-online-brightgreen)](https://idealo.github.io/imagededup/)
[![codecov](https://codecov.io/gh/idealo/imagededup/branch/master/graph/badge.svg)](https://codecov.io/gh/idealo/imagededup)
[![PyPI Version](https://img.shields.io/pypi/v/imagededup)](https://pypi.org/project/imagededup/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://github.com/idealo/imagededup/blob/master/LICENSE)

imagededup is a python package that simplifies the task of finding **exact** and **near duplicates** in an image collection.

<p align="center">
  <img src="readme_figures/mona_lisa.png" width="600" />
</p>

This fork does add `.avif` support. (Does need `pip install pillow_avif_plugin`.)

You can install it with:
```
pip install git+https://github.com/styler00dollar/imagededup.git
```

Minimalistic example to remove all files from a directory:
```python
from imagededup.methods import PHash, AHash, DHash, WHash, CNN
import os
from tqdm import tqdm

# phasher = PHash()
# phasher = AHash()
# phasher = DHash()
# phasher = WHash()
phasher = CNN()

image_dir = '/content/data/'

# Generate encodings for all images in an image directory
encodings = phasher.encode_images(image_dir=image_dir)

# Find duplicates using the generated encodings
duplicates = phasher.find_duplicates_to_remove(encoding_map=encodings)

for f in tqdm(duplicates):
  os.remove(os.path.join(image_dir, f))
```
For big folders it is recommended to use this code:
```python
# https://github.com/idealo/imagededup/issues/133
from imagededup.utils.general_utils import get_files_to_remove
from imagededup.methods import CNN
from tqdm import tqdm
import os
cnn = CNN()
image_dir = '/content/data2'
encodings = cnn.encode_images(image_dir)  # In your case, this should be a dictionary with 127K entries (if there are no corrupt images or images with unsupported format)

# Large scale similarity search
import nmslib
import numpy as np

data = np.array(list(encodings.values()))

index = nmslib.init(method='hnsw', space='cosinesimil')
index.addDataPointBatch(data)

# Set index parameters
M = 40  #  Max links per node
efConstruction = 40 # Size of the dynamic list used during construction. A larger value means a better quality index, but increases build time. Should be an integer value between 1 and the size of the dataset.

num_threads = 4
index_time_params = {'M': M, 'indexThreadQty': num_threads, 'efConstruction': efConstruction, 'post' : 0} # 'post': postprocessing
index.createIndex(index_time_params, print_progress=True)

K = data.shape[0] # number of neigbours (setting to the size of dataset, usual practice is to specify a value such as 100 or so)
efSearch = 50 # Size of the dynamic list used during search. Higher values lead to improved recall at the expense of longer search time. Can take values between k and the size of the dataset and may be greater or smaller than ef_construction. Typical values are 100 - 2000.
query_time_params = {'efSearch': efSearch}
print('Setting query-time parameters', query_time_params)
index.setQueryTimeParams(query_time_params)
neighbours = index.knnQueryBatch(data, k=K)

def retrieve_neighbours_one_file(neighbours_onefile, onefile_matrix_row_index, sim_thresh, all_filenames):
    # gets duplicates for one file
    self_retrived_file_pos = np.where(neighbours_onefile[0] == onefile_matrix_row_index) # Avoid self retrieval
    neighbours_onefile_files = np.delete(neighbours_onefile[0], self_retrived_file_pos)
    neighbours_onefile_sims = np.delete(neighbours_onefile[1], self_retrived_file_pos)
    
    sim_neighbors = 1 - neighbours_onefile_sims  # convert distance to similarity
    thresh_sims = sim_neighbors[np.where(sim_neighbors >= sim_thresh)]
    thresh_neighbours = neighbours_onefile_files[np.where(sim_neighbors >= sim_thresh)]
    thresh_neighbours_filenames = [all_filenames[i] for i in thresh_neighbours]
    dups = list(zip(thresh_neighbours_filenames, thresh_sims))
    return dups

filenames = list(encodings.keys())
file_matrix_inds = range(data.shape[0])
min_sim_threshold = 0.9
res = list(map(retrieve_neighbours_one_file, neighbours, file_matrix_inds, [min_sim_threshold] * data.shape[0], [filenames] * data.shape[0]))
duplicates = dict(zip(filenames, res))

for f in tqdm(get_files_to_remove(duplicates)):
  os.remove(os.path.join(image_dir, f))
```

-------------------

This package provides functionality to make use of hashing algorithms that are particularly good at finding exact
duplicates as well as convolutional neural networks which are also adept at finding near duplicates. An evaluation
framework is also provided to judge the quality of deduplication for a given dataset.

Following details the functionality provided by the package:

- Finding duplicates in a directory using one of the following algorithms:
    - [Convolutional Neural Network](https://arxiv.org/abs/1704.04861) (CNN)
    - [Perceptual hashing](http://www.hackerfactor.com/blog/index.php?/archives/432-Looks-Like-It.html) (PHash)
    - [Difference hashing](http://www.hackerfactor.com/blog/index.php?/archives/529-Kind-of-Like-That.html) (DHash)
    - [Wavelet hashing](https://fullstackml.com/wavelet-image-hash-in-python-3504fdd282b5) (WHash)
    - [Average hashing](http://www.hackerfactor.com/blog/index.php?/archives/432-Looks-Like-It.html) (AHash)
- Generation of encodings for images using one of the above stated algorithms.
- Framework to evaluate effectiveness of deduplication  given a ground truth mapping.
- Plotting duplicates found for a given image file.

Detailed documentation for the package can be found at: [https://idealo.github.io/imagededup/](https://idealo.github.io/imagededup/)

imagededup is compatible with Python 3.6+ and runs on Linux, MacOS X and Windows. 
It is distributed under the Apache 2.0 license.

## 📖 Contents
- [Installation](#%EF%B8%8F-installation)
- [Quick Start](#-quick-start)
- [Benchmarks](#-benchmarks)
- [Contribute](#-contribute)
- [Citation](#-citation)
- [Maintainers](#-maintainers)
- [License](#-copyright)

## ⚙️ Installation
There are two ways to install imagededup:

* Install imagededup from PyPI (recommended):

```
pip install imagededup
```

> ⚠️ **Note**: The TensorFlow >=2.1 and TensorFlow 1.15 release now include GPU support by default.
> Before that CPU and GPU packages are separate. If you have GPUs, you should rather
> install the TensorFlow version with GPU support especially when you use CNN to find duplicates.
> It's way faster. See the [TensorFlow guide](https://www.tensorflow.org/install/gpu) for more
> details on how to install it for older versions of TensorFlow.

* Install imagededup from the GitHub source:

```bash
git clone https://github.com/idealo/imagededup.git
cd imagededup
pip install "cython>=0.29"
python setup.py install
```  

## 🚀 Quick Start

In order to find duplicates in an image directory using perceptual hashing, following workflow can be used:

- Import perceptual hashing method

```python
from imagededup.methods import PHash
phasher = PHash()
```

- Generate encodings for all images in an image directory

```python
encodings = phasher.encode_images(image_dir='path/to/image/directory')
```

- Find duplicates using the generated encodings

```python
duplicates = phasher.find_duplicates(encoding_map=encodings)
```

- Plot duplicates obtained for a given file (eg: 'ukbench00120.jpg') using the duplicates dictionary

```python
from imagededup.utils import plot_duplicates
plot_duplicates(image_dir='path/to/image/directory',
                duplicate_map=duplicates,
                filename='ukbench00120.jpg')
```
The output looks as below:

<p align="center">
  <img src="readme_figures/plot_dups.png" width="600" />
</p>


The complete code for the workflow is:
```python
from imagededup.methods import PHash
phasher = PHash()

# Generate encodings for all images in an image directory
encodings = phasher.encode_images(image_dir='path/to/image/directory')

# Find duplicates using the generated encodings
duplicates = phasher.find_duplicates(encoding_map=encodings)

# plot duplicates obtained for a given file using the duplicates dictionary
from imagededup.utils import plot_duplicates
plot_duplicates(image_dir='path/to/image/directory',
                duplicate_map=duplicates,
                filename='ukbench00120.jpg')
```
For more examples, refer [this](https://github.com/idealo/imagededup/tree/master/examples) part of the
repository.

For more detailed usage of the package functionality, refer: [https://idealo.github.io/imagededup/](https://idealo.github.io/imagededup/)

## ⏳ Benchmarks
Detailed benchmarks on speed and classification metrics for different methods have been provided in the [documentation](https://idealo.github.io/imagededup/user_guide/benchmarks/).
Generally speaking, following conclusions can be made:

- CNN works best for near duplicates and datasets containing transformations.
- All deduplication methods fare well on datasets containing exact duplicates, but Difference hashing is the fastest.

## 🤝 Contribute
We welcome all kinds of contributions.
See the [Contribution](CONTRIBUTING.md) guide for more details.

## 📝 Citation
Please cite Imagededup in your publications if this is useful for your research. Here is an example BibTeX entry:
```BibTeX
@misc{idealods2019imagededup,
  title={Imagededup},
  author={Tanuj Jain and Christopher Lennan and Zubin John and Dat Tran},
  year={2019},
  howpublished={\url{https://github.com/idealo/imagededup}},
}
```

## 🏗 Maintainers
* Tanuj Jain, github: [tanujjain](https://github.com/tanujjain)
* Christopher Lennan, github: [clennan](https://github.com/clennan)
* Dat Tran, github: [datitran](https://github.com/datitran)

## © Copyright
See [LICENSE](LICENSE) for details.

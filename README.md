# Note
This is a localized and bug-fixed fork of Membrain-v1. Original repo: https://github.com/CellArchLab/MemBrain .

# MemBrain-v1
### Automated detection of membrane-bound proteins

<p align="center">
<img src="./imgs/memBrain_github2.png" width="820">
</p>

<br>




MemBrain is a pipeline for the automated detection of membrane-bound proteins in cryo-electron tomograms. It utilizes 
the geometry of a pre-segmented membrane to reduce the complexity of the detection task. As a result, MemBrain only requires a small amount of 
annotated data (even one single annotated membrane can be enough!) and can generalize well to unseen tomograms and membranes.

pre-trained model (see folder `./models`)



<a name="instructions"></a>
## User instructions

### Data structure
MemBrain relies on a specific data structure, where tomograms, membrane segmentations, membrane meshes, and ground truth positions should be stored: (an exemplary data structure can also be found in our [example dataset](#example-data))
```
tomograms
│
└───Tomo1
│   │   Tomo1_(...)_bin4_denoised.rec
|   |   Tomo1_(...)_bin4_dose-filt.rec
│   │   Remarks: "_denoised" token should be added in case you want to decide later which version to use; "bin4" token is recommended (or respective other binning)
│   │
│   └───membranes
│   │       T1S1M1.mrc
│   │       T1S18M23.mrc
│   │       T1S19M1.mrc
│   │       ...
│   │       Remarks: stack token (S1, S18, ...) and membrane token (M1, M23, ...) should be included in filenames, ending with (membrane_token + ".mrc")
|   |
│   └───meshes
│   │       T1S1M1.obj
│   │       T1S18M23.obj
│   │       T1S19M1.obj
│   │       ...
│   │       Remarks: folder only necessary when training with membranorama data and using orientations; files should have names corresponding to membrane segmentations
|   |   
│   └───positions
│   │       T1S1M1.xml
│   │       T1S18M23.xml
│   │       T1S19M1.xml
│   │       ...
│   │       Remarks: folder only necessary for training. Files shoud be named corresponding to membrane segmentations.
|
└───Tomo23
    │   Tomo23_(...)_bin4_denoised.mrc
    │   Tomo23_(...)_bin4_dose-filt.mrc
```

### Setting up the environment
For the correct Python environment, set up a virtual environment of Python 3.8 (e.g., using [miniconda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/linux.html)). When using Miniconda, the following commands can be performed to create a Python environment, after Miniconda is installed:
```
git clone https://github.com/qmzhao26/MemBrain-v1-WorkflowAdaptation
cd MemBrain-v1-WorkflowAdaptation
conda env create -f MemBrain_requirements.yml
conda activate MemBrain
```
In case you have a GPU available, you can run ```conda env create -f MemBrain_requirements_gpu.yml``` instead of the corresponding line above. This will create an environment that is able to make use of the GPU (for training and inference of the NN; other steps are not influenced). Activate the GPU environment using ```conda activate MemBrain_GPU```.

Note that the virtual environment should always be activated when running Python scripts from this repository.

### Script usage
#### 0. Adjust config file
First, open the config file (config.py) and adjust the values according to your needs.

Hint: Setting "PICK_ON_BOTH_SIDES" to True will skip the step for manual picking of membrane sides, and process both sides of each membrane. While this setting is recommended, if graphical interfaces cannot be displayed (e.g. on a compute cluster), it is not recommended for training, as both sides use the same ground truth (which introduces probably wrong information).

#### 1. Choose the correct picking sides of your membranes and sample points.

Run the command
```
python step1_sample_points.py
```
This will first generate the new pipeline directory. Then, it will display the membrane segmentations one by one and lets you choose the side of the membrane to pick points on. Clicking a point with a relatively large distance to the membrane segmentation makes the side-picking more robust.

Afterwards, points and corresponding membrane normal vectors are sampled. Using the command 
```
python step1b_inspect_picked_sides.py
```
you can inspect the picked sides of the membranes and verify whether they are correct.

#### 2. Extract subvolumes
For the extraction and preprocessing of subvolumes, run
```
python step2_create_subvolumes.py
```
In the config file, you can specify the size of the extracted subvolumes, as well as the cutoff value for the low-pass filter applied to the tomogram beforehand.

#### 3.1. Train the neural network
In case you don't want to use a pre-trained network, you will need to train MemBrain yourself. You can do this using 
```
python step3a_training.py --cktp /path/to/previous/ckpt
```
The flag --ckpt is optional. If you want to continue training from a previously trained model, you can specify the path to that model here.

Again, you can specify training parameters in the config file, such as number of epochs, batch size, cutoff for particle distances, and particles to use for distance computation, in the config file.

Trained models and checkpoints will be stored in the folder `lightning_logs`.

#### 3.2. Inference on data
Apply the trained model on unseen data by using 
```
python step3b_predict_heatmaps.py /path/to/trained/model
```
This time, the path to the trained model is not optional, as we need to have a trained model for inference.

Executing this script will generate heatmaps for each membrane and for each particle class specified in the training parameters (config file). They are stored as .csv files and as .vtp files. The latter ones can easily be inspected, e.g. in Paraview.

#### 4. Find particle center positions
Finally, we can run the script
```
python step4_extract_particle_centers.py
```
or 
```
python step4_extract_particle_centers.py --eval True
```

--eval is again an optional argument. If set to True, evaluation metrics will be computed, such as the Chamfer distance between prediction and ground truth positions, as well as hit statistics based on various hit threshold distances.
Note: For ```--eval True``` to work, you will need to have corresponding .xml files for each membrane side (currently, this might not be the case in the toy example).

In the config file, you can also specify the bandwidths that should be used for clustering. (Per clustering, only one is used. Setting multiple bandwidths can help to compare them). 

Outputs will be particle centers in .csv and .vtp format, stored in the folder particle_centers/raw/`

#### 5. Find particle orientations
As an additional feature, we can extract estimates of particle orientations for each extracted particle position. This can be done using the command
```
python step5a_extract_orientations.py
```


## Example data

toy_data: containing 3 membranes from one tomogram (https://elifesciences.org/articles/53740). 

### Data preparation
run

```sh prepare_toy_dataset.sh```

This will automatically unzip the compressed membrane files, download the raw tomogram from EMDB (https://www.ebi.ac.uk/emdb/EMD-10780), and integrate everything into the data structure.

### Adjustments of config.py
Ideally, this toy example should work without adjusting config file values. However, if problems with the paths arise, changing to absolute paths might help:
- PROJECT_NAME
- PROJECT_DIRECTORY (this is where all outputs of MemBrain are stored; directory should exist beforehand)
- TOMO_DIR (where your toy_data tomograms are stored, e.g., `/path/to/MemBrain/folder/MemBrain/toy_data/tomograms`)
- USE_GPU


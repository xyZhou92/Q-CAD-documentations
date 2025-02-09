Machine Learning Force Field calculation
========================================
Machine Learning Force Field for MD calculation

Theoretical Backgrounds
-----------------------
| Phys. Rev. Lett. 98, 146401
| Phys. Rev. B 99, 064103  

Environment Setup 
-----------------

We recommmend using anaconda for package installing and management. You should first install Anaconda following the steps shown on the official website. Please don't attempt to use miniconda instead of Anaconda, since it might incur various dependecy and path errors that are hard to debug. 

To install conda environment, use the following command. You may choose a new version of Anaconda. 

::

    wget https://repo.anaconda.com/archive/Anaconda3-2020.07-Linux-x86_64.sh

Then, create a new environment for this module. 

::
    
    conda create -n mlff python=3.8

Next, install relevant packages in the current environment. Notice that the first line has included the required packages for GPU acceleration. Make sure your CUDA version is larger than that of cudatoolkit. 

::

    conda install pytorch=1.8.1
    conda install pandas
    conda install matplotlib
    conda install scikit-learn-intelex
    conda install numba 
    conda install tensorboard


::

    new:
    conda install pytorch torchvision torchaudio cudatoolkit=10.2 -c pytorch
    conda install pandas
    conda install matplotlib
    conda install scikit-learn-intelex
    conda install numba 

Compiling pytorch from source
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You might need to update submodules for multiple times due to unstable internet connection to GitHub server. 

::  
    
    git submodule update --init --recursive 



After installation, re-enter the current environment. 

:: 

    conda deactivate
    conda activate mlff

Now, enter the src directory and compile source codes. Intel 2020 module must be loaded. 

:: 

    module load intel/2020
    cd src
    sh build.sh
    

After compilation is done, you should modify environment variables. The absolute path of src/bin should be exported in ~/.bashrc. You can use "echo $PWD" to obtain the absolute path. 

::

    vim ~/.bashrc 
    export PATH=absolute/path/of/src/bin:$PATH
    source ~/.bashrc 

Prepare training data 
---------------------

The format of training data is same as the MOVEMENT files generated by PWmat's molecular dynamics caculation. Please refer to PWmat manual for details involving MD calculations. **Notice that it is very important to set energy_decomp = T in etot.input**, otherwise there will be no feature to extract. 

A sample etot.input is shown below. 

:: 

    16  1
    JOB = MD
    IN.PSP1 = Cu.SG15.PBE.UPF
    IN.PSP2 = Au.SG15.PBE.UPF
    IN.ATOM = atom.config
    MD_DETAIL = 3 400 0.8 300 300
    E_Cut = 60
    precision = double
    energy_decomp = T       #this flag must be true
    mp_n123 = 1 1 1 0 0 0 2
    xcfunctional = GGA
    E_error = 1.0e-6
    Rho_error = 1.0e-4

The resultant MOVEMENT files should be combined into one single file. Use the following command to do so. 

::

    cat MOVEMENT_1 MOVEMENT_2 ... > MOVEMENT 

Names other than "MOVEMENT" are not allowed.  

**Principles for generating trianing data**

As the first principle, training data set should well represent the 3N-dimensional phase space, where N is the number of atoms. That is, data should include the system’s spatial configurations as many as possible. The reason is sell-evident under the framework of energy decomposition. In our example, the training data is usually made up of images from more several MD results with varying condtitions. However, these images are sampled from the raw data, otherwise data size can be overwhelming. We now use some naïve rules to pick up images from the raw data. We may introduce more complex sampling method in the future. 

Configuration and features generation  
-------------------------------------

First, export the absolute path to src/bin in the ~/.bashrc. You can obtain the absolute path via command "pwd". 
::
    cd src/bin
    pwd
    (copy the absolute path)
    vim ~/.bashrc
    export PATH=/absolute/path/to/bin:$PATH

Create a new directory (call it examples) that will contain all the cases. This directory should be created in the directory that contains README.md. Enter directory examples, and create a new directory for a single system. In our exmaple, we study Copper, whose directory is called Cu1646. 


In Cu1646, create a directory callled **PWdata** and move the MOVEMENT file in it. A parameters.py file should appear in the same directory. An environmental configuration also has to be done.

**codedir**: the absolute path of the MLFF package, which is the one that contains directory src. Notice that letter r must appear in front of the path string. This step 

::
    
    codedir=r'/your/path/to/MLFF_torch'

For feature generation, the folllowing parameters should be set correctly. 

**atomType**: the atomic numbers. In the example case, system consists of only Cu, thus atomType should be [29]. If the system contains more than one element, all atomic numbers should be specified. For instance, atomType should be [8,29] for CuO. Order does not matter here. 

**use_Ftype**: features fed into the training process. 8 types of features are provided, which are 

        1. 2-body(2b)

        2. 3-body(3b) 

        3. 2-body Gaussian(2bgauss)

        4. 3-body Cosine(3bcos) 

        5. Multiple Tensor Potential(MTP)

        6. Spectral Neighbor Analysis Potential(SNAP)

        7. deepMD-Chebyshev(deepMD1)
        
        8. deepMD-Gaussian(deepMD2) 

Please refer to Theoretical Backgrounds section for more details. Usually, combinations such as [1,2],[3,4],[5],[7],[8] are used, but you are free to explore other combinations. In the given example, we use [1,2]. 

**isCalcFeat**: set to be True. Notice that this step will generate feature output files that can be reused by other training processes. They are stored in directory fread_dfeat. 

**maxNeighborNum**: its default value is 100, so you can try it with altering. However, for some system it is not enough to accommodate all the neighbors, and the feature generation fails. The singal of such an error can be found in /output. For each feature, an out file is generated. There should be out1 and out2 if feature combination [1,2] is chosen. In each out file, feature generation detail of each MD step is recorded. The correct scenario is shown below. 

.. image:: pictures/feature_success.png

If, however, you find that no MD information was printed, like the scenario shown below, you shoud assign **maxNeighborNum** with a larger number. 

.. image:: pictures/feature_fail.png 

After parameters are all set, run mlff.py to obtain the features. 
::
    
    mlff.py

Having generated the feature data, you can now feed them in various training engines. **isCalcFeat** should be turned off now. 

Engine 1: Linear Model
----------------------

1.Training
^^^^^^^^^^

Turn on **isFitLinModel** to lanuch linear fitting. After training, turn off **isFitLinModel**. 

2.Inference
^^^^^^^^^^^

After training, you can use the model to run MD calculation in an alternative data set. We call this step inference. To do so, prepare another Ab Initio MOVEMENT file. Create a new directory called MD and move another MOVEMENT into it.  

Several parameters should be set. 

**isNewMd100**: set True

**imodel**: set to be 1, which is linear model. 

**md_num_process**: the mpi process number you wish to use. Its value can be up to the number of available cores in you CPU. 

Next, run mlff.py. You may also use the bash file we provided to submit a mlff job. 

::
    
    mlff.py

A sample slurm script is given below. Notice that when submitting jobs through slurm, ntasks-per-node determines how many cores you can use. 

::

    #!/bin/sh
    #SBATCH --partition=mypartition
    #SBATCH --job-name=cu1646_l12
    #SBATCH --nodes=1
    #SBATCH --ntasks-per-node=1
    #SBATCH --threads-per-core=1

    conda activate mlff

    mlff.py

In our example, a new MOVEMENT file can be found after the inference step. You can copy plot_mlff_inference.py from utils/ directory to visualize the results. Below is the plot of results for Cu1646 case. 

.. image:: pictures/cu1646_linear.png


Engine 2: Nonlinear Model(VV) 
-----------------------------

VV (vector * vector)goes beyond linear fitting by introducing nonlinearity. In linear model, we approximate the total energy by a linear combination of features. But in VV, we build a new set of features from the old ones. These new features are generated by feeding old ones into nonlinear functions. For example, they could be exp(-F_i), F_i* F_i, F_i* F_i *F_i, .etc.

1.Training
^^^^^^^^^^

First, perform feature generation and fitting as in linear model. To do so, set isCalcFeat=True and isFitLinModel=True, and run mlff.py. 

Next, run select_VV_MM.r twice to obtain some new features. In the first run, 

2. Inference 
^^^^^^^^^^^^^ 

Engine 3: Kalman Filter-based Neural Network
--------------------------------------------

In this engine, we use Kalman filter to improve the bare neural network(NN). Essentially, Kalman filter smooths the “spikes” of the high dimension cost function, curbing the likelihood of falling into local minimum. 

1.Training
^^^^^^^^^^

First, several NN parameters should be set. 

**batch_size**: must be 1. We may support different batch sizes in the future. 

**nLayer** The layer of neural network. Notice that more layers does not mean better result! In our example, we set it to be 3. 

**nNode** The dimension of nodes. We use 15 as default. Since a slight increase in it will lead to dramatic increase in training time. The reason is that the triaining scales as O(nNode^6). 


After this, several parameters should also be set.  

**natoms** If more than one type of atom present, one should also set natoms correctly. For example, if the system of interest consists of 4 Cu atom and 7 Au atom, then you should set atomType = [29,79] and natoms = [4,7]. 

**nFeatures** It is the number of features. It should be the sum of the two numbers in the last line of   /fread_dfeat/feat.info. In our example, nFeatures is 42. 

We now use seper.py to devide data into a training set and a validation set. Currently, the division is a simple cut between first 80% and 20%. We might provide more complicated division method in the future. 

::

    seper.py

Next, use gen_data.py to re-format data. After this step you will find them in the directory train_data. 

::

    gen_data.py

Finally, set the following parameters:

**dR_neigh**: set to be False 

**use_GKalman**: set to be True

**use_LKalman**: set to be False

**is_scale**: set to be True

**itype_Ei_mean**: the estimation of mean energy of each type of atom. You should go to train_data/final_train and take a look at engy_scaled.npy via the following commands,

::

    cd train_data/final_train
    python 
    import numpy 
    numpy.load("engy_scaled.npy")

You don't need an excact mean, and a rough estimate should suffice. For example, if the commands above returns something like this:

::

    array([[174.0633357],
       [174.0604308],
       [174.0453315],
       ...,
       [437.0013048],
       [437.3404306],
       [437.2137406]])

you can just set 

::

    itype_Ei_mean=[174.0,437.0] 

**n_epoch**: the number of epoch for training. 

You can now launch train.py. You should also specify a directory with flag -s to save the logs and models.

::
    
    train.py -s records 

You can also use scripts to submit a job on you cluster. For example, 

::
        
    #!/bin/sh
    #SBATCH --partition=mypartition
    #SBATCH --job-name=myjobname
    #SBATCH --nodes=1
    #SBATCH --ntasks-per-node=32
    #SBATCH --threads-per-core=1
    
    conda activate mlff 

    train.py -s records


2. During and after training
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

During training, you can monitor te progress by checking the logs in records directory. 

**epoch_loss.dat**: loss, RMSE_Etot, RMSE_Ei, RMSE_F of training set in each epoch

**epoch_loss_valid.dat**: RMSE_Etot, RMSE_Ei, RMSE_F of valid set in each epoch

**model**: directory that contains the obtained models. The latest and the best model will be saved. 

You should compare epoch_loss.dat and epoch_loss_valid.dat to see if an overfitting occurs. 

Engine 4: DeepMD + Kalman 
---------------------------


Only 1 feature can be used at each time. 

Files in OP must be compiled first. 

**dR_neigh**: set to be True 

::
    
    train.py --deepmd=True -n DeepMD_cfg_dp -s record


Or submit the file via deepMD.sh


Case Study: Linear Model in Sulphur 
-----------------------------------

From this section, we show mutiple examples in order to demonstrate the predictive ability of linear model in various systems. Notice that since these examples are almost pedagogical, and the parameters used wihtin, such as ones to control the formation of training and testing data and the combination of features, are by no means optimal. You should try different parameters to meet you needs in your own projects. 

We start with Sulphur bulk that contains 64 atoms. First, one should prepare training set and testing set. Right now, we only consider linear sampling. You may use python script make_set.py to generate them. 

::
    
    python make_set.py MOVEMENT-1 MOVEMENT-2 ... 

You can change "sample_dist" to control the distance between two samples. For example, if a MOVEMENT file has 1000 images, then image 0, image 10, imag 20 ... image 1000 will be picked put together as a new MOVEMENT file. In this case we use 10 for training set and 30 for test set. Evidently, sample_dist for training set and test set can determine the overlap between training set and test set. 

In our first try, feature 1 and 2 are used. The training result is shown below. 

.. image:: pictures/S_bulk_lft_[1,2]_tr=10_te=30.png 

The RMSE of the total energy is 0.946, which is not quite good. One might intuitively propose to include more images in the trianing data to improve the training result. However, from our experience this might not be the best option. Rather, adding more features might bring more improvement. The following example uses feature 1,2,3,and 4. 

.. image:: pictures/S_bulk_lft_[1,2,3,4]_tr=10_te=30.png 

Now, RMSE is 0.5093, an significant improvement in comparison to the previous trianing. We further add feature 5 in the training, and the result is shown below. 

.. image:: pictures/S_bulk_lft_[1,2,3,4,5]_tr=10_te=30.png 

By including feature 5, RMSE is lowered by alomst 50% again. 

Case Study: Linear Model in bulk Cu 
------------------------------------

We show the training results of Cu bulk. The system contains 108 atoms. ample_dist is 10 and 13 for training set and testing set respectively. Features used for training are [1,2],[1,2,3,4]. Notice that when only feature 1 and 2 are used, the training outcome is already quite good. 

.. image:: pictures/Cu_[1,2].png

.. image:: pictures/Cu_[1,2,3,4].png

Case Study: Linear Model in BCC Lithium  
---------------------------------------
We show the training results of BCC Lithium. The system contains 230 atoms. sample_dist is 10 and 13 for training set and testing set respectively. Features used for training are [1,2],[1,2,3,4],[1,2,3,4,5]. 

.. image:: pictures/Li_bcc_lft_[1,2]_tr=10_te=13.png

.. image:: pictures/Li_bcc_lft_[1,2,3,4]_tr=10_te=13.png

.. image:: pictures/Li_bcc_lft_[1,2,3,4,5]_tr=10_te=13.png

Case Study: Linear Model in FCC Lithium  
---------------------------------------
We show the training results of FCC Lithium. The systems contains 192 atoms. sample_dist is 10 and 13 for training set and testing set respectively. Features used for training are [1,2],[1,2,3,4],[1,2,3,4,5]. 

.. image:: pictures/Li_fcc_lft_[1,2]_tr=10_te=13.png

.. image:: pictures/Li_fcc_lft_[1,2,3,4]_tr=10_te=13.png

.. image:: pictures/Li_fcc_lft_[1,2,3,4,5]_tr=10_te=13.png

(Bad) Case Study: Linear Model in CuO   
-------------------------------------

Linear model is not a cure-all. In fact, we find that so far, linear model works reasonably well only in bulk systems that contains one type of atom. Below, we will show some systems that are not well modeled by the linear model. 

Below are the training results of Copper oxide. The system contains 64 atoms. Notice that RMSE of the total energy is not signicifanctly improved as the number of feature increses. 

.. image:: pictures/CuO_[1,2].png

.. image:: pictures/CuO_[1,2,3,4].png


(Bad) Case Study: Linear Model in graphene  
-------------------------------------------

For 2D materials such as graphene, lienar model also appears to be much less effective. Below are the trianing results for graphene. The system contains 64 atoms. sample_dist is 10 and 13 for training set and testing set respectively. Features used for training are [1,2],[1,2,3,4],[1,2,3,4,5]. Notice that as number of feature increases, no obvious improvement is observed. 

.. image:: pictures/Graphene_lft_[1,2]_tr=10_te=23.png

.. image:: pictures/Graphene_lft_[1,2,3,4]_tr=10_te=23.png

.. image:: pictures/Graphene_lft_[1,2,3,4,5]_tr=10_te=23.png

# Copy of Cross-Modal Supervision (Policy Training Code using RMA) for Go1

This repo builds on the code from ```RMA: Rapid Motor Adaptation for Legged Robots``` to train the locomotion policy associated with the paper ```Learning Visual Locomotion with Cross-Modal Supervision```.
For more information, please check the [RMA project page](https://ashish-kmr.github.io/rma-legged-robots/) and [CMS project webpage](https://antonilo.github.io/vision_locomotion/). For using the policy on a real robot, please refer to [this repository](https://github.com/antonilo/vision_locomotion).


#### Paper, Video, and Datasets

If you use this code in an academic context, please cite the following two publications:

Papers:     
[RMA: Rapid Motor Adaptation for Legged Robots](https://arxiv.org/abs/2107.04034), Narration: [Video](https://www.youtube.com/watch?v=qKF6dr_S-wQ)           
[Learning Visual Locomotion With Cross-Modal Supervision](https://antonilo.github.io/vision_locomotion/pdf/manuscript.pdf), Narration: [Video](https://youtu.be/d7I34YIdMdk)

```
  @InProceedings{kumar2021rma,
   author={Kumar, Ashish and Fu, Zipeng and Pathak, Deepak and Malik, Jitendra},
   title={{RMA: Rapid motor adaptation for legged robots}},
   booktitle={Robotics: Science and Systems},
   year={2021}
  }

@inproceedings{loquercio2023learning,
  title={Learning visual locomotion with cross-modal supervision},
  author={Loquercio, Antonio and Kumar, Ashish and Malik, Jitendra},
  booktitle={2023 IEEE International Conference on Robotics and Automation (ICRA)},
  pages={7295--7302},
  year={2023},
  organization={IEEE}
}
```

## Usage

This code trains a policy using reinforcement learning to walk on complex terrains with minimal information. The code uses the Raisim simulator for training. Note that simulator is CPU-based.

### Raisim Install

Please follow the [installation guide](https://raisim.com/sections/Installation.html) of raisim. Note that we do not support the latest version of raisim. Please checkout the commit `f0bb440762c09a9cc93cf6ad3a7f8552c6a4f858` after cloning raisimLib.

### Training Environment Installation

Run the following commands to install the training environments

```
cd raisimLib
git clone git@github.com:antonilo/rl_locomotion.git
rm -rf raisimGymTorch
mv rl_locomotion raisimGymTorch
cd raisimGymTorch
mv rl_locomotion/raisimGymTorch ..
rm -r rl_locomotion
cd ..
mv raisimGymTorch/go1 rsc/
cd raisimGymTorch
# You might want to create a new conda environment if you did not do it already for the vision part
conda create --name cms python=3.8
conda activate cms
pip install -e .

# installation of the environments
python setup.py develop
```

### Training a policy with priviledged information

You can use the following code to train a policy with access to priviledged information about the robot (e.g. mass, velocity, and motor strenght) and about the environment (e.g. the terrain geometry). The optimization will be guided by trajectories generated by [a previous policy](data/base_policy) we provide. You can control the strenght of the imitation by changing the parameter `RL_coeff` in [this file](./raisimGymTorch/env/envs/rsg_go1_task/runner.py).

To start training, you can use the following commands:

```
cd raisimLib/raisimGymTorch/raisimGymTorch/env/envs/rsg_go1_task
python runner.py --name random --gpu 1 --exptid 1
```
It will take approximately 4K iterations to train a good enough policy. If you want to make any changes to the training environment, feel free to edit [this file](./raisimGymTorch/env/envs/rsg_go1_task/Environment.hpp). Note that every time you make changes, you need to recompile the file by running this commands:

```
cd raisimLib/raisimGymTorch
python setup.py develop
```

If you wish to continue a previous run, use the following commands:

```
cd raisimLib/raisimGymTorch/raisimGymTorch/env/envs/rsg_go1_task
python runner.py --name random --gpu 1 --exptid 1 --loadid ITR_NBR --overwrite
```

### Visualizing a policy

You can use the following code to see if your policy training worked. First run the unity renderer:

```
cd raisimLib/raisimUnity/linux
./raisimUnity.x86_64
```

In a separate terminal, run the policy
```
conda activate cms
cd raisimLib/raisimGymTorch/raisimGymTorch/env/envs/rsg_go1_task
python viz_policy.py ../../../../data/rsg_go1_task/EXPT_ID POLICY_ID

```
You can now analysize the behaviour!

### Benchmark a policy

If you want to know how the policy performs over a set of controlled experiments, use the following commands:
```
conda activate cms
cd raisimLib/raisimGymTorch/raisimGymTorch/env/envs/rsg_go1_task
python evaluate_policy.py ../../../../data/rsg_go1_task/EXPT_ID POLICY_ID

```
This will generate a json file in the experiment folder. You can visualize the results in a nice table using [this script](./eval_scripts/compute_results.py):

```
python compute_results.py ../data/rsg_go1_task/EXPT_ID/evaluation_results.csv

```

If you want to make any changes to the evaluation, feel free to edit [this file](./raisimGymTorch/env/envs/rsg_go1_task/Environment.hpp). Note that in the evaluation, the flag `Eval` is set to true. Remeber to recompile any time you edit the environment file!


### Distilling a policy with priviledged information into a blind policy 

A policy trained with priviledged information cannot be used on a physical robot. Therefore, we distill a priviledged policy using a slightly different version of [RMA](https://arxiv.org/abs/2107.04034) optimized for walking on complex terrains.

To start training, you can use the following commands:

```
cd raisimLib/raisimGymTorch/raisimGymTorch/env/envs/dagger_go1
python dagger.py --name cms_dagger --exptid 1 --loadpth ../../../../data/rsg_go1_task/EXPT_ID --loadid PRIV_POLICY_ID --gpu 1
```
It will take approximately 2K iterations to train a good enough policy. If you want to make any changes to the training environment, feel free to edit [this file](./raisimGymTorch/env/envs/dagger_go1/Environment.hpp). Note that every time you make changes, you need to recompile the environment (see above).

### Visualizing and evaluating a blind policy

You can follow exactly the same steps as for the priviledged policy (but now running commands from [this folder](./raisimGymTorch/env/envs/dagger_go1/)) to visualize and benchmark a blind policy.

### Using a blind policy on a real robot

If you want to use a policy you trained on a real robot, you should first move it in the [models folder](https://github.com/antonilo/vision_locomotion/tree/master/controller/models), change the path to the model in the [launch_file](https://github.com/antonilo/vision_locomotion/blob/master/controller/launch/cms_ros.launch#L7) and the policy id in the [parameter_file](https://github.com/antonilo/vision_locomotion/blob/master/controller/parameters/default.yaml#L2). 

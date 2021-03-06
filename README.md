# GLAMpoints: Greedily Learned Accurate Match points

This is the official pytorch implementation of our paper [ICCV 2019 paper "GLAMPoints: Greedily Learned Accurate Match points"](https://www.retinai.com/glampoints) ([arXiv](https://arxiv.org/pdf/1908.06812.pdf), [supplemental](https://static1.squarespace.com/static/5967a5599de4bb65a7bb9736/t/5d7a6d2b9e03815777188115/1568304461382/Supplementary_Material.pdf)). 
Please note that the reference implementation, with corresponding results presented in the paper, is in TensorFlow ([GLAMpoints_tensorflow](https://gitlab.com/retinai_sandro/glampoints)), therefore the results obtained with this version might be slightly different. 


## Settings

Use Python 3.7 and install the requirements with:
```bash
pip install -r requirements.txt
```


## Reminder of the steps of the training

**Goal of the method**: It enables the training of a domain-specific keypoint detector over non-differentiable registration methods. This code exemplifies the presented method using root SIFT, a homography model and RANSAC optimization.

![alt text](https://github.com/PruneTruong/GLAMpoints_pytorch/blob/master/images/summary_GLAMpoints.png)


== Preprocessing ==

- Load original data
- Crop and pad samples to 256x256

== Model ==
- Create 4-level deep Unet with batch normalization model f


== Data selection ==

At each iteration: 
- select original image 
- compute random geometric transformations g and g' (the maximum degree of the transformations can be chosen by the user depending on its test set)
- transform original image with g and g' to create respectively I and I'
- compute relation between images  I->I': H = g' * g^-1
- apply a subset of appearance changes: gaussian noise, changes of contrast, illumination, gamma, motion blur and the inverse of image.
Again, the maximum degree of the appearance changes can be chosen by the user. 

== Training ==

- compute S=f(I) and S'=f(I')
- compute NMS for both (N, N')
- compute root Sift descriptor N, N' (D, D')
- match points with (N, D), (N', D')
- find true positives and false positives, where true positives fullfill ||H*x - x'|| <= epsilon for epsilon=3
- compute reward map R
- compute mask M
- compute loss as L = sum( (f(I)-R)**2)*M / sum(M, axis=[1,2,3]) )



## Training

To train the model on your dataset, prepare a folder with reference images (of approximately same height and width). 
Configure the file `config_training.yml` with at least `write_dir` and `training, TRAIN_DIR`. 
Please note that `training, TRAIN_DIR` must contain the path to the folder containing the reference images used to create the 
synthetic pairs of training images. If `training, train_list` is equal to ' ', all images contained in `training, TRAIN_DIR` 
will be used for the creation of training pairs. Alternatively, if `training, train_list` is not equal to ' ', `training, train_list` can contain 
the absolute path to  a file listing the names of the reference images to use in order to create the synthetic training images. 
Those images must be within the root directory `training, TRAIN_DIR` indicated. 

The same set-up must be performed for the validation set. 


One can adapt the hyperparameters in `training` to your dataset. 



**Note on the synthetic image pair creation:** At each epoch, random geometric transformations are applied to the original 
images so as to synthetically create the pairs of training images. One can adapt the degree of those geometric transformations 
by changing the parameters in `sample_homography` of the training_config. 
  Similarly, which appearance changes are applied to the training images as well as their strength can be adapted in`augmentation`. 
  **The degree of the geometric transformations should be chosen so that the resulting synthetic training set resembles the test set.**
In our case, our test set composed of retinal images only showed rotation up to 30 degrees and only little scaling changes, therefore we limited
the geometric transformations of the training set accordingly. 
However, **any degree of geometric transformation** or even non-linear ones can be applied to the original images to synthetically create pairs of training images. 


If you want to start the training from pre-trained model weights, put `training,load_pre_training` equal to True and indicate the 
path to the weights in `training,pre_trained_weights`.


With this setup, run the script:

```bash
python training_glam_detector.py --path_ymlfile config_training.yaml --compute_metrics True
```

## Testing


### Detect GLAMpoints from a list of images

To test the pretrained model on a set of selected images, use the following script:

```bash
python compute_glam_kp.py --path_images /path/to/images/ --write_dir /path/to/results/folder --NMS 10 --path_glam_weights weights/Unet4_retina_images_converted_tf_weights.pth --green_channel True
```

This script only detects keypoints out of each images 
Outputs of this script are figures, showing for each image the detected keypoints. These figures are saved in --write_dir. It can also be a text file with the kp saved. 
The parameters are the following:
* --path_images is a path to a directory of images. Keypoints are detected from each images in the folder.  
* --write_dir designates the directory where to save the images with the keypoints drawn on them
* --path_glam_weights is the path to the weights of the trained GLAMpoint model. 

optional arguments are
* --NMS which is the non max suppression window, default is 10
* --min_prob which is the minimum probability that a pixel must have to be considered as a keypoint on the score map (output of the Reti model), default is 0
* --green_channel: bool to determine if we are using the green channel instead of gray image to compute the score map (default is True).
* --save_text: bool to save a text file containing the matrix of kp extracted for each image. (default: False)


### Pipeline example of keypoint detection and matching from a pair of images

An example of the matching pipeline applied to a pair of images: compute_GLAMpoint_matches_and_registration.py

```bash
python compute_GLAMpoint_matches_and_registration.py --path_image1 /path/to/image1 --path_image2 /path/to/image2 --write_dir ///path/to/results/folder --NMS 10 --path_glam_weights weights/Unet4_retina_images_converted_tf_weights.pth --green_channel True
```

It extracts GLAMpoints out of both images as well as their corresponding root SIFT descriptor. Then the script looks for matches and estimates a homography transformation using RANSAC. 
It can also be compared to SIFT detector. The output is a figure showing the matches and registration. 
The parameters are the following:
* --path_image1 is a path to the first image of the pair
* --path_image2 is a path to the second image of the pair
* --write_dir designates the directory where to save the images with the keypoints drawn on them
* --path_glam_weights is the path to the weights of the trained GLAMpoint model.

optional arguments are
* --SIFT is a bool stating if the same pipeline should be applied to the image pair when using SIFT detector (and descriptor). Default is True
* --NMS which is the non max suppression window, default is 10
* --min_prob which is the minimum probability that a pixel must have to be considered as a keypoint on the score map (output of the Reti model), default is 0
* --green_channel: bool to determine if we are using the green channel instead of gray image to compute the score map (default is True).


## GLAMPOINTS LICENSE CONDITIONS

Copyright (2019), RetinAI Medical AG.

This software for the training and application of Greedily Learned Matching keypoints is being made available for individual research use only. For any commercial use contact RetinAI Medical AG.

For further details on obtaining a commercial license, contact RetinAI Medical AG Office (sales@retinai.com). 

RETINAI MEDICAL AG MAKES NO REPRESENTATIONS OR
WARRANTIES OF ANY KIND CONCERNING THIS SOFTWARE.

This license file must be retained with all copies of the software,
including any modified or derivative versions.
>>>>>>> 54a9b6b918191207bf7a166226d7d6284548333c


## Reference

Should you make use of this work, please cite the paper accordingly:

```bash
@article{Truong2019GLAMpointsGL,
  title={GLAMpoints: Greedily Learned Accurate Match Points},
  author={Prune Truong and Stefanos Apostolopoulos and Agata Mosinska and Samuel Stucky and Carlos Ciller and Sandro De Zanet},
  journal={2019 IEEE/CVF International Conference on Computer Vision (ICCV)},
  year={2019},
  pages={10731-10740}
}
```

And this particular implementation: 

```bash
 @misc{pytorch-GLAMpoints,
         author = {Prune TRUONG},
         title = {A Reimplementation of {GLAMpoints} Using {PyTorch}},
         year = {2020},
         howpublished = {\url{https://github.com/PruneTruong/GLAMpoints_pytorch}}
    }
```
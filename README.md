#  Visual Localization for Autonomous Driving: Mapping the Accurate Location in the City Maze

This is the official github repository for ["Visual Localization for Autonomous Driving: Mapping the Accurate Location in the City Maze"](https://arxiv.org/pdf/2008.05678) which is an intuitive extension from [NetVLAD](https://openaccess.thecvf.com/content_cvpr_2016/papers/Arandjelovic_NetVLAD_CNN_Architecture_CVPR_2016_paper.pdf).

The code is made proper adjustment to take three-directional images as inputs and uses NetVLAD backbone to predict visual localization for autonomous driving. 

This work is distributed under the MIT License (see the `LICENCE` file).

## Data (Conventional one-directional images)
[Tokyo 24/7](https://data.ciirc.cvut.cz/public/projects/2015netVLAD/Tokyo247/database_gsv_vga/)

[Pittsburgh 250k](http://www.ok.ctrl.titech.ac.jp/~torii/project/repttile/)

[Tokyo Time Machine](https://data.ciirc.cvut.cz/public/projects/2015netVLAD/TokyoTimeMachine/)

[San Francisco Landmark Dataset](https://purl.stanford.edu/vn158kj2087)

### Trained model

[VGG-16 + NetVLAD + whitening trained by Pitts30k](https://drive.google.com/file/d/1Wy2Zy72_g_aNjUpC4_R9IIVQs4ZUxjH4/view?usp=sharing).

[VGG-16 + NetVLAD + whitening trained by TokyoTM](https://drive.google.com/file/d/1b8dPTvBAXyfWm8hJkl_k4JMMS2vKgHmx/view?usp=sharing).


### If you want to train your networks

In [1] we always started from networks pretrained on other tasks (ImageNet / Places205), download these from the [MatConvNet website](http://www.vlfeat.org/matconvnet/pretrained/). Download `imagenet-caffe-ref` and `imagenet-vgg-verydeep-16` for the AlexNet and VGG-16 experiments, respectively.

However, one can also start from any custom CNN. Change `loadNet.m` to load your initial network.

## Configure the NetVLAD library

Copy `localPaths.m.setup` into `localPaths.m` and edit the variables to point to dependencies, dataset locations, pretrained models, etc (detailed information is provided in the file).

## Run

See `demo.m` for examples on how to train and test the networks, as explained below. We use Tokyo as a runnning example, but all is analogous if you use Pittsburgh (just change the dataset setup and use the appropriate networks).

The code samples below use the GPU by default, if you want to use the CPU instead (very slow especially for training!), add `'useGPU', false` to the affected function calls (`trainWeakly`, `addPCA`, `serialAllFeats`, `computeRepresentation`).

Note that if something fails (e.g. you are missing a dependency, your GPU runs out of RAM, you manually stop execution, etc), you should make sure to delete the potentially created corrupt files before rerunning the code. E.g. if you terminate feature extraction, the output file will be incomplete, so trying to perform testing will fail (files are never recomputed if they exist).


Set the MATLAB paths:

    setup;

Load our network:

    netID= 'vd16_tokyoTM_conv5_3_vlad_preL2_intra_white';
    paths= localPaths();
    load( sprintf('%s%s.mat', paths.ourCNNs, netID), 'net' );
    net= relja_simplenn_tidy(net); % potentially upgrate the network to the latest version of NetVLAD / MatConvNet

Compute the image representation by simply running the forward pass using the network `net` on the appropriately normalized image (see `computeRepresentation.m`).

    im= vl_imreadjpeg({which('football.jpg')}); im= im{1}; % slightly convoluted because we need the full image path for `vl_imreadjpeg`, while `imread` is not appropriate - see `help computeRepresentation`
    feats= computeRepresentation(net, im); % add `'useGPU', false` if you want to use the CPU

To compute representations for many images, use the `serialAllFeats` function which is much faster as it uses batches and it moves the network to the GPU only once:

    serialAllFeats(net, imPath, imageFns, outputFn);

`imageFns` is a cell array containing image file names relative to the `imPath` (i.e. `[imPath, imageFns{i}]` is a valid JPEG image), the representations are saved in binary format (single 4-byte floats). Batch size used for computing the forward pass can be changed by adding the `batchSize` parameter, e.g. `'batchSize', 10`. Note that if your input images are not all of same size (they are in place recognition datasets), you should set `batchSize` to 1.

To test the network on a place recognition dataset, set up the test dataset:

    dbTest= dbTokyo247();

Set the output filenames for the database/query image representations:

    paths= localPaths();
    dbFeatFn= sprintf('%s%s_%s_db.bin', paths.outPrefix, netID, dbTest.name);
    qFeatFn = sprintf('%s%s_%s_q.bin', paths.outPrefix, netID, dbTest.name);

Compute db/query image representations:

    serialAllFeats(net, dbTest.dbPath, dbTest.dbImageFns, dbFeatFn, 'batchSize', 10); % adjust batchSize depending on your GPU / network size
    serialAllFeats(net, dbTest.qPath, dbTest.qImageFns, qFeatFn, 'batchSize', 1); % Tokyo 24/7 query images have different resolutions so batchSize is constrained to 1

Measure recall@N

    [recall, ~, ~, opts]= testFromFn(dbTest, dbFeatFn, qFeatFn);
    plot(opts.recallNs, recall, 'ro-'); grid on; xlabel('N'); ylabel('Recall@N'); title(netID, 'Interpreter', 'none');

To test smaller dimensionalities, all that needs to be done (only valid for NetVLAD+whitening networks!) is to keep the first D dimensions and L2-normalize. This is done automatically in `testFromFn` using the `cropToDim` option:

    recall= testFromFn(dbTest, dbFeatFn, qFeatFn, [], 'cropToDim', 256);

It is also very easy to test our trained networks on the standard object/image retrieval benchmarks, using the same set of steps: load the network, construct the database, compute the features, run the evaluation. See `demoRetrieval.m` for details.

### Train

Set the MATLAB paths:

    setup;

Load the train and validation sets, e.g. for Tokyo Time Machine:

    dbTrain= dbTokyoTimeMachine('train');
    dbVal= dbTokyoTimeMachine('val');

Run the training:

    sessionID= trainWeakly(dbTrain, dbVal, ...
        'netID', 'vd16', 'layerName', 'conv5_3', 'backPropToLayer', 'conv5_1', ...
        'method', 'vlad_preL2_intra', ...
        'learningRate', 0.0001, ...
        'doDraw', true);

All arguments of `trainWeakly` are explained in more details in the `trainWeakly.m` file, here is a brief overview of the essential ones:

 - `netID`: The name of the network (`caffe` for AlexNet, `vd16` for verydeep-16, i.e. VGG-16)
 - `layerName`: Which layer to crop the initial network at, we always use the last convolutional layer (i.e. conv5 for caffe and conv5_3 for vd16)
 - `backPropToLayer`: Down to which layer to perform the learning. If not specified, the entire network is trained, see [1] for the analysis
 - `method`: Which aggregation method to use for the image representation, default is `vlad_preL2_intra` (i.e. NetVLAD with input features L2-normalized, and with intra-normalization of the NetVLAD vector). You can also use `max` for max pooling, `avg` for average pooling, or other vlad variants (e.g. `vlad_preL2` to disable intra-normalization)
 - `learning rate`: The learning rate for SGD
 - `useGPU`: Use the GPU or not
 - `doDraw`: To plot or not some performance curves as training goes along

Other parameters are explained in `trainWeakly.m`, including SGD parameters (batch size, momentum, weight decay, learning rate schedule, ..), method parameters (margin size, number of negatives, size of the hard negative memory, ..), etc.

The training periodically saves the latest network and performance curves in files which include the sessionID (can be specified, otherwise generated randomly) and the epoch number, e.g.: 0fd5_ep000002_latest.mat , as well as a copy of that file for the latest epoch in 0fd5_latest.mat .

To find the best network, i.e. the one that performs the best on the validation set (we use recall@N here, where N=5, but any value can be used), run:

    [~, bestNet]= pickBestNet(sessionID);

#### Train PCA + whitening

The best performance is achieved if the dimensionality of the image representation is reduced using PCA together with whitening:

    finalNet= addPCA(bestNet, dbTrain, 'doWhite', true, 'pcaDim', 4096);

### Additional information

More information is available `README_more.md` and in comments in the code itself.

# References
@inproceedings{cityMaze2020,

  title={Visual Localization for Autonomous Driving: Mapping the Accurate Location in the City Maze},
  
  author={Dongfang Liu, Yiming Cui, Xiaolei Guo, Wei Ding, Baijian Yang, Yingjie Chen},
  
  booktitle={Proceedings of the 25th International Conference on Pattern Recognition},
  
  year={2020}
}

@inproceedings{arandjelovic2016netvlad,

  title={NetVLAD: CNN architecture for weakly supervised place recognition},
  
  author={Arandjelovic, Relja and Gronat, Petr and Torii, Akihiko and Pajdla, Tomas and Sivic, Josef},
  
  booktitle={Proceedings of the IEEE conference on computer vision and pattern recognition},
  
  pages={5297--5307},
  
  year={2016}
}

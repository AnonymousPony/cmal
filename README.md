Note: Some code in this repo are copied/modified from opensource implementations made available by
[PyTorch](https://github.com/pytorch/pytorch),
[HuggingFace](https://github.com/huggingface/transformers),
[OpenNMT](https://github.com/OpenNMT/OpenNMT-py),
and [Nvidia](https://github.com/NVIDIA/DeepLearningExamples/tree/master/PyTorch).
The image features are extracted using [BUTD](https://github.com/peteanderson80/bottom-up-attention).


## Requirements
We follow UNITER to provide Docker image for easier reproduction. Please install the following:
  - [nvidia driver](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#package-manager-installation) (418+), 
  - [Docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/) (19.03+), 
  - [nvidia-container-toolkit](https://github.com/NVIDIA/nvidia-docker#quickstart).

Our scripts require the user to have the [docker group membership](https://docs.docker.com/install/linux/linux-postinstall/)
so that docker commands can be run without sudo.
We only support Linux with NVIDIA GPUs. We test on Ubuntu 18.04 and A100 cards.
We use mixed-precision training hence GPUs with Tensor Cores are recommended.

## Quick Start
*NOTE*: Please run `bash scripts/download_pretrained.sh $PATH_TO_STORAGE` to get our latest pretrained
checkpoints. This will download both the base and large models.

We use NLVR2 as an end-to-end example for using this code base.

1. Download processed data and pretrained models with the following command.
    ```bash
    bash scripts/download_nlvr2.sh $PATH_TO_STORAGE
    ```
    After downloading you should see the following folder structure:
    ```
    ├── ann
    │   ├── dev.json
    │   └── test1.json
    ├── finetune
    │   ├── nlvr-base
    │   └── nlvr-base.tar
    ├── img_db
    │   ├── nlvr2_dev
    │   ├── nlvr2_dev.tar
    │   ├── nlvr2_test
    │   ├── nlvr2_test.tar
    │   ├── nlvr2_train
    │   └── nlvr2_train.tar
    ├── pretrained
    │   └── uniter-base.pt
    └── txt_db
        ├── nlvr2_dev.db
        ├── nlvr2_dev.db.tar
        ├── nlvr2_test1.db
        ├── nlvr2_test1.db.tar
        ├── nlvr2_train.db
        └── nlvr2_train.db.tar
    ```

2. Launch the Docker container for running the experiments.
    ```bash
    # docker image should be automatically pulled
    source launch_container.sh $PATH_TO_STORAGE/txt_db $PATH_TO_STORAGE/img_db \
        $PATH_TO_STORAGE/finetune $PATH_TO_STORAGE/pretrained
    ```
    The launch script respects $CUDA_VISIBLE_DEVICES environment variable.
    Note that the source code is mounted into the container under `/src` instead 
    of built into the image so that user modification will be reflected without
    re-building the image. (Data folders are mounted into the container separately
    for flexibility on folder structures.)


3. Run finetuning for the NLVR2 task.
    ```bash
    # inside the container
    python train_nlvr2.py --config config/train-nlvr2-base-1gpu.json

    # for more customization
    horovodrun -np $N_GPU python train_nlvr2.py --config $YOUR_CONFIG_JSON
    ```

4. Run inference for the NLVR2 task and then evaluate.
    ```bash
    # inference
    python inf_nlvr2.py --txt_db /txt/nlvr2_test1.db/ --img_db /img/nlvr2_test/ \
        --train_dir /storage/nlvr-base/ --ckpt 6500 --output_dir . --fp16

    # evaluation
    # run this command outside docker (tested with python 3.6)
    # or copy the annotation json into mounted folder
    python scripts/eval_nlvr2.py ./results.csv $PATH_TO_STORAGE/ann/test1.json
    ```
    The above command runs inference on the model we trained. Feel free to replace
    `--train_dir` and `--ckpt` with your own model trained in step 3.
    Currently we only support single GPU inference.


5. Customization
    ```bash
    # training options
    python train_nlvr2.py --help
    ```
    - command-line argument overwrites JSON config files
    - JSON config overwrites `argparse` default value.
    - use horovodrun to run multi-GPU training
    - `--gradient_accumulation_steps` emulates multi-gpu training


6. Misc.
    ```bash
    # text annotation preprocessing
    bash scripts/create_txtdb.sh $PATH_TO_STORAGE/txt_db $PATH_TO_STORAGE/ann

    # image feature extraction (Tested on Titan-Xp; may not run on latest GPUs)
    bash scripts/extract_imgfeat.sh $PATH_TO_IMG_FOLDER $PATH_TO_IMG_NPY

    # image preprocessing
    bash scripts/create_imgdb.sh $PATH_TO_IMG_NPY $PATH_TO_STORAGE/img_db
    ```
    In case you would like to reproduce the whole preprocessing pipeline.

## Downstream Tasks Finetuning

### VQA
NOTE: train and inference should be ran inside the docker container
1. download data
    ```
    bash scripts/download_vqa.sh $PATH_TO_STORAGE
    ```
2. train
    ```
    horovodrun -np 4 python train_vqa.py --config config/train-vqa-base-4gpu.json \
        --output_dir $VQA_EXP
    ```
3. inference
    ```
    python inf_vqa.py --txt_db /txt/vqa_test.db --img_db /img/coco_test2015 \
        --output_dir $VQA_EXP --checkpoint 6000 --pin_mem --fp16
    ```
    The result file will be written at `$VQA_EXP/results_test/results_6000_all.json`, which can be
    submitted to the evaluation server

### Visual Entailment (SNLI-VE)
NOTE: train should be ran inside the docker container
1. download data
    ```
    bash scripts/download_ve.sh $PATH_TO_STORAGE
    ```
2. train
    ```
    horovodrun -np 2 python train_ve.py --config config/train-ve-base-2gpu.json \
        --output_dir $VE_EXP

### Referring Expressions
1. download data
    ```
    bash scripts/download_re.sh $PATH_TO_STORAGE
    ```
2. train
    ```
    python train_re.py --config config/train-refcoco-base-1gpu.json \
        --output_dir $RE_EXP
    ```
3. inference and evaluation
    ```
    source scripts/eval_refcoco.sh $RE_EXP
    ```
    The result files will be written under `$RE_EXP/results_test/`

Similarly, change corresponding configs/scripts for running RefCOCO+/RefCOCOg.


## Pre-tranining
download
```
bash scripts/download_indomain.sh $PATH_TO_STORAGE
```
pre-train
```
horovodrun -np 8 python pretrain.py --config config/pretrain-indomain-base-8gpu.json \
    --output_dir $PRETRAIN_EXP
```
Unfortunately, we cannot host CC/SBU features due to their large size. Users will need to process
them on their own. We will provide a smaller sample for easier reference to the expected format soon.


## Acknowledgement:


```
@inproceedings{chen2020uniter,
  title={Uniter: Universal image-text representation learning},
  author={Chen, Yen-Chun and Li, Linjie and Yu, Licheng and Kholy, Ahmed El and Ahmed, Faisal and Gan, Zhe and Cheng, Yu and Liu, Jingjing},
  booktitle={ECCV},
  year={2020}
}
```


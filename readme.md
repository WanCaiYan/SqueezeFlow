# SqueezeFlow: Adaptive Text-to-Speech in Low Computational Resource Scenarios


## Abstract

Adaptive text-to-speech (TTS) has many important applications on edge devices, such as synthesizing personalized voices for the speech impaired, producing customized speech in translation apps, etc. However, existing models either require too much memory to adapt on the edge or too much computation for real-time inference on the edge. On the one hand, some auto-regressive TTS models can run inference in real-time on the edge, but the limited memory available on edge devices precludes training these models through backpropagation to adapt to unseen speakers. On the other hand, flow-based models are fully invertible, allowing efficient backpropagation with limited memory; however, the invertibility requirement of flow-based models reduces their expressivity, leading to larger and more expensive models to produce audio of the same fidelity. In this paper, we propose a flow-based adaptive TTS system with an extremely low computational cost, which is achieved through manipulating dimensions of the "information bottleneck" between a series of flows. The system, which requires only 7.2G MACs for inference (42x smaller than its flow-based baselines), can run inference in real-time on the edge. And because it is flow-based, the system also has the potential to perform adaptation with the limited amount of memory available at the edge. Despite its low cost, we show empirically that the audio generated by our system matches target speakers' voices with no significant reduction to fidelity and audio naturalness compared to baseline models.


Audio examples: https://low-cost-adaptive-tts.github.io/SqueezeFlow-Demo


## Credits

- Blow: https://github.com/joansj/blow
- WaveGlow: https://github.com/NVIDIA/waveglow


## Results

Our SqueezeFlow (SF) model can achieve an audio naturalness (MOS) and similarity scores as below. GT is ground truth audio, Blow is our baseline which uses ground truth audio for voice conversion, SF is our model which converts and generates audio from mel-spectrograms, SF+WG is a model for abelation which uses SF for convertion and WaveGlow for audio generation. Seen and Unseen refers to the target speaker being seen or unseen during training. For details on our evaluation, please refer to our paper.

Table 1 （Similarity to Source - the lower the better; Similarity to Target - the higher the better）:

| Model                 |    MOS    | Similarity to Source | Similarity to Target  |
| ---------------       | --------- | --------- | ----- |
| GT (target)           | 4.28±0.05 | 13.98% | 82.86%   |
| Blow, Seen            | 2.85±0.06 | 18.3%  | 36.5%    |
| SF, Seen              | 2.80±0.06 | 5.3%   | 38.8%    |
| SF+WG, Seen           | 2.96±0.06 | 17.5%  | 35.1%    |
| SF, Unseen            | 2.55±0.06 | 15.8%  | 22.3%    |
| SF+WG, Unseen         | 2.84±0.06 | 14.3%  | 24.9%    |

For an abelation study on the SqueezeFlow vocoder, we compare it against WaveGlow (See the table below). We also introduce 4 variants of SqueezeFlow vocoder in our paper, and we present their results here. Details on the evaluation are in the paper.

Table 2:

   | Model            | length | n_channels| MACs  | Reduction | MOS       |
   | ---------------  | ------ | --------- | ----- | --------- | --------- |
   |WaveGlow          |  2048  | 8         | 228.9 | 1x        | 4.57±0.04 |
   |SqueezeFlow-V     |  128   | 256       | 3.78  | 60x       | 4.07±0.06 |
   |SqueezeFlow-V-64L |  64    | 256       | 2.16  | 106x      | 3.77±0.05 |
   |SqueezeFlow-V-128S|  128   | 128       | 1.06  | 214x      | 3.79±0.05 |
   |SqueezeFlow-V-64S |  64    | 128       | 0.68  | 332x      | 2.74±0.04 |



# Reproduction

In our code, we use codenames: the SqueezeFlow converter is named after Blow: `blow-mel`, and the SqueezeFlow vocoder is called `SqueezeWave`. Corresponding code are in their respective folders.

## Installation

Suggested steps are:

1. Clone repository.
1. Create a conda environment (you can use the `environment.yml` file).
`conda env create -n test -f environment.yml`
1. `conda activate test; pip install tensorflow; pip install tensorboardX`
1. Install [Apex]
```1
   cd ../
   git clone https://www.github.com/nvidia/apex
   cd apex
   python setup.py install
```

## Reproducing Table 1

### Preprocessing

Download the [VCTK] dataset. 

1. `cd blow-mel/src`
1. To preprocess the audio files for VCTK:
`python preprocess.py --path_in=../VCTK/wav48 --extension=.wav --path_out=../VCTK_22kHz --sr=22050`
    - Our code expects audio filenames to be in the form `<speaker/class_id>_<utterance/track_id>_whatever.extension`, where elements inside `<>` do not contain the character `_` and IDs need not to be consecutive (example: `s001_u045_xxx.wav`). Therefore, if your data is not in this format, you should run or adapt the script `misc/rename_dataset.py`.

1. Prepare the VCTK dataset for seen/unseen speakers: 
`
mv VCTK_22kHz VCTK_22kHz_108
mkdir VCTK_22kHz_10
mkdir VCTK_22kHz_98
`
    - To use the same unseen speakers as us, copy folders `p236  p245  p251  p259  p264  p283  p288  p293  p298  p360` to `VCTK_22kHz_10` and others to `VCTK_22kHz_98`. Otherwise, randomly choose 10 speakers to be excluded from the dataset.

### Generate audio with our pretrained model

1. Download our [pretrained models]. Models contained in the folder are: 
    - `SqueezeFlow-C-108.*`, converter trained on full VCTK, used for inference for seen speakers
    - `SqueezeFlow-C-98.*`, converter trained on 98-speaker VCTK, used for inference for unseen speakers
    - `SqueezeFlow-C-10.*`, `SqueezeFlow-C-98` adapted on 10 unseen speaker VCTK, with embeddings for the 10 unseen speakers
    - `SqueezeFlow-V-VCTK`, vocoder trained on full VCTK

1. Inference for Seen speakers:
`python3 synthesize.py --base_fn_model=SqueezeFlow-C-108 --path_out=SqueezeFlow-C-108/out --sw_path=SqueezeFlow-V-VCTK --convert`

1. Inference for Unseen speakers:
`python3 synthesize_unseen.py --path_data_root=[parent folder of VCTK_22kHz_10 and VCTK_22kHz_98] --adapted_base_fn_model=SqueezeFlow-C-10 --trained_base_fn_model=SqueezeFlow-C-98 --path_out=[your output path] --sw_path=SqueezeFlow-V-VCTK --convert`


### Train your own model

#### Vocoder on VCTK

1. `cd SqueezeWave-adaptive`
1. Check `configs/config_a256_c256.json` and make sure all data paths are correct
1. Start training: `python3 train.py -c configs/config_a256_c256.json`
    - Substitute `train.py` with `distributed.py` if using multi-gpu.
1. Generate results: `python3 inference.py -c configs/config_a256_c256.json -w [path to your best checkpoint] -o [your output path]`. Checkpoints are saved every 2000 iterations by default.

#### Converter: Seen (full VCTK)

1. `cd blow-mel/src`
1. Start training: `python train.py --path_data=VCTK_22kHz_108 --base_fn_out=[your checkpoint path + experiment name] --model=blow --sw_path=[your best vocoder checkpoint]`
    - add `--multigpu` if using multi-gpu
1. Generate results using SqueezeWave vocoder: `python3 synthesize.py --base_fn_model=[your checkpoint path + experiment name] --path_out=[your output path] --sw_path=[your best vocoder checkpoint] --convert`
    - Best converter checkpoint is automatically save to `[your checkpoint path + experiment name]` when training
    - This step saves both the converted mel-spectrogram (as `*.pt`) and that mel-spectrogram turned into speech (using SqueezeWave)
1. To generate results using WaveGlow, clone your [WaveGlow] repository, go into its corresponding folder, and run `python3 inference.py -f <(ls [your SqueezeWave output path]/*.pt) -w waveglow_256channels_ljs_v3.pt -o [your WaveGlow output path] --is_fp16 -s 0.6`

#### Converter: Unseen (with 98/10 split on VCTK)

1. `cd blow-mel/src`
1. Training is similar to "Converter: Seen" section: `python train.py --path_data=VCTK_22kHz_98 --base_fn_out=[your checkpoint path + experiment name] --model=blow --sw_path=[your best vocoder checkpoint] --multigpu`
1. Adapt to unseen speakers: `python adapt.py --path_data=VCTK_22kHz_10 --base_fn_model=[your checkpoint path + experiment name] --path_out=[path to save your adapted model] --sw_path=[your best vocoder checkpoint] --sbatch=256 --multigpu --lr=1e-2`
1. Generate results on Unseen speakers using SqueezeWave: `python3 synthesize_unseen.py --path_data_root=[parent folder of VCTK_22kHz_10 and VCTK_22kHz_98] --adapted_base_fn_model=[best checkpoint to your adapted model on 10 speakers] --trained_base_fn_model=[best checkpoint to your trained model on 98 speakers] --path_out=[your output path] --sw_path=[your best vocoder checkpoint] --convert`
    - Similar to the previous section, the converted mel-spectrogram and generated audios will be saved to `[your output path]`
1. Use the same steps as in the previous section to generate audios with WaveGlow vocoder


## Reproducing Table 2 ([LJ Speech Data])

### Generate audio with our pretrained model 

1. Download our [pretrained vocoders]. We provide 4 pretrained models as described in the paper.
2. Download [mel-spectrograms]
3. Generate audio. Please replace `SqueezeWave.pt` to the specific pretrained model's name.

   ```python3 inference.py -f <(ls mel_spectrograms/*.pt) -w SqueezeWave.pt -o . --is_fp16 -s 0.6```


### Train your own model

1. Download [LJ Speech Data]. We assume all the waves are stored in the directory `^/data/`

2. Make a list of the file names to use for training/testing

   ```command
   ls data/*.wav | tail -n+10 > train_files.txt
   ls data/*.wav | head -n10 > test_files.txt
   ```

3. We provide 4 model configurations with audio channel and channel numbers specified in the table below. The configuration files are under ```/configs``` directory. To choose the model you want to train, select the corresponding configuration file.

4. Train your SqueezeWave model

   ```command
   mkdir checkpoints
   python train.py -c configs/config_a256_c128.json
   ```

   For multi-GPU training replace `train.py` with `distributed.py`.  Only tested with single node and NCCL.

   For mixed precision training set `"fp16_run": true` on `config.json`.

5. Make test set mel-spectrograms

   ```
   mkdir -p eval/mels
   python3 mel2samp.py -f test_files.txt -o eval/mels -c configs/config_a128_c256.json
   ```

6. Run inference on the test data. 

   ```command
   ls eval/mels > eval/mel_files.txt
   sed -i -e 's_.*_eval/mels/&_' eval/mel_files.txt
   mkdir -p eval/output
   python3 inference.py -f eval/mel_files.txt -w checkpoints/SqueezeWave_10000 -o eval/output --is_fp16 -s 0.6
   ```
   Replace `SqueezeWave_10000` with the checkpoint you want to test.



[pretrained vocoders]: https://drive.google.com/file/d/1RyVMLY2l8JJGq_dCEAAd8rIRIn_k13UB/view?usp=sharing
[pretrained models]: https://drive.google.com/drive/folders/1mskxbnMT-jgtXRBCMbky4X5sAO8VpGvs?usp=sharing
[mel-spectrograms]: https://drive.google.com/file/d/1g_VXK2lpP9J25dQFhQwx7doWl_p20fXA/view?usp=sharing
[LJ Speech Data]: https://keithito.com/LJ-Speech-Dataset
[Apex]: https://github.com/nvidia/apex
[VCTK]: https://homepages.inf.ed.ac.uk/jyamagis/page3/page58/page58.html
[Blow]: https://github.com/joansj/blow
[WaveGlow]: https://github.com/NVIDIA/waveglow

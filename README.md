This project contains the open-source dataset and code used in the paper. Two datasets were used in the experiment:

Basic Dataset. This experiment was conducted using the same smartphone paired with the same wireless charger, collecting longitudinal data over 10 days.

Extension Dataset. This dataset includes data collected from 4 additional smartphones and 5 different wireless chargers.

All datasets mentioned above have been published on the SciencedB platform (https://www.scidb.cn/detail?dataSetId=95f4f7b3a7854e918c18a7f31880e49b). 



This repository contains the dataset and MATLAB code for **MagHarp**, a passive sensing system that repurposes the electromagnetic leakage of an in-car wireless charging pad for mid-air gesture recognition. The system does not modify the wireless charger or the smartphone. It records body-coupled IQ signals, extracts harmonic-band spectro-temporal features, and classifies gestures with a lightweight Conv1D-BiLSTM model.

## Repository Contents

The public release is split into several archives because the dataset is large.

| Archive | Description | MD5 |
|---|---|---|
| `Code to be shared.zip` | MATLAB scripts for preprocessing, training, and fine-tuning. | `5a369da6418f5386f400dace8fbc34e0` |
| `MagHarp1-5.zip` | Base dataset. | `0b91ba7529da85ddefde91f03f159481` |
| `MagHarp6-10.zip` | Base dataset. | `2d5a74249bd6914ae7ddc8b14a079066` |
| `MagHarp11-13.zip` | Base dataset. | `b4fab7a7eeaa821b5cfb38e2a6055545` |
| `MagHarp14-16.zip` | Base dataset. | `53749bda27c1cabc7f17b88742986915` |
| `对比实验.zip` | Additional comparison, robustness, and condition-specific experiments. | `b2f33dc289260cee3f787c6a36a2e020` |

After downloading, verify each archive before extraction:

```bash
md5sum MagHarp1-5.zip
md5sum MagHarp6-10.zip
md5sum MagHarp11-13.zip
md5sum MagHarp14-16.zip
md5sum 对比实验.zip
```

## Dataset Overview

MagHarp includes the data used in the paper experiments.

### Base Dataset

The base dataset was collected using one smartphone and one wireless charger over 10 days. Each gesture sample is stored as a two-channel IQ waveform, where channel 1 is the I stream and channel 2 is the Q stream. The main paper protocol uses seven gesture classes:

| Folder name | English label |
|---|---|
| `按一次` | Single Press |
| `按两次` | Double Press |
| `长按` | Long Press |
| `左滑` | Single Swipe |
| `滑两次` | Double Swipe |
| `上滑` | Swipe Up |
| `下滑` | Swipe Down |

The released preprocessing script also contains optional labels. These are kept for users who want to process additional experiments, but the main paper model uses the seven classes listed above.

### Extended Dataset

The extended dataset includes data collected from four additional smartphones and five different wireless chargers. It is intended for cross-device, cross-charger, and deployment-condition evaluation. Use this dataset to test generalization or to run the lightweight fine-tuning/adaptation procedure.

### Expected Data Layout

The MATLAB scripts expect a class-folder layout. A typical raw-data folder should look like this:

```text
<raw_session_root>/
  按一次/*.wav
  按两次/*.wav
  长按/*.wav
  左滑/*.wav
  滑两次/*.wav
  上滑/*.wav
  下滑/*.wav
  background_or_no_gesture.wav
```

After preprocessing, the generated feature folder has the following structure:

```text
<session_root>/processed_stitch_envmed_256/
  按一次/*.mat
  按两次/*.mat
  长按/*.mat
  左滑/*.mat
  滑两次/*.mat
  上滑/*.mat
  下滑/*.mat
```

Each `.mat` file contains:

- `X1`: a `256 x 256 x 1` single-channel feature tensor.
- `meta`: optional metadata, including sampling rate, center frequency, STFT settings, harmonic-band settings, and source filename.

The public dataset already contains preprocessed `.mat` files, you can skip the preprocessing step and run training directly.

## Code Files

| File | Purpose |
|---|---|
| `Data_Preprocessing3.m` | Converts raw two-channel IQ `.wav` files into normalized `256 x 256 x 1` harmonic-band feature maps. |
| `Train_Conv1D_BiLSTM2_weighted.m` | Trains and evaluates the main Conv1D-BiLSTM classifier using day-based cross-validation and class-weighted loss. |
| `Fine_tuning_incar.m` | Loads a trained base model, selects a small calibration set from new data, fine-tunes `conv2`, `bn2`, and `fc`, and evaluates before/after adaptation. |

## Requirements

The code is written in MATLAB. Recommended environment:

- MATLAB R2022b or later.
- Signal Processing Toolbox.
- Image Processing Toolbox.
- Deep Learning Toolbox.
- Parallel Computing Toolbox and a CUDA-capable GPU are optional but recommended for training.

The scripts automatically fall back to CPU if no GPU is available.

## Preprocessing Raw IQ Data

Use `Data_Preprocessing3.m` if you start from raw `.wav` files.

1. Open `Data_Preprocessing3.m`.
2. Edit the following paths at the top of the script:

```matlab
rootDir   = '<path_to_raw_session_root>';
outDir    = fullfile(rootDir, 'processed_stitch_envmed_256');
bgWavPath = '<path_to_no_gesture_background_wav>';
```

3. Set the class list. For the main paper protocol, use:

```matlab
classNames = ["按两次","按一次","滑两次","左滑","上滑","下滑","长按"];
```

4. Run the script in MATLAB:

```matlab
run('Data_Preprocessing3.m')
```

The preprocessing pipeline performs:

1. Read two-channel IQ `.wav` files and form the complex signal `I + jQ`.
2. Compute STFT with `wlen = 4096`, `nfft = 8192`, and 50% overlap.
3. Estimate an environmental spectrum floor from a gesture-free background recording.
4. Subtract the background spectrum from each gesture sample.
5. Retain only harmonic bands around integer multiples of 360 kHz with a ±5 kHz half-bandwidth.
6. Apply 2D median filtering.
7. Stitch the retained harmonic bands along the frequency axis.
8. Pool the stitched feature map to `256 x 256` using p95 pooling.
9. Apply robust normalization and save `X1` as a `.mat` file.

## Training the Main Model

Use `Train_Conv1D_BiLSTM2_weighted.m` to reproduce the main classifier training.

1. Extract or generate the preprocessed data folders.
2. Open `Train_Conv1D_BiLSTM2_weighted.m`.
3. Update the `processedDir*` paths so that each path points to a `processed_stitch_envmed_256` folder.
4. Set the class list. For the main paper protocol:

```matlab
classNames = ["按两次","按一次","滑两次","左滑","上滑","下滑","长按"];
```

5. Set the day/session split by editing `processedDirs` and `testDirNums`.
6. Run:

```matlab
run('Train_Conv1D_BiLSTM2_weighted.m')
```

The script trains a sequence model with the following structure:

```text
Input sequence: 256 features x 256 timesteps
Conv1D(64, kernel=5) -> BatchNorm -> ReLU -> MaxPool -> Dropout(0.05)
Conv1D(128, kernel=5) -> BatchNorm -> ReLU -> MaxPool -> Dropout(0.15)
BiLSTM(128, OutputMode='last') -> Dropout(0.25)
Fully Connected -> Softmax -> Classification
```

The training script uses:

- Adam optimizer.
- Initial learning rate `3e-4`.
- Maximum `40` epochs.
- Mini-batch size `32`.
- Validation ratio `0.15` from non-test days.
- Day-based held-out testing to avoid temporal leakage.
- Class-weighted cross-entropy.
- Data augmentation, including time scaling, time shifting, contrast/gain jitter, noise injection, sparse dropout, and time-frequency masking.

Typical outputs include:

```text
models_kfold_byday_singleX1/
  fold*_test_*.mat
  fold_results.csv
  misclassified_*.csv
  confusion matrices
```

## Fine-Tuning and Deployment Adaptation

Use `Fine_tuning_incar.m` for new users, new phones, new chargers, or new in-vehicle operating conditions.

1. Train or select a base model saved by `Train_Conv1D_BiLSTM2_weighted.m`.
2. Open `Fine_tuning_incar.m`.
3. Update:

```matlab
modelPath = '<path_to_saved_base_model.mat>';
newProcessedDirs = ["<path_to_new_processed_stitch_envmed_256>"];
nCalPerClass = 5;
```

4. Run:

```matlab
run('Fine_tuning_incar.m')
```

The script selects the earliest `nCalPerClass` samples from each class as the calibration set and uses the remaining samples as the held-out test set. It freezes most of the network and fine-tunes only:

```text
conv2 + bn2 + fc
```

Default fine-tuning settings:

- Mini-batch size `4`.
- Maximum `10` epochs.
- Initial learning rate `5e-4`.
- `conv2` and `bn2` learning-rate factor `1`.
- `fc` learning-rate factor `5`.

Typical outputs include:

```text
adapt_conv2_bn2_fc_eval_<timestamp>/
  BeforeAdapt/
  AfterAdapt/
  summary.csv
  adapted_conv2_bn2_fc_model_<timestamp>.mat
```

## Reproducing Paper-Style Evaluation

For the base 10-day dataset, use day-based cross-validation rather than random sample-level splitting. This prevents temporal leakage between training and testing. The intended workflow is:

1. Unzip Basic dataset.
2. Use the preprocessed `.mat` folders directly, or run `Data_Preprocessing3.m` if starting from raw `.wav` files.
3. Set the corresponding `processedDirs` in `Train_Conv1D_BiLSTM2_weighted.m`.
4. Hold out complete recording days as test days.
5. Train the model and report test accuracy, macro-F1, and confusion matrices.

For the extended dataset, use the same feature format and either:

- evaluate the generic model directly, or
- run `Fine_tuning_incar.m` with a small calibration set from the target condition.

## Notes for Users

- The raw IQ files are large, so the dataset is distributed as multiple archive parts.
- The scripts contain local absolute paths from the authors' workstation. These paths must be replaced before running.
- The folder names are in Chinese because they were used as labels during data collection. The English mapping is provided above.
- The main model expects `.mat` files containing `X1` with shape `256 x 256 x 1`.
- The input feature map is converted into a `256 x 256` sequence before classification by applying row-wise z-score normalization and clipping.
- Training results can vary slightly because of random initialization, random validation sampling, and data augmentation. For strict reproducibility, set a fixed MATLAB random seed, for example `rng(0)`, before training.

## Citation

If you use this dataset or code, please cite the paper:

```bibtex
@article{magharp2026,
  title   = {Eyes-Free, Touch-Free, and Worry-Free: Repurposing Wireless Chargers for Safer In-Car Gesture Interaction},
  author  = {Anonymous Authors},
  journal = {Proceedings of the ACM on Interactive, Mobile, Wearable and Ubiquitous Technologies},
  year    = {2026},
  note    = {MagHarp dataset and code}
}
```

Please replace the anonymous author and publication fields with the final camera-ready citation after publication.


## Contact

For questions about the dataset or code, please contact the corresponding author listed in the final paper.

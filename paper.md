---
title: 'CWS-PResUNet: Music Source Separation with Channel-wise Subband Phase-aware ResUNet'
tags:
  - music
  - separation
  - channel-wise subband
  - resunet
authors:
  - name: Haohe Liu # note this makes a footnote saying 'co-first author'
    orcid: 0000-0003-1036-7888
    affiliation: "1, 2" # (Multiple affiliations must be quoted)
  - name: Qiuqiang Kong # note this makes a footnote saying 'co-first author'
    affiliation: 1
  - name: Jiafeng Liu
    affiliation: 1
affiliations:
 - name: Sound, Audio, and Music Intelligence (SAMI) Group, ByteDance
   index: 1
 - name: The Ohio State University
   index: 2
date: 10 Oct 2021
bibliography: paper.bib
# arxiv-doi: 10.21105/joss.01667
---

# Abstract  

Music source separation (MSS) shows active progress with deep learning models in recent years. Many MSS models perform separations on spectrograms by estimating bounded ratio masks and reusing the phases of the mixture. When using convolutional neural networks (CNN), weights are usually shared within a spectrogram during convolution regardless of the different patterns between frequency bands. In this study, we propose a new MSS model, channel-wise subband phase-aware ResUNet (CWS-PResUNet), to decompose signals into subbands and estimate an unbound complex ideal ratio mask (cIRM) for each source. CWS-PResUNet utilizes a channel-wise subband (CWS) feature to limit unnecessary global weights sharing on the spectrogram and reduce computational resource consumptions. The saved computational cost and memory can in turn allow for a larger architecture. On the MUSDB18HQ test set, we propose a 276-layer CWS-PResUNet and achieve state-of-the-art (SoTA) performance on `vocals` with an 8.92 signal-to-distortion ratio (SDR) score. By combining CWS-PResUNet and Demucs, our ByteMSS system ranks the 2nd on `vocals` score and 5th on average score in the 2021 ISMIR Music Demixing (MDX) Challenge limited training data track (leaderboard A). Our code and pre-trained models are publicly available^[Open sourced at: https://github.com/haoheliu/2021-ISMIR-MSS-Challenge-CWS-PResUNet].


# Introductions and Related Works
Music source separation aims at decomposing a music mixture into several soundtracks, such as `Vocals`, `Bass`, `Drums`, and `Other` tracks. It is closely related to topics like music transcription, remixing, and retrieval. Based on deep learning models, most of the early studies [@jansson2017singing;@takahashi2018mmdenselstm] perform separations in the frequency domain by estimating the ideal ratio masks (IRM) of the magnitude spectrogram and reusing the phase of the mixture. Later, time-domain models [@defossez2019demucs] start to demonstrate SoTA performance using direct waveform modeling, which does not involve transformations like short-time fourier transform (STFT). In this case, phase information can be implicitly estimated and models will not be restricted with the fixed time-frequency resolution. To enhance the MSS performance, @liu2020voice chose to employ a self-attension mechanism and Dense-UNet architecture. @choi2019investigating compared the performance of several types of UNet built with different intermediate blocks. To alleviate the computational cost, @kadandale2020multi designed a multi-task model to replace source-dedicated models. Also, @liu2020channel proposed to use the channel-wise subband feature to reduce resource consumptions and improve separation performance. Recently, @kong2021decoupling conducted an experiment on the MSS system theoretical upper bound, which proved the limitation of IRMs and the importance of phase estimation. 

In the next section, we will introduce the detailed architecture of CWS-PResUNet as well as ByteMSS, the system we submitted for the MDX Challenge [@mitsufuji2021music]. 

![Overview of the CWS-PResUNet and a comparison between using magnitude spectrogram and channel-wise subband spectrogram as the input feature.^[We use mono signal for simple illustration.]](graphs/main.png){ width=100% }

# Method

CWS-PResUNet is a ResUNet [@liu2021voicefixer] based model integrating the CWS feature [@liu2020channel] and the cIRM estimation strategies described in @kong2021decoupling. The overall pipeline is summarized in Figure 1a. We modeling separation on the subband spectrogram and phase domains. The analysis and synthesis filters in subband operations are designed by optimizing reconstruction error using the open-source toolbox^[https://www.mathworks.com/matlabcentral/fileexchange/40128-filter-bank-design].

As is illustrated in Figure 1b, the CWS feature has a lower frequency dimension and more channels compared with the full band spectrogram. To adapt the conventional full band CNN-based model to the CWS input feature, it just needs to modify the input and final output channel with internal CNN blocks unchanged. In this way, the internal feature map of the model becomes smaller, leading to a direct reduction in computational cost. Also, models become more efficient by enlarging receptive fields and diverging subband information into different channels.

The detailed computation procedure of our CWS-PResUNet model is described as follows. For a stereo mixture signal $x \in \mathbb{R}^{2\times L}$, where $L$ stands for signal length, we first utilize a set of analysis filters ${h}^{(j)},j=1,2,3,4$ to perform subband decompositions:
$$
x^{\prime}_{8\times \frac{L}{4}} = [\text{DS}_4({x_{2\times 1 \times L}}*{h}^{(j)}_{1\times 64})]_{j=1,2,3,4},
$$
where $\text{DS}_{4}(\cdot)$, $*$, and $[\cdot]$ denote the downsampling by 4, convolution, and stacking operators, respectively. The analysis filters we used are uniform filter banks with a filter length of 64. Then we calculate the STFT of the downsampled subband signals $x^{\prime}$ to obtain their magnitude spectrograms $|X^{\prime}|_{8\times T \times \frac{F}{4}}$, which is the input of Phase-aware ResUNet.

![The architecture of Phase-aware ResUNet](graphs/arc.png){ width=100% }

As is shown in Figure 2, the phase-aware ResUNet is a symmetric architecture containing a down-sampling and an up-sampling path with skip-connections between the same level. It accepts $|X^{\prime}|$ as input and estimates four tensors with the same shape: mask estimation $\hat{M}$, phase variation $\hat{P}_{r}$, $\hat{P}_{i}$, and direct magnitude prediction $\hat{Q}$. The complex spectrogram can be reconstructed with the following equation:
$$
\hat{S}^{\prime} = \text{relu}(|X^{\prime}|\odot \text{sigmoid}(\hat{M})+\hat{Q})\exp^{j(\angle X^{\prime} +\angle \hat{\theta})},
$$
in which $cos\angle \hat{\theta}=\hat{P}_{r}/(\sqrt{\hat{P}_{r}^2+\hat{P}_{i}^2})$ and $sin\angle \hat{\theta}=\hat{P}_{i}/(\sqrt{\hat{P}_{r}^2+\hat{P}_{i}^2})$. We pass the mask estimation $\hat{M}$ through a sigmoid function to obtain a mask with values between 0 and 1. Then by estimating $\hat{Q}$ and $\hat{\theta}$, models can avoid using mixture phase and estimating mask with only bounded values to calculate the unbounded cIRM. We use relu activation to ensure the positve magnitude value. Finally, after the inverse STFT, we perform subband reconstructions to obtain the source estimation $\hat{s}$:
$$ 
\hat{s}_{2\times L} = \sum_{j=1}^{4}(\text{US}_4(\hat{s}^{\prime}_{2\times 4\times \frac{L}{4}})*g^{(j)}_{4\times 64}),
$$
where $g^{(j)}, j=1,2,3,4$ are the pre-defined synthesis filters and $\text{US}_4(\cdot)$ is the zero-insertion upsampling function. 

Our model for `vocals` is optimized by calculating L1 loss between $\hat{s}$ and its target source $s$. Although we also use a model dedicated to separating the `other` track, we notice estimating and optimizing four sources together in one model can result in a 0.2 SDR [@vincent2006performance] gain on `other`. In this case, we not only use L1 loss on the waveform, but also employ energy-conservation loss, which calculates the L1 loss between the mixture and the sum of four source estimations. Our CWS-PResUNet models for `bass` and `drums` reported in the next section employ the same setup as the model for `other`.

<!-- Moreover, because bounded mask and mixture phase can limit the theoretical upper bound of the MSS system [@kong2021decoupling], we estimate unbounded mask and phase variations in each subband to compute the unbounded cIRM.  -->

In our ByteMSS system, we set up Demucs [@defossez2019demucs] to separate `bass` and `drums` tracks because it performs better than CWS-PResUNet on these two sources. Demucs is a time-domain MSS model. In our study, we adopted the open-sourced pre-trained Demucs^[https://github.com/facebookresearch/demucs] and do not apply the shift trick because it will slow down the inference speed. To separate the `vocals` track, we train a 276-layer CWS-PResUNet. For the `other` track, which is usually prone to overfitting due to the limited training data, we setup a smaller 166-layer CWS-PResUNet in order to achieve better generalization ability on the hidden test set. 

# Experiments

Our models are optimized using the training subset of MUSDB18HQ [@rafii2019musdb18]. We calculate the STFT of the downsampled 11.05 kHz subband signals with a window length of 512 and a window shift of 110. We use Adam optimizer with an initial learning rate of 0.001 and exponential decay. CWS-PResUNet takes approximately four days to train on a Tesla V100 GPU. During inference, we utilize a 10-second long `boxcar` windowing function [@schuster2008influence] with no overlapping to segment the signal. For evaluation, we report the SDR on the MUSDB18HQ test set with the open-sourced *museval* tool [@SiSEC18]. 

The subband analysis and synthesis operations usually cannot achieve perfect reconstruction. To assess the errors introduced by subband operations, we decompose the test set `vocals` tracks into 2,4, and 8 subbands and reconstruct them back to evaluate the reconstruction error of the filterbanks. We perform the computation using 32 bits float numbers. As is presented in Table 1, in all cases subband reconstructions achieve high performance with neglectable errors, which show an increasing trend with more subband numbers.

<!-- | Subband numbers |   2   |   4  |   8  |
|:---------------:|:-----:|:----:|:----:|
|       SDR       | 102.3 | 93.7 | 79.9 | -->


![](graphs/table1.png){ width=100% }

Table 2 lists the results of the baselines and our proposed systems. Our CWS-PResUNets achieve an SDR of 8.92 and 5.84 on `vocals` and `other` sources, respectively, outperforming the baseline X-UMX [@x-umx-sawata2021all], D3Net [@takahashi2020d3net], and Demucs systems by a large margin. Demucs performs better than CWS-PResUNet on `bass` and `drums` tracks. We assume that is because time-domain models can learn better representations than time-frequency features so are more suitable for separating percussive and band-limited sources. The average performance of our ByteMSS system is 6.97, marking a SoTA performance on MSS. Considering the high performance of the `vocals` model, we also attempt to separate three instrumental sources from `mixture` minus `vocals`. In this case, the average score remains 6.97, in which the `drums` score increase to 6.72 but the other three sources drop slightly. In the future, we will address the integration of time and frequency models for the compensations in both domains.

![](graphs/table2.png){ width=100% }

<!-- |    Models    | Vocals | Drums |  Bass | Other | Average |
|:------------:|:------:|:-----:|:-----:|:-----:|:-------:|
|     X-UMX    |  6.61  |  6.47  | 5.43  | 4.64  |  5.79  |
|     D3Net    |  7.24  |  7.01 |  5.25 |  4.53 |  6.01   |
|    Demucs    |  6.89  | **6.57**  | **6.53**  | 5.14  |  6.28   |
| CWS-PResUNet |  **8.92**  | 6.38  | 5.93  | **5.84**  |  6.77   |
|    ByteMSS   |  8.92  | 6.57  | 6.53  | 5.84  |  **6.97**   | -->

# Conclusions

Our experiment result shows CWS-PResUNet can achieve a leading performance on the separation of `vocals` and `other` tracks. And channel-wise subband feature is an effective alternative to magnitude spectrogram on music source separation task.

# Acknowledgements

This project is funded by ByteDance Inc. We acknowledge the supports from Haonan Chen for testing our system. 


# Reference
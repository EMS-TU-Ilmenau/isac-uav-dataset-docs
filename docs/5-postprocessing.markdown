---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: page
permalink: /postprocessing/
title: Data Postprocessing
nav_order: 5
---

# Applied Postprocessing Steps
Raw data from wireless channel measurement campaigns, as present in this dataset, takes a lot of file storage capacity.
Any reduction, e.g., via undersampling, is usually directly related to a loss of accessible channel properties.
For example, a reduction of bandwidth will result in a loss of delay (or *fast-time*) resolution.
However, with some reasonable assumptions about our objects of interest, we can reduce the amount of data without loosing too much information.

{: .note}
For ease-of-processing and sharing, we have reduced the total disk space requirements of our dataset to 200 GB.
The full data takes a total of 4 TB of disk space.

We also outline the remaining postprocessing steps here, that are not directly related to the reduction of data size.
They rather deal with accounting for the legal aspects in the used frequency band, synchronization error corrections, and hwo the CIRs (Channel Impulse Responses) are extracted.

## Rx Systems

The configuration parameters of our measurement system were introduced in the measurement section.  

| Parameter                | Value        | Comment                               |
| ------------------------ | ------------ | ------------------------------------- |
| Center Frequency         | 3.75 GHz     | Band center                           |
| Sampling Rate            | 100 MSPS     | MSamples/s                            |
| Bandwidth                | 80 MHz       | Effective bandwidth without Guardband |
| Symbol Length            | 16 $$\mu s$$ | OFDM Symbol length                    |
| Subcarrier Spacing       | 62.5 kHz     | Inverse of Symbol length              |
| Subcarriers (used/total) | 1280/1600    | 320 subcarriers used as Guardband     |

To avoid violation of spectral mask requirements for the used frequency band, we did not use the full 100 MHz bandwidth.
Instead, the subcarriers at the edges of the band were used as a guardband (in addition to bandpass filters at the transmitter).
This results in a usable bandwidth of 80 MHz.
The first step in the preprocessing is hence to remove the unused subcarriers, which already reduces the data size by 20%.
In addition, we performed a synchronization error correction based on the erros logged by the [FS740](https://www.thinksrs.com/products/fs740.html) oscillators.

## B2B Calibration and Channel Deconvolution

The received signal $$Y(f)$$ does not correspond to the desired CIRs $$H(f)$$, instead, $$Y(f)$$ is a superposition of multiple, differently shifted and distorted copies of the transmitted signal $$X(f)$$.
In frequency-domain notation we can write

$$ Y(f) = X(f) \cdot H(f)$$.

To obtain the CIRs we need to perform deconvoution of the received signal with the transmitted signal via

$$ H(f) = Y(f) \cdot X(f)^{-1}$$.

This works, because we use a multi-sine measurement signal with a nearly constant envelope for $$X(f)$$, hence, there are no zeros in the signal which would result in a division by zero.
In this step we use $$X_{\text{cal}}$$ instead of $$X(f)$$, which we obtaine from a Back-to-Back (B2B) calibration measurement.
The setup for the B2B calibration measurement is illustrated in the following figure.

...

The B2B calibration measurement is performed by connecting the Tx sequentially to each Rx by a coaxial cable.
Using the same Tx and Rx hardware as in the measurement, we recorded 1 minute of data $$X_{\textbf{b2b}}$$ for each Tx-Rx combination.
By also acounting for the frequency response of the coaxial cable, we can then (ideally) remove the frequency response of most used hardware components, except for the antennas.
To obtain $$X_{\text{cal}}$$, we first averaged over the frequency domain of all OFDM symbols in $$X_{\textbf{b2b}}$$ and computed 

$$ X_{\text{cal}} = \frac{X_{\textbf{b2b}}}{H_{\text{coax}}}$$

Hence, using $$X_{\text{cal}}$$ for the deconvolution means the resulting CIRs 
- do not contain the frequency response of the Tx and Rx hardware
- do not contain the frequency response $$H_{\text{coax}}$$ of the coaxial cables in the measurements (measured with a VNA)
- do not contain the transmitted waveform $$X(f)$$
- **do contain** the frequency response of the antennas 

Finally, after applying a delay correction to $$X_{\text{cal}}$$, the CIRs are obtained by computing
$$ H(f) = \frac{Y(f)}{X_{\text{cal}}(f)} $$

## Averaging and Downsampling

At this point, the measurement data $$H(f,t)$$ is stored in the uncompressed and takes a sizeable amount of disk space.
As the data still has a significant low OFDM symbol rate of $$16e-6$$, we decided to downsample the data in the slow-time domain.
This downsampling reduces the Doppler-Bandwidth of the data, but does not affect the delay resolution.

The original Doppler-Bandwidth $$B_\alpha$$ corresponds to the Subcarrier Spacing $$\Delta_{sc}$$ of the OFDM-like measurement signal.

$$B_\alpha \in -\frac{\Delta_{sc}}{2} ... +\frac{\Delta_{sc}}{2}$$

To compute the maximum Doppler-shift in the scenarios we need to consider the formula for the multistatic Doppler-shift $$\alpha$$ with static transmitter and receiver positions:

$$\alpha_{max} = \frac{2 v_{max}}{\lambda} \cos(\delta) \cos\left(\frac{\beta}{2}\right)  $$

We can see, that the maximum Doppler-shift occurs when the cosine terms have a magnitude of 1, then the maximum Doppler-shift is given by:

$$ \alpha_{max} = \frac{2 v_{max}}{\lambda} $$

With the carrier frequency of the signal (3.75 GHz) and the narrowband-approximation we get:

$$ \alpha_{max} = 2f_c\frac{v_{max}}{c_0} $$

For the UAV, we know the magnitude of the velocity vector at all times does not exceed $$v_{max} = 11 \text{m/s}$$ (40 km/h).
However, due to the urban propagation environment, we also need to consider other possible sensing targets, such as the pedestrians, bicyclists, trains, and more.
With some backoff factor, we decided on an maximum velocity limit of $$v_{max} = 50 \text{m/s}$$ (180 km/h).
The maximum required Doppler-Bandwidth in this case is given by:

$$ \alpha_{max} = 1250 \text{Hz} $$

To simplify the downsampling effort, we decided on a integert downsampling factor of $$D=20$$, which results in $$B_\alpha \in -1562 \text{Hz} ... +1562 \text{Hz}$$ and then 

$$ \begin{eqnarray}
\alpha_{max} &= 1562 \text{Hz} \\
B_\alpha &= 3124 \text{Hz}
\end{eqnarray} $$

Simply dropping 19/20 samples in $$H(f,t)$$ would result in an SNR loss the subsequent correlations, we decided to perform the downsampling by averaging over 20 consecutive symbols, resulting in a slow-time sampling interval of $$\Delta_t = 320 \mu s$$.

## Final Signal Parameters

This postprocessing results in the final signal parameters of the data in this dataset. 
For brevity, the following table only shows the changes with respect to the original signal parameters.

| Parameter                   | Value                      | Comment |
| --------------------------- | -------------------------- | ------- |
| Bandwith                    | 80 MHz                     |         |
| Subcarrier Spacing          | 62.5 kHz                   |         |
| Slow-Time Sampling Interval | 320 $$\mu s$$              |         |
| Doppler-Bandwidth           | -1.5625 kHz -  +1.5625 kHz |         |

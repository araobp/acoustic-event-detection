# Acoustic Event Detection with STM32L4 and TensorFlow

![](./oscilloscope/screenshots/spectrogram(psd).jpg)

Framenco (Bulerias)

## Background and motivation

Although AI is booming, most of AI researchers use open data on the web for training a neural network. However, I focus on special AED(Acoustic Event Detection) use cases for myself, and I need to collect a lot of data by myself. It is a very time-consuming work, so I need to develop a data collecting device that satisfies the following requirements:

- visualize sound in real-time: raw wave, FFT, spectrogram/mel-spectrogram and MFCCs.
- optimize parameters of MEMS mic parameters (DFSDM parameters) and filter/transform functions (pre-processing) to obtain the best sound image (mel-spectrogram) for training convolution layers of CNN.
- perform pre-processing on the edge: low-pass filtering, pre-emphasis and mel-spectorgram.
- collect data/image as an input to CNN.
- use the data collection device as an IoT edge device for deploying a trained CNN.
- low power consumption and small size.
- free development tools avaiable for developing the edge device.

## Platform and tool chain

### Platform

STMicro STM32L4 (ARM Cortex-M4 with DFSDM, DAC, UART etc) is an all-in-one MCU that satisfies all the requirements above:
- [STMicro NUCLEO-L476RG](https://www.st.com/en/evaluation-tools/nucleo-l476rg.html): STM32L4 development board
- [STMicro X-NUCLEO-CCA02M1](https://www.st.com/en/ecosystems/x-nucleo-cca02m1.html): MEMS mic evaluation board

In addition, I use Knowles MEMS mics to add extra mics to the platform above (for beam forming etc).

I already developed [an analog filter (LPF and AC couping)](https://github.com/araobp/stm32-mcu/tree/master/analog_filter) to monitor sound from DAC in real-time.

### Tool chain

- STMicro's [CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html) and [TrueSTUDIO(Eclipse/GCC/GDB)](https://atollic.com/truestudio/) for firmware development.
- Jupyter Notebook for simulation.
- IDLE and numpy/pandas/matplotlib/Tkinter for developing Oscilloscope GUI.
- Google's Colab to train CNN.

## IoT network

```
Sound/voice ))) [MEMS mic]-[DFSDM][ARM Cortex-M4(STM32L4)]--Bluetooth/LPWA/CAN---+
                                                                                 |
Sound/voice ))) [MEMS mic]-[DFSDM][ARM Cortex-M4(STM32L4)]--Bluetooth/LPWA/CAN---+--[gateway]--> IoT application on the cloud
                                                                                 |
Sound/voice ))) [MEMS mic]-[DFSDM][ARM Cortex-M4(STM32L4)]--Bluetooth/LPWA/CAN---+
                                     |           [DAC]
                                     |             |
                                 USB serial     [Analog filter] --> head phone for monitoring sound from mic
                                     |
                                     v
                           [Oscilloscope GUI(Tk)] ----------------------> Google Drive --> Google Colab for training CNN
```

Refer to this page for the analog filter: https://github.com/araobp/stm32-mcu/tree/master/analog_filter

## Making use of DMA

STMicro's HAL library supports "HAL_DFSDM_FilterRegConvHalfCpltCallback" that is very useful to implemente ring-buffer-like buffering for real-time processing.

I splitted buffers for DMA into two segments: segment A and segment B.

```
Sound/voice ))) [MEMS mic]-PDM->[DFSDM]-DMA->[A|B]->[ARM Cortex-M4]
                                                    [ARM Cortex-M4]->[A|B]->DMA->[UART] --- > PC(pyserial)
                                                    [ARM Cortex-M4]->[A|B]->DMA->[DAC] ))) Sound/Voice

```

All the DMAs are synchronized, because their master clock is the system clock.

## Sampling frequency

- The highest frequency on a piano is 4186Hz, but it generate overtones: ~10kHz.
- Human voice also generates overtones: ~ 10kHz.

So the samplig frequency of MEMS mic should be around 20kHz: 20kHz/2 = 10kHz ([Nyquist frequency](https://en.wikipedia.org/wiki/Nyquist_frequency))

## Parameters of DFSDM (digital filter for sigma-delta modulators) on STM32L4

- System clock: 80MHz
- Clock divider: 32
- FOSR/decimation: 128
- sinc filter: sinc3
- right bit shift: 3 (2 * 128^3 = 2^22, so 6-bit-right-shift is required to output 16bit PCM)
- Sampling frequency: 80_000_000/32/128 = 19.5kHz

## Pre-processing on STM32L4/CMSIS-DSP

```
      MEMS mic
         |
         V
   DFSDM w/ DMA
         |
  [16bit PCM data] --> DAC w/ DMA for montoring the sound with a headset
         |
  float32_t data
         |
         |                .... CMSIS-DSP APIs() .........................................
  [ AC coupling  ]-----+  arm_mean_f32(), arm_offset_f32
         |             |
  [ Pre-emphasis ]-----+  arm_fir_f32()
         |             |
[Overlapping frames]   |  arm_copy_f32()
         |             |
  [Windowins(hann)]    |  arm_mult_f32()
         |             |
  [   Real FFT   ]     |  arm_rfft_fast_f32()
         |             |
  [     PSD      ]-----+  arm_cmplx_mag_f32(), arm_scale_f32()
         |             |
  [Mel-spectrogram]----+  arm_dot_prod_f32()
         |             |
 [DCT Type-II(MFCCs)]  |  arm_rfft_fast_f32(), arm_scale_f32(), arm_cmplx_mult_cmplx_f32()
         |             |
         +<------------+
         |
 data the size of int8_t (in ASCII)
         |
         V
    UART w/ DMA
         |
         V
Oscilloscope GUI/IoT gateway
```

- My conclusion is that 80_000_000(Hz)/64(clock divider)/64(FOSR) with pre-emphasis(HPF) is the best setting for obtaining the best images of mel-spectrogram.
- I use a triangler filter bank to obtain mel-spectrogram, and I make each triangle filter having a same amount of area.

## Frame/stride/overlap

- number of samples per frame: 512
- length: 512/19.5kHz = 26.3msec
- stride: 13.2msec
- overlap: 50%(13.2msec)

```
26.3msec         stride
[b0|a1]            1a --> mel-scale spectrogram via filter bank or 12 MFCCs
   [a1|b1]         1b --> mel-scale spectrogram via filter bank or 12 MFCCs
      [b1|a2]      2a --> mel-scale spectrogram via filter bank or 12 MFCCs
         [a2|b2]   2b --> mel-scale spectrogram via filter bank or 12 MFCCs
            :
```
## Filter banks

Mel-scale spectrogram is used for training CNN

- Mel-scale: 40 filters (512 samples divided by (40 + 1))
- Linear-scale: 255 filters (512 samples divide by (255 + 1))

## Processing time (actual measurement)

In case of 1024 samples per frame:
- fir (cfft/mult/cifft/etc * 2 times): 17msec
- log10: 54msec
- log10 fast approximation: 1msec
- atan2: 53msec

Note: log10(x) = log10(2) * log2(x)

Reference: https://community.arm.com/tools/f/discussions/4292/cmsis-dsp-new-functionality-proposal

## log10 processing time issue

log10 of math.h is too slow for audio signal processing, but CMSIS DSP does not support log10. I decided to adopt [log10 approximation](./ipynb/log10%20fast%20approximation.ipynb).

## Command over UART (USB-serial)

UART baudrate: 921600bps

```

        Sequence over UART(USB-serial)

    ARM Cortex-M4L                    PC
           |                          |
           |<-------- cmd ------------|
           |                          |
           |------ data output ------>|
           |                          |


Data is send in ASCII characters, and the data format is as follows:

d: data delimiter
e: data transmission end

1,2,3,4,d,...,5,6,7,8,d\n
9,10,11,12,d,...,13,14,15,16,d\n
17,18,19,20,d,...,21,22,23,24,e\n

```

|cmd|description     | output size             | purpose               |
|---|----------------|-------------------------|-----------------------|
|0  | RAW_WAVE       | N x 1                   | Input to oscilloscope |
|1  | PSD            | N/2 x 1                 | Input to ML           |
|2  | FILTERBANK     | N/6 x NUM_FILTERS       | (for testing)         |
|3  | FILTERED_MEL   | NUM_FILTERS x 200       | Input to ML           |
|4  | MFCC           | NUM_FILTERS x 200       | Input to ML           |
|5  | MFCC_STREAMING | NUM_FILTERS x 07fffffff | (for testing)         |
|6  | FILTERED_LINEAR| NUM_FILTERS x 200       | Input to ML           |

|cmd|description     | output size             | purpose               |
|---|----------------|-------------------------|-----------------------|
|P  | Enable pre-emphasis |                    |                       |
|p  | Disable pre-emphasis |                   |                       |

## Oscilloscope GUI

I use Tkinter with matplotlib to draw graph of waveform, FFT, spectrogram, MFCCs etc.

![](./oscilloscope/screenshots/waveform.jpg)

![](./oscilloscope/screenshots/fft(psd).jpg)

- [Oscilloscope GUI implementation on matplotlib/Tkinter](./oscilloscope)

## CNN experiments on TensorFlow

I use the edge device above to obtain a lot of Mel-scale spectrograms for each class label, then I feed the data into CNN on Colab with GPU acceleration for training.

![](./tensorflow/filterd_spectrogram.png)

- [Experiments on Colab with GPU acceleration](./tensorflow)

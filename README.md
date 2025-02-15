# Python Tools for Speech Emotion Recognition (SER)

This repository contains the **Speech Emotion Recognition (SER)** tools developed during the development of [Mário Silva's dissertation](https://github.com/MarioCSilva/Speech_Emotion_Recognition_Thesis). It includes SER machine learning models and an audio pipeline to process audio in online or offline time to be used for SER classifications.

## SER Models

The developed models are available through the `SERClassifier` class in the file `ser_classifier.py`. It allows choosing the model developed and trained on the IEMOCAP dataset using a **traditional feature-based SER** and another using a **deep learning-based SER** approach.

This class has built-in methods that allow users to predict emotions from an audio file or directly from an audio vector. The audio must have **16000 Hz** frequency and be **mono** channel. The class also **preprocesses** the audio by **reducing the noise** and **trimming silence** at the beginning and end of the audio before extracting features for classification.

The traditional model constitutes an Extreme Gradient Boosting (XGBoost) model and utilizes manually extracted audio features from the audio signals to make emotional predictions.

The deep learning model utilizes transfer learning techniques, using a pre-trained ResNet-50 on the ImageNet dataset. This classifier utilizes the image of the audio spectrogram to make classifications.

Additionally, there is the `STRATIFIED` argument of the `SERClassifier` class, which allows selecting the usage of these models trained on limited data that achieved better results, based on a set of conditions that resulted from a study of the training dataset limitations.

### Usage Examples

Here is an example of how to use the classifier class:

    import librosa
    from ser_classifier import SERClassifier

    models_dir, audio_file = "path_to_ml_models_dir", "path_to_audio_file"

    trad_model = SERClassifier(config_file='config.json')

    print("Using the audio file path:")
    print(trad_model.predict(audio_file, is_file=True, return_proba=True))
    print(trad_model.predict(audio_file, is_file=True, return_proba=False))

    y, sr = librosa.load(audio_file, sr=16000)
    print("\nUsing the audio signal vector:")
    print(trad_model.predict(y, is_file=False, return_proba=True))
    print(trad_model.predict(y, is_file=False, return_proba=False))

Output:

    Using the audio file path:
    Probabilities: {'anger': 0.02343750, 'happiness': 0.08203125, 'sadness': 0.08593749, 'neutral': 0.80859375}
    Emotion Label: neutral

    Using the audio signal vector:
    Probabilities: {'anger': 0.02343750, 'happiness': 0.08203125, 'sadness': 0.08593749, 'neutral': 0.80859375}
    Emotion Label: neutral

## SER Pipeline

The audio pipeline for performing SER on a video conference system can be used both in online and offline time, and it contains several stages for properly identifying emotional content from the audio.

The first step of the pipeline is to continuously consume binary audio data of a video conference participant corresponding with a certain duration (defined in the class parameters).

The next step is converting the consumed binary data to an array of floats, and afterward, normalizing the audio signal. The normalization consists of, when necessary, **resampling** the audio to a sampling rate of 16000 Hz and converting the signal to **mono** by averaging samples across the channels.

The third step of the pipeline is to detect voiced speech of the previously consumed second of audio, using the **Silero Voice Activity Detection (VAD)** model (the minimum confidence level associated with the detection of voiced speech is one class parameter). 

Finally, the pipeline consumes a second of audio, it stores the segment if there is enough confidence that it detected voice activity, if it does not pass the threshold and it has previously saved any audio segment, it feeds it to a SER model to predict the emotion in the segment (it also takes into account if the segment reaches the minimum or maximum duration parameters).

### Usage Examples

The pipeline class is simple to use, it requires defining a set of parameters of the class, and then it must be fed consequent audio data every with at least 1 second of duration. There is an example of the pipeline showing a real-time progress plot of the detected emotions in the file `real_time_example.ipynb`, and, here is the code without the jupyter notebook plot:

    import pyaudio
    import numpy as np
    from ser_pipeline import SERPipeline

    # create the pipeline
    ser_pipeline = SERPipeline(config_file='config.json')

    while (True):
        # create an audio stream from the user microphone that reads 1 second of data each time
        stream = pyaudio.PyAudio().open(
            format=pyaudio.paFloat32,   # 32 bits
            channels=1,                 # mono
            rate=16000,                 # 16000 Hz
            input=True)

        # feed the pipeline every second
        proba = ser_pipeline.consume(stream.read(16000))
        if proba != None:
            print(f"Emotions Probabilities: {proba}")
            print(f"Recognized emotion: {max(proba, key=proba.get)}\n")


Output:

    Emotions Probabilities: {'anger': 0.1308593, 'happiness': 0.421875, 'sadness': 0.1796875, 'neutral': 0.267578}
    Recognized emotion: happiness

    Emotions Probabilities: {'anger': 0.2753906, 'happiness': 0.394531, 'sadness': 0.109375, 'neutral': 0.2207031}
    Recognized emotion: happiness


## Configurations

In addition to manually passing the parameters to both classes, there is also an option to pass a JSON configuration file. An example of this config is present in the file `config.json`:

    "MODELS_DIR": "models" -> Path for the directory where the machine learning models are stored
    "TRADITIONAL_SER": true -> Type of SER model to utilize
    "STRATIFIED": true -> Use SER models resulting from the stratification study
    "FORMAT": "float32" -> Data type of the audio samples fed to the pipeline
    "SAMPLE_RATE": 16000 -> Sample rate of the audio fed to the pipeline
    "NO_CHANNELS": 1 -> Number of audio channels of the audio fed to the pipeline (1 for mono, 2 for stereo)
    "MIN_CONFIDENCE": 0.6 -> Minimum confidence level for voice activity detection
    "MIN_DURATION": 1 -> Minimum duration of speech segments to be classified (in seconds)
    "MAX_DURATION": 6 -> Maximum duration of speech segments to be classified (in seconds)

---
title: "AiLive Mixer"
build:
  render: never
  list: never
  publishResources: false
---

<style>
/* Expand content width */
.container {
    max-width: 100% !important;
    width: 100% !important;
}

/* Center content */
.main-content {
    padding: 20px;
    margin: 0 auto;
}

/* Style the audio table for better responsive behavior */
.audio-table {
    width: 100%;
    border-collapse: collapse;
    margin: 20px 0;
    table-layout: fixed; /* This helps with equal column widths */
}

.audio-table th,
.audio-table td {
    padding: 12px;
    text-align: left;
    border-bottom: 1px solid #ddd;
    word-wrap: break-word; /* This prevents text from overflowing */
}

/* Make table responsive */
@media screen and (max-width: 1200px) {
    .audio-table {
        display: block;
        overflow-x: auto;
        white-space: nowrap;
    }
}
</style>

<style>
.figure-container {
    text-align: center;
    margin: 20px 0;
}

.figure-container img {
    max-width: 100%;
    height: auto;
    margin-bottom: 10px;
}

.figure-caption {
    font-style: italic;
    color: #666;
    margin-top: 10px;
}

.audio-table {
    width: 100%;
    border-collapse: collapse;
    margin: 20px 0;
}

.audio-table th, .audio-table td {
    padding: 12px;
    text-align: left;
    border-bottom: 1px solid #ddd;
}
</style>

# Abstract
In this work, we present a deep learning-based automatic multitrack music mixing system catered towards live performances. In a live performance, channels are often corrupted with acoustic bleeds of co-located instruments. Moreover, audio-visual synchronization is of critical importance, thus putting a tight constraint on the audio latency. In this work we primarily tackle these two challenges of handling bleeds in the input channels to produce the music mix with 0 latency. Although there have been several developments in the field of automatic music mixing in recent times, most or all previous works focus on offline production for isolated instrument signals and to the best of our knowledge, this is the first end-to-end deep learning system for live music performances. Our proposed system currently predicts mono gains for a multitrack input, but its design allows for easy adaptation to future work of predicting other relevant music mixing parameters.

# AiLive Mixer System Overview

<div class="figure-container">
    <img src="/images/AiLiveMixer-ICASSP.png" alt="System Overview">
    <div class="figure-caption">Fig.1.: AiLive Mixer - System Overview. Blue blocks to the left of the RATE-SPLIT-LINE operate with a frame size of F1 = 975 ms. Orange blocks to the right of the line operate with a frame size of F2. In this work we present results when setting F2 = 50 ms and when setting F2 = F1 = 975 ms</div>
</div>

Every raw audio channel is first passed through an audio embedding model to return information pertaining to the instrument type, followed by Transformer-encoder 1 to learn instrument type-related inter-channel context. Then we condition the returned embeddings with RMS information to inject information about the levels at which the channels are being received, followed by Transformer-encoder 2 to inject the multitrack relative levels information into the embeddings, followed by the GRU block to learn temporal context. We then pass the returned embeddings through the final Temporal-encoder 3 to learn temporally informed inter-channel context before passing through the MLP model to predict the gain parameter. The predicted gains for all channels are then applied to the respective audio channel waveforms before summing the waveforms together to obtain the resultant mono mix. All the neural networks share weights across all channels.

## Multi-Rate Processing

<div class="figure-container">
    <img src="/images/Multi-RateProcessing.png" alt="Multi-Rate Processing">
    <div class="figure-caption">Fig.2.: Multi-Rate Processing for a single channel. We set F1 frame size to be 975 ms and F2 frame size to be 50 ms when in Multi-Rate (MR) mode. In Single-Rate (SR) mode, F1 = F2 = 975 ms</div>
</div>

To enable low latency processing, we split the system into two rates. Typically audio embedding models require large temporal context to embed relevant information. Thus we extract longer frames (975 ms) to pass through the audio embedding model. Transformer-encoder 1 also operates on this frame size. Features such as RMS on the other hand, do not need such long receptive fields. Thus, we provide the capability to run the rest of the processing on shorter frame sizes, 50 ms in our case. By splitting the processing into two rates, the system can make gain predictions faster, without having to wait for the longer audio frames to become available.

We first extract an F1 frame (975 ms) and several non-overlapping F2 frames (50 ms) starting from the last 50 ms of the F1 frame. AiLive Mixer then predicts gains per F2 frame, thus resulting in several F2 frames of the mixed audio, which are then concatenated together to obtain the mix. Because the F2 frame is extracted from the end of the F1 frame, and we predict the gains for the F2 frames, the latency of the system is now essentially reduced to the frame size of F2. Theoretically a new F1 frame can be extracted for every new F2 frame, which would mean striding the F1 frame by the size of F2 frame. However, to manage computational load, we use the same F1 frame for several F2 frames, before extracting a new F1 frame. For the specific case of F2 frames being 50 ms we stride the F1 frame by 6 F2 frames, which accounts to a stride duration of 300 ms.

Note that setting F2 frame size to be equal to the size of F1 frames would mean that there are no rate splits and in this work we also provide results for this case (denoted as ALM-SR). In this case, both F1 and F2 frames are 975 ms and non-overlapping.

## Zero-Latency Processing

The multi-rate (MR) processing, reduces the latency of the system from 975 ms to 50 ms (or even lower for a lower F2 frame size). To then enable zero latency processing, for every F2 frame, we apply the gain that was predicted for the previous F2 frame. To enable this, we train the system in this fashion, by applying the gains predicted for a given frame, to the next frame before back propagating, thus expecting the model to make futuristic predictions. Although this can theoretically be achieved without MR processing, we hypothesize that through MR processing the model has to only predict the future by 50 ms, as opposed to having to predict 975 ms into the future as would be the case with SR processing. This training strategy of using zero latency processing will be referred to as zero latency training.

# Audio Results

Below we present some audio examples from the MedleyDB dataset which consists of raw tracks containing bleeds from other tracks, gain mixed with 0 latency. Results are provided for our models ALM-MR & ALM-SR along with the baseline DMC model variants DMC-OG & DMC-B-0L (DMC architecture with our proposed training strategies) and the Raw mix. Note that to mimic a live mixing scenario no normalization was applied to the generated results.

<div class="audio-table-container">
    <table class="audio-table">
        <thead>
            <tr>
                <th>ALM-MR</th>
                <th>ALM-SR</th>
                <th>DMC-B-0L</th>
                <th>DMC-OG</th>
                <th>RAW</th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>{{< audio src="audio/SasquatchConnection_ThanksALatte_1/MR-T3-Bleeds-0Lat_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/SasquatchConnection_ThanksALatte_1/SR-T3-Bleeds-0Lat_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/SasquatchConnection_ThanksALatte_1/dolby_bleeds_0lat_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/SasquatchConnection_ThanksALatte_1/dolby_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/SasquatchConnection_ThanksALatte_1/raw_mix.wav" >}}</td>
            </tr>
            <tr>
                <td>{{< audio src="audio/SasquatchConnection_ThanksALatte_2/MR-T3-Bleeds-0Lat_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/SasquatchConnection_ThanksALatte_2/SR-T3-Bleeds-0Lat_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/SasquatchConnection_ThanksALatte_2/dolby_bleeds_0lat_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/SasquatchConnection_ThanksALatte_2/dolby_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/SasquatchConnection_ThanksALatte_2/raw_mix.wav" >}}</td>
            </tr>
            <tr>
                <td>{{< audio src="audio/Schubert_Erstarrung/MR-T3-Bleeds-0Lat_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/Schubert_Erstarrung/SR-T3-Bleeds-0Lat_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/Schubert_Erstarrung/dolby_bleeds_0lat_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/Schubert_Erstarrung/dolby_pred_mix.wav" >}}</td>
                <td>{{< audio src="audio/Schubert_Erstarrung/raw_mix.wav" >}}</td>
            </tr>
            <!-- Your other rows remain the same -->
        </tbody>
    </table>
</div>
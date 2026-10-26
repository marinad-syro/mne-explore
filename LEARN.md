## MNE-Python
**What it is:** A Python library for processing and analyzing brain signal recordings — EEG, MEG, and related modalities.
**Why we're using it:** It's the standard open-source toolkit for this kind of neural time-series data, with built-in sample datasets so you can learn the workflow without your own recordings.
**How it works here:** `intro.py` walks through the core pipeline: `Raw` (continuous signal) → filter → find `events` (stimulus markers) → `Epochs` (trials cut around each event) → `Evoked` (trial-averaged response). Each of these is a distinct MNE object with its own `.plot()` method, e.g. `raw.plot()` vs `epochs.plot()` vs `evoked.plot()`.

## Virtual environment (`venv/`)
**What it is:** An isolated Python installation local to this project, separate from your system/conda Python.
**Why we're using it:** Keeps MNE and its dependencies from clashing with packages in other projects or your base conda environment.
**How it works here:** Activate it before running anything: `source venv/bin/activate` (from the project folder). Your prompt will show `(venv)` when it's active.

## `mne.datasets.sample`
**What it is:** A helper that downloads a real, publicly-shared MEG/EEG recording (auditory/visual stimulus experiment) the first time you call it.
**Why we're using it:** Lets you practice the real analysis workflow immediately, instead of needing your own EEG data first.
**How it works here:** `mne.datasets.sample.data_path()` downloads to `~/mne_data/` (a few hundred MB, one-time) and returns the local folder path.

## Stimulus events / trigger channel
**What it is:** A digital marker recorded alongside brain data marking exactly when something happened in the experiment (a stimulus presented, a cue given) — a small integer code sent as a voltage pulse on its own dedicated channel.
**Why it matters:** Epochs are cut relative to these markers, so understanding what a "1" or "2" in the events array actually corresponds to is essential before trusting any epoch-based analysis.
**How it works here:** `mne.find_events(raw, stim_channel="STI 014")` scans that channel for value changes and returns `(sample, prev_value, new_value)` rows — pure signal reading, no modeling. The human-readable `event_id` dict (e.g. `{"auditory/left": 1, ...}`) is documented metadata from the dataset itself, not something MNE computes.

## MEG vs. EEG polarity
**What it is:** In `evoked.plot_topomap()`, red/blue (positive/negative) means something different depending on channel type.
**Why it matters:** Misreading topomap polarity as directly comparable between MEG and EEG is a common mistake.
**How it works here:** EEG measures voltage *relative to a reference electrode* — polarity can flip if you change the reference. MEG magnetometers measure an absolute magnetic field direction (into vs. out of the scalp), tied via the right-hand rule to the direction of current flow in the underlying neurons — no reference needed. `evoked.plot_topomap(times=0.1)` with `ch_type=None` silently picks the first available type in priority order `mag → grad → planar1 → planar2 → eeg`, so in the sample dataset it plots **MEG magnetometers**, not EEG, unless you pass `ch_type="eeg"` explicitly.

## Gradiometers (`grad`) and planar1/planar2
**What it is:** A gradiometer measures the *spatial gradient* of the magnetic field (the difference between two nearby coils), rather than its absolute value like a magnetometer.
**Why it matters:** Distant noise sources produce a near-uniform field across both coils and cancel out, while nearby brain sources don't — gradiometers are MEG's main built-in noise rejection, like noise-canceling headphones.
**How it works here:** The sample dataset's MEG system (Neuromag/Elekta) puts two orthogonal planar gradiometers at each of ~102 sensor locations, labeled `planar1`/`planar2` (gradient along two perpendicular in-plane directions). Together these make up the `grad` channel type (204 channels). Plotting `ch_type="grad"` combines each planar1/planar2 pair via RMS, which is why grad topomaps never show negative values.

## Power Spectral Density (PSD) / Welch's method
**What it is:** A description of how a signal's power is distributed across frequencies, built on the idea that any signal decomposes into a sum of sine waves (Fourier transform) — power at a frequency ≈ squared amplitude of the sine wave needed there.
**Why it matters:** A raw FFT-based periodogram is noisy; Welch's method (used under the hood by `raw.compute_psd()`) averages periodograms over overlapping windowed segments of the signal to get a much less noisy estimate, at the cost of frequency resolution.
**How it works here:** Comes up any time you care about *rhythms* rather than waveform shape — e.g. the mu/beta band power changes used in `bci_motor_imagery.ipynb`'s motor imagery task, instead of a time-locked ERP shape.

## ICA for artifact removal (ECG/EOG)
**What it is:** Independent Component Analysis decomposes multi-channel data into statistically independent source components. Heartbeat (ECG) and eye blinks/movements (EOG) are the classic targets because both come from a spatially fixed dipole-like source that mixes into the sensors the same way every time — exactly what ICA assumes.
**Why it matters:** Lets you identify and subtract out one specific artifact's contribution per channel without throwing away the rest of the recording.
**How it works here:** Eye blinks: sparse, large, slow bumps (~200-400ms) with a tight, focal, frontal topography. Heartbeat: a fast, very regular repeating spike train (~1-1.5 Hz) with a broad, diffuse topography (the heart is far from the sensors, so it's less spatially focal). `ica.plot_sources()` + `ica.plot_components()` let you eyeball both; `ica.find_bads_eog()`/`find_bads_ecg()` confirm by correlating against recorded EOG/ECG channels. `ica.plot_overlay()` shows the before/after per channel directly — removing the right component should visibly shrink spikes concentrated at the channels (e.g. frontal EEG for blinks) where that component's mixing weight was largest.

## Raw plot vertical axis
**What it is:** Each line in `raw.plot()` shows amplitude in that channel's native physical unit — EEG in microvolts (µV), MEG magnetometers in femtotesla (fT), MEG gradiometers in fT/cm (a spatial gradient, not a field value).
**Why it matters:** EEG (~µV) and MEG (~fT) differ by ~9 orders of magnitude, so channels are plotted with a per-type `scalings` display factor just to make them visually comparable — stacked vertically with a fixed offset, not against one shared numeric axis. This is a zoom/display setting, not a unit conversion of the stored data.
**How it works here:** Comparing amplitude before vs. after ICA cleanup on the same channel is a direct, visual sanity check of which component you removed — see the ICA entry above.

## What EEG/MEG oscillation frequency actually measures
**What it is:** Not individual neuron firing rate. EEG/MEG mostly picks up synchronized postsynaptic (dendritic) currents summed across huge populations of similarly-oriented neurons — individual spikes are too brief and spatially scattered to show up. "Frequency" refers to how fast a large population rhythmically swings between high and low synchrony together, like a stadium wave (the wave has its own speed, independent of any one person's movement).
**Why it matters:** Sensorimotor cortex idles at ~8-30Hz (mu + beta) when not engaged; moving or *imagining* moving desynchronizes that population rhythm over the corresponding (contralateral) hemisphere — event-related desynchronization (ERD). That's the actual signal `bci_motor_imagery.ipynb`'s CSP step is built to detect.
**How it works here:** This is why filtering to 8-30Hz before CSP isn't "isolating the motor command frequency" — it's isolating the band where this task-relevant population-synchrony drop happens to live.

## Cross-validation, and Monte Carlo / ShuffleSplit specifically
**What it is:** Repeatedly splitting data into train/test subsets (test portion never seen during training) and averaging the resulting accuracy, to get an honest estimate of how a model generalizes rather than how well it memorized its own training data (overfitting).
**Why it matters:** With only 45 trials, one single train/test split is noisy — whichever specific trials land in the test set can make the result look better or worse than the model really is. Repeating the split many times and averaging smooths that noise out.
**How it works here:** k-fold (`cv=5`) partitions data into fixed, non-overlapping chunks, each used once as test. "Monte Carlo" cross-validation (`sklearn.model_selection.ShuffleSplit`) instead redraws a random train/test split independently each repeat (named after the casino — repeated random sampling to estimate something, same logic as averaging many dice rolls to estimate a die's bias). `cross_val_score(clf, X, y, cv=cv)` runs the whole repeated fit-train-test loop and returns one accuracy score per split; CSP must be refit fresh on each split's training data only, or test-set information leaks into training and inflates the score dishonestly.

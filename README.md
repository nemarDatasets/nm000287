[![DOI](https://img.shields.io/badge/DOI-10.82901%2Fnemar.nm000287-blue)](https://doi.org/10.82901/nemar.nm000287)

# Muse sleep-onset EEG

This dataset contains 540 at-home EEG recordings from 203 participants, collected
by Muse for Track 03 of the EEG/EMG Foundation Challenge 2026. The task is to
predict the time remaining until the first N2 sleep epoch using only EEG recorded
up to the time of prediction.

The recordings total approximately 157.52 hours and last about 6 to 30 minutes
each. All have four channels (TP9, AF7, AF8, TP10) sampled at 128 Hz, with one
annotation marking the first N2 epoch. Full sleep-stage annotations are not
included.

[Competition](https://neural-interfaces26.github.io/tracks.html) ·
[Track 03 on Codabench](https://www.codabench.org/competitions/17983/)

## Recordings and split

All 540 recordings are included in the NEMAR deposit, `nm000287`. The supplied
split is stored in each participant's `sub-<label>_sessions.tsv`:

| Split | Recordings | Participants |
| --- | ---: | ---: |
| Train | 500 | 203 |
| Test | 40 | 40 |

Every test participant also appears in training. This split therefore evaluates
new recordings from known participants, not generalization to unseen people.
It reproduces the supplied `splits.csv`; the assignments are included in the
session tables, so that external file is not needed to use the deposit.

The competition evaluates both seen and unseen participants using weighted
binned mean absolute error (W-bMAE). This deposit does not contain an unseen-person
test cohort. Follow the competition code for evaluation windows, target clipping
and scoring.

## Read the data

Recordings use EEG-BIDS with BrainVision `.vhdr`, `.vmrk` and `.eeg` files.
Read them with MNE-BIDS, which applies the units and scaling from the header and
loads the accompanying BIDS metadata:

```python
from mne_bids import BIDSPath, read_raw_bids

path = BIDSPath(
    root="nm000287", subject="001", session="001", task="sleeponset",
    datatype="eeg", suffix="eeg", extension=".vhdr",
)
raw = read_raw_bids(path)
```

Set `root` to your local dataset directory. The binary EEG contains multiplexed
32-bit floats; reading it without the header scaling gives incorrect amplitudes.

Each `events.tsv` contains one `n2_onset` event (`value=1`). Its `onset` is in
seconds from recording start, and `sample` is a zero-based sample index.
BrainVision marker positions are one-based. A `duration` of zero marks an instant,
not the length of an N2 epoch. The marker's `Stimulus` label is an export convention.
The N2 scoring method is not documented.

Session tables also contain sample counts, durations, N2 onset and signal-quality
flags; `sessions.json` defines these columns. `participants.tsv` contains recording
counts and total durations. Demographics and acquisition dates are unavailable.
Session onset times are relative to recording start, not necessarily lights out.
In EEG sidecars, `RecordingDuration` is the time to the last sample,
`(number_of_samples - 1) / 128`; session durations use `number_of_samples / 128`.

## Keep recording length out of the model

Every recording ends exactly 300 seconds after N2 onset. Knowing the full
recording length therefore reveals the target without using EEG:

```text
recording start                 first N2                 recording end
      |---------------------------|---------------------------|
      0                         onset                    onset + 300 s

At time t: use EEG up to t to predict onset - t.
```

For causal evaluation, hide total duration, sample counts, end-of-file information,
N2 annotations and future EEG from the model. Whole-recording quality summaries
and participant total durations must also remain outside the inputs. Preprocessing
must not use future samples. These files do not enforce those restrictions;
the evaluation pipeline must enforce them.

## Acquisition and signal quality

The curator identified the hardware as Muse S family. The exact generation,
firmware, reference and ground are unconfirmed. Headers identify pybv 0.7.5 as
the export software, not the acquisition software. The stored rate is 128 Hz.
Downsampling from 256 Hz is a curator-supplied assumption; the original rate,
resampling method and anti-aliasing filter have not been verified.

Prior filtering is unknown. `HardwareFilters: n/a` means that this information
is unavailable. The supplied 60 Hz power-line setting and channel cutoff fields
have been preserved but not independently verified. No filtering, resampling
or signal correction was performed during metadata preparation.

A screen of all 2,160 channel-recordings found no nonfinite samples or exactly
constant aligned two-second windows. Median Welch spectra flagged 726
channel-recordings at 50 Hz and 958 at 60 Hz, using peaks more than 10 dB above
adjacent bands. In 217 channel-recordings, more than 1% of samples exceeded
500 microvolts in absolute amplitude. At the recording level, 200 had 50 Hz
flags, 294 had 60 Hz flags and 128 had amplitude flags; these groups overlap.
The flags identify recordings to inspect, not a diagnosis of artifacts.

No isolated 50/60 Hz dip exceeded 10 dB below both spectral shoulders; apparent
60 Hz suppression against a combined baseline can reflect broad roll-off.
These spectra do not establish whether a notch filter was previously applied.
All source channels are marked `good`, but those labels are not independent
quality checks. Per-recording flags are described in `SubjectArtefactDescription`
and the session tables. Detailed audit scripts and spectra are not included in
this deposit.

Electrode and anatomical-landmark coordinates are identical across recordings,
in metres in the CapTrak frame. Their provenance is unknown; they should not be
treated as participant-specific measurements. Exact binary comparisons found
no duplicate EEG files, including across splits. Partial overlap and transformed
duplicates were not tested. Missing dates prevent checking chronological separation.

## Credit, consent and validation

Credit **Muse Team** and cite dataset `nm000287` with the version used. The data
are licensed under [CC-BY-NC-SA-4.0](LICENSE), matching the emg2pose release.

Muse determined internally that this collection was exempt from ethics review.
The depositor confirmed authorization and participant consent for sharing,
absence of identifiable personal information, and destruction of the
re-identification key.

Before upload, BIDS validation passed with no errors. All 540 recordings were
checked for consistent splits, channel order, sample counts and event timing.
Warnings remain for unavailable recommended metadata and collective authorship;
no HED tags are used. To validate a local copy, run:

```sh
nemar dataset validate nm000287
```

For the file conventions, see [EEG-BIDS (Pernet et al., 2019)](https://doi.org/10.1038/s41597-019-0104-8)
and [MNE-BIDS (Appelhoff et al., 2019)](https://doi.org/10.21105/joss.01896).

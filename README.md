# 🧠 DRIPS: Domain Randomisation for Image-based Perivascular Space Segmentation

DRIPS is a physics-inspired deep learning framework for accurate and generalisable segmentation of perivascular spaces (PVS) on structural MRI.
It uses domain randomisation to train on synthetic data and generalises across modalities (T1w, T2w) and resolutions (isotropic / anisotropic) without retraining or fine-tuning.

## 🧩 Pull the container
<pre><code>docker pull joseabernal/drips:latest</code></pre>

## ⚙️ Example usage
Run DRIPS on a single MRI image (e.g. T2-weighted) to segment PVS:
<pre><code>docker run --rm \
  -v $(pwd)/data:/data \
  joseabernal/drips:latest \
  --in /data/t2w.nii.gz \                     # Input MRI volume
  --out /data/t2w_seg.nii.gz \                # Output PVS segmentation mask
  --posterior /data/t2w_posterior.nii.gz \    # Posterior probability map
  --cropping 512 \                            # Inference crop size (power of two)
  --hyperintense 1 \                          # 1 = PVS appear hyperintense; 0 = hypointense
  --threshold 0.05 \                          # Probability threshold for binary mask
  --synthseg /data/t2w_synthseg_native.nii.gz # SynthSeg parcellation used as anatomical prior
</code></pre>

## 🧠 Notes
The input and output paths must be inside the mounted directory (/data here).

The argument --hyperintense should reflect the PVS contrast in your MRI sequence (T2w → 1, T1w → 0).

The --threshold can be adjusted (typical range: 0.01 – 1.00).

The model and required dependencies are pre-packaged in the container.

## 🧩 Running SynthSeg before running DRIPS
The following steps generate a SynthSeg parcellation aligned to the native image space, which is required to exclude segmentations outside of the region of interest (CSO + BG).
<pre><code>docker pull freesurfer/freesurfer:7.4.1</code></pre>

🔄 1. Run SynthSeg segmentation
<pre><code>docker run --rm \
  -v $(pwd)/test_data:/data \
  freesurfer/freesurfer:7.4.1 \
  mri_synthseg \
    --i /data/t2w.nii.gz \
    --o /data/t2w_synthseg.nii.gz \
    --resample /data/t2w_resampled.nii.gz \
    --cpu \
    --threads 5
</code></pre>
🔄 2. Compute rigid registration
<pre><code>docker run --rm \
  -v $(pwd)/license.txt:/usr/local/freesurfer/license.txt \
  -e FS_LICENSE=/usr/local/freesurfer/license.txt \
  -v $(pwd)/test_data:/data \
  freesurfer/freesurfer:7.4.1 \
  mri_robust_register \
    --mov /data/t2w_resampled.nii.gz \
    --dst /data/t2w.nii.gz \
    --lta /data/t2w_transf.lta \
    --satit \
    --iscale 0 \
    --maxit 1000
</code></pre>
🔄 3. Apply the transform to SynthSeg output
<pre><code>docker run --rm \
  -v $(pwd)/license.txt:/usr/local/freesurfer/license.txt \
  -e FS_LICENSE=/usr/local/freesurfer/license.txt \
  -v $(pwd)/test_data:/data \
  freesurfer/freesurfer:7.4.1 \
  mri_convert \
    -rt nearest \
    -oi \
    -at /data/t2w_transf.lta \
    -i /data/t2w_synthseg.nii.gz \
    -o /data/t2w_synthseg_native.nii.gz
    </code></pre>

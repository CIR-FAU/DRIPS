## 🧠 DRIPS: Domain Randomisation for Image-based Perivascular Space Segmentation

DRIPS is a physics-inspired deep learning framework for accurate and generalisable segmentation of perivascular spaces (PVS) on structural MRI.
It uses domain randomisation to train models exclusively on synthetic data, enabling robust performance across different modalities (T1w, T2w) and resolutions (isotropic and anisotropic) — without retraining or fine-tuning.

## Citation
If you use DRIPS in your research, please cite the following paper:
<pre>Bitar, L., Díaz, M., Duarte Coello, R., Valdés-Hernández, M. D. C., Mattern, H., Neumann, K., ... & Bernal, J. (2025). 
  DRIPS: Domain Randomisation for Image-based Perivascular Spaces Segmentation. medRxiv, 2025-10. 
  https://doi.org/10.1101/2025.10.22.25337423</pre>

## Prerequisites
### 1. Docker must be installed and running
<pre><code>docker --version</code></pre>

### 2. FreeSurfer is required to exclude segmentations outside of the region of interest
<pre><code>docker pull freesurfer/freesurfer:7.4.1</code></pre>

### 3. FreeSurfer licence file
You need a valid FreeSurfer license (license.txt) from
👉 https://surfer.nmr.mgh.harvard.edu/registration.html.
Save it in your working directory, as it will be needed later.

## 🚀 Step 1. Pull DRIPS
Download the latest DRIPS container from Docker Hub:
<pre><code>docker pull joseabernal/drips:latest</code></pre>

## 🗂️ Step 2. Prepare your working directory
Create a folder for your data and place your input MRI there:
<pre><code>mkdir data
cp your_t2w_image.nii.gz data/t2w.nii.gz
cp your_license.txt data/license.txt</code></pre>
Example structure:
<pre><code>data/
├── t2w.nii.gz
└── license.txt</code></pre>

## ⚙️ Step 3. Generate the region of interest 
These steps use the FreeSurfer 7.4.1 container to generate a SynthSeg parcellation aligned to your native image space.
This parcellation is used by DRIPS to restrict segmentation to relevant brain regions.

### 3.1 Run SynthSeg segmentation
<pre><code>docker run --rm \
  -v $(pwd)/data:/data \
  freesurfer/freesurfer:7.4.1 \
  mri_synthseg \
    --i /data/t2w.nii.gz \
    --o /data/t2w_synthseg.nii.gz \
    --resample /data/t2w_resampled.nii.gz \
    --cpu \
    --threads 5
</code></pre>
### 3.2 Compute rigid registration
<pre><code>docker run --rm \
  -v $(pwd)/license.txt:/usr/local/freesurfer/license.txt \
  -e FS_LICENSE=/usr/local/freesurfer/license.txt \
  -v $(pwd)/data:/data \
  freesurfer/freesurfer:7.4.1 \
  mri_robust_register \
    --mov /data/t2w_resampled.nii.gz \
    --dst /data/t2w.nii.gz \
    --lta /data/t2w_transf.lta \
    --satit \
    --iscale 0 \
    --maxit 1000
</code></pre>
### 3.3 Apply the transform to SynthSeg output
<pre><code>docker run --rm \
  -v $(pwd)/license.txt:/usr/local/freesurfer/license.txt \
  -e FS_LICENSE=/usr/local/freesurfer/license.txt \
  -v $(pwd)/data:/data \
  freesurfer/freesurfer:7.4.1 \
  mri_convert \
    -rt nearest \
    -oi \
    -at /data/t2w_transf.lta \
    -i /data/t2w_synthseg.nii.gz \
    -o /data/t2w_synthseg_native.nii.gz
    </code></pre>

After this step, you should have:
<pre><code>data/
├── t2w.nii.gz
├── t2w_resampled.nii.gz
├── t2w_synthseg.nii.gz
├── t2w_synthseg_native.nii.gz
├── t2w_transf.lta
└── license.txt</code></pre>

## 🤖 Step 4. Run DRIPS
Run DRIPS on your MRI image (e.g. T2w) to segment PVS:
<pre><code>docker run --rm \
  -v $(pwd)/data:/data \
  joseabernal/drips:latest \
  --in /data/t2w.nii.gz \                     # Input MRI volume
  --out /data/t2w_seg.nii.gz \                # Output binary PVS segmentation mask
  --posterior /data/t2w_posterior.nii.gz \    # Posterior probability map
  --cropping 512 \                            # Inference crop size ()
  --hyperintense 1 \                          # 1 = PVS appear hyperintense (T2w), 0 = hypointense (T1w)
  --threshold 0.05 \                          # Posterior probability threshold for binary mask
  --synthseg /data/t2w_synthseg_native.nii.gz # SynthSeg parcellation used as anatomical prior
</code></pre>

## 🧾 Notes and Tips

### License requirement:
FreeSurfer commands (SynthSeg and mri_convert) require a valid license.txt mounted into the container.

### Mounting:
Input and output files must be located within the mounted folder (here /data).

### Contrast setting:
The --hyperintense flag should reflect your MRI contrast:

1 → PVS appear bright (T2w)

0 → PVS appear dark (T1w)

### Threshold tuning:
Defines the minimum posterior probability for classifying a voxel as PVS in the final binary mask. Lower values yield more sensitive but potentially noisier segmentations; higher values produce more conservative masks.

In practice, values around 0.05–0.10 generally provide robust results across modalities and cohorts. Adjust only if you observe systematic over- or under-segmentation in your data.

### Cropping parameter (--cropping):
DRIPS performs inference in patches of size defined by this parameter.
It should be set to the closest power of two (e.g. 128, 256, 512) that matches or slightly exceeds the largest spatial dimension of your MRI volume (i.e. the dimension with the most voxels).
This ensures efficient use of GPU memory and full coverage of the image during tiling.

#### Example:
If your input volume has dimensions 240 × 256 × 180, the largest dimension is 256 → the recommended cropping size is 256.
If your image is 320 × 400 × 300, use 512, which is the next power of two above 400.

### Output:
After completion, the following files are generated:
<pre><code>data/
├── t2w.nii.gz
├── t2w_resampled.nii.gz
├── t2w_synthseg.nii.gz
├── t2w_synthseg_native.nii.gz
├── t2w_transf.lta
├── t2w_seg.nii.gz         # Binary PVS mask
├── t2w_posterior.nii.gz   # Posterior probability map
└── license.txt</code></pre>

## 🐛 Reporting Issues or Errors
If you encounter any errors while running DRIPS (e.g. Docker permission problems, missing libraries, unexpected outputs, or crashes), please open a new issue on the GitHub repository including:
* The exact command you ran
* A short description of the problem
* Relevant log excerpts (avoid uploading full images or patient data)

This helps improve DRIPS for the community and ensures that edge cases or environment-specific issues can be addressed quickly.

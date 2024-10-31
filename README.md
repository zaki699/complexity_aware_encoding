
# CAE (Complexity-Aware Encoding) Filter for FFmpeg

## Overview
The CAE (Complexity-Aware Encoding) Filter for FFmpeg is a highly optimized solution for detecting scene changes based on various frame metrics. It dynamically adapts to changes in video complexity and adjusts its encoding decisions accordingly, making it ideal for adaptive streaming, content-aware encoding, and resource-sensitive video applications.

This filter leverages multiple advanced metrics, including DCT energy, SSIM, histogram differences, Sobel edge detection, entropy, and color variance. It features ARM NEON optimizations, OpenMP parallelization, dynamic weighting, and adaptive thresholds for efficient and accurate scene change detection.

## Features
- **Multiple Complexity Metrics**: Uses SAD, SSIM, DCT Energy, Sobel Edge Detection, Histogram Difference, Entropy, and Color Variance to assess frame complexity.
- **Dynamic Adaptive Thresholding**: Calculates adaptive thresholds based on Median and MAD (Median Absolute Deviation), making it responsive to content variability.
- **NEON and OpenMP Optimizations**: Accelerates performance on ARM architectures with NEON intrinsics and parallelizes computations with OpenMP.
- **Adaptive Frame Interval**: Dynamically adjusts `frame_interval` based on activity scores to optimize performance during low and high activity periods.
- **Advanced Scene Change Detection**: Includes cooldown handling and confirmation logic to minimize false positives.

## Requirements
- **FFmpeg** with custom filter support.
- **ARM NEON** support (optional, improves performance on ARM-based architectures).
- **FFTW** for DCT computations.
- **OpenMP** for parallel processing (optional, recommended for multi-core systems).

## Installation
1. Clone and navigate to your FFmpeg source directory.
2. Copy the `cae.c` filter file into the appropriate FFmpeg filter directory (e.g., `libavfilter`).
3. Register the filter in FFmpeg by modifying the `Makefile` and `allfilters.c`.
4. Recompile FFmpeg:
    ```bash
    ./configure \
		--prefix="$BUILD_DIR" \
		--pkg-config-flags="--static" \
		--extra-cflags="-I/opt/homebrew/opt/fftw/include -Xpreprocessor -fopenmp -I$LIBOMP_PATH/include" \
		--extra-ldflags="-L/opt/homebrew/opt/fftw/lib -L$LIBOMP_PATH/lib  -lomp" \
		--extra-libs="-lfftw3 -lfftw3_threads -lpthread -lm " \
		--cc="clang" \
		--arch=arm64 \
		--enable-gpl \
		--enable-version3 \
		--enable-libx264 \
		--enable-libvmaf \
		--enable-nonfree \
		--disable-shared \
		--enable-static
    make -j$(nproc)
    sudo make install
    ```
5. Verify the installation:
    ```bash
    ffmpeg -filters | grep cae
    ```

## Usage
The CAE filter can be invoked in FFmpeg using the following syntax:

```bash
ffmpeg -i input.mp4 -vf "cae=alpha_complexity=0.5:alpha_ssim=0.8:alpha_hist=0.5" output.mp4
```

### Key Parameters
- **alpha_complexity**: Multiplier for complexity threshold (default: 0.5).
- **alpha_ssim**: Multiplier for SSIM threshold (default: 0.8).
- **alpha_hist**: Multiplier for histogram difference threshold (default: 0.5).
- **alpha_dct**: Multiplier for DCT energy threshold (default: 0.5).
- **alpha_sobel**: Multiplier for Sobel energy threshold (default: 0.5).
- **window_size**: Number of frames in the sliding window for threshold calculations (default: 10).
- **cooldown_frames**: Number of frames ignored after a confirmed scene change (default: 10).
- **required_consecutive_changes**: Number of consecutive frames above threshold to confirm a scene change (default: 2).
- **frame_interval**: Number of frames between scene change checks (default: 2).
- **min_frame_interval**: Minimum interval for adaptive frame interval adjustment (default: 1).
- **max_frame_interval**: Maximum interval for adaptive frame interval adjustment (default: 30).

### Example
To adjust the CAE filter with higher sensitivity to SSIM and DCT metrics:

```bash
ffmpeg -i input.mp4 -vf "cae=alpha_complexity=0.7:alpha_ssim=1.2:alpha_dct=1.0:window_size=15" output.mp4
```

### Advanced Configuration
- **Adaptive Frame Interval**: The filter will automatically increase the interval between processed frames when low activity is detected, conserving resources. This is controlled by the `min_frame_interval` and `max_frame_interval` parameters.
- **Dynamic Weighting**: Weights for each metric are adjusted based on recent frames, which allows the filter to emphasize different metrics as content changes. Use `max_weight` to cap the total weight.

## Technical Details
The filter calculates a weighted sum of normalized metrics for each frame. It maintains a sliding window for each metric, which helps adjust thresholds dynamically. Metrics are normalized using logarithmic scaling for stability, and outliers are managed using MAD to prevent threshold distortion.

### Metric Descriptions
- **SAD**: Sum of Absolute Differences between frames.
- **SSIM**: Structural Similarity Index, measuring visual similarity.
- **DCT Energy**: Energy of the DCT-transformed frame, indicating texture.
- **Sobel Edge Detection**: Gradient magnitude via Sobel operator.
- **Histogram Difference**: Chi-square difference between histograms of consecutive frames.
- **Entropy**: Measures randomness within the frame.
- **Color Variance**: Variance in color intensity, highlighting dynamic regions.

### Development Notes
- **FFTW**: Multi-threaded DCT computations enhance frequency analysis efficiency.
- **NEON Intrinsics**: ARM-specific optimizations speed up histogram and color variance calculations.
- **OpenMP Parallelization**: Applied to per-frame metrics for multi-core performance gains.

## Logs and Debugging
The filter logs metrics, computed weights, adaptive thresholds, and activity scores, providing insights for tuning. Enable verbose logging to observe frame-by-frame metric calculations:

```bash
ffmpeg -i input.mp4 -vf "cae=verbose=1" output.mp4
```

## License
This filter is distributed under the same license as FFmpeg, primarily LGPL v2.1. Contributions and forks are welcome to improve and extend functionality.

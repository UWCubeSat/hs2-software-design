# FOUND::SED

`SED` (Sobel Edge Detection) is an edge detection algorithm.

This algorithm will apply a 3x3 Sobel filter to a provided image, then scan it along a specified direction to mark points that have a steeper gradient than a specified threshold.
Once a scan ray finds a hit, that ray stops scanning.

`SED` receives any image, a 2D vector specifying a scan direction, and a float specifying a gradient threshold, and outputs a list of 2D points corresponding to edge points.

## Inputs/Outputs

### Images
Instances of `SED` require one image. The type of image required may be specified per instance (i.e. if used in conjunction with `FOUND::CRA` an instance would require a `HORIZON` image from the FOUND camera)

| Index | ImageType | Name |  Description | Optional |
|-------|----|---------|-------------|----------|
| 0 | `ANY` | image | Any image | no |


### Inputs
Instances of `SED` require a scan direction, usually corresponding to the direction of the sun.

| Index | TypeEnum | Name |  Description | Optional | Flag |
|-------|----------|------|--------------|----------|------|
| 0 | `Vec2` | scanDir | The direction to scan along | no | Scan Direction |


### Outputs
Instances of `SED` always return a list of edge points.

| Index | TypeEnum | Name | Description | Optional | Flag | Flag Value |
|-------|----------|------|-------------|----------|------|------------|
| 0 | `Vec2List` | edgePoints | The detected edge points | no | - | - |

### Flags
Instances of `SED` must be passed the following flags when run from the command line.

| Index | Name | Format | Description | Optional |
|-------|------|--------|-------------|----------|
| 0 | Threshold | `--threshold [Float]` | What value a filtered pixel has to be for the algorithm to consider it an edge | no |
| 1 | Scan Rays | `--scanRays [Int]` | The number of rays to scan along; The algorithm will output this many edge points | no |
| 2 | Scan Direction | `--scanDir [Vec3]` | What direction to scan along | no |
| 3 | Image | `--imagePath [path]` | The filepath to the image | no |

## Libraries
This algorithm uses the Sobel filter from OpenCV

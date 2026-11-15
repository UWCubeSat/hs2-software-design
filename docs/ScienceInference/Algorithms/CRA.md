# FOUND::CRA

`CRA` (Christian-Robinson Algorithm) is a position determination algorithm

This algorithm is the core of FOUND. `CRA` receives edge points corresponding to Earth's horizon in an image and uses the curvature to determine the satellite's position.

`CRA` receives a list of 2D edge points with at least three elements and a quaternion describing the satellite's attitude in ECI coordinates, 
and outputs a 3D position describing the satellite's position in ECI coordinates.

## Inputs/Outputs

### Images
Instances of `CRA` do not require any images.

### Inputs

| Index | TypeEnum | Name |  Description | Optional | Flag |
|-------|----------|------|--------------|----------|------|
| 0 | `Vec2List` | edgePoints | Points that lie on earth's horizon in an image | no | `points` |
| 1 | `Quaternion` | satelliteAttitude | The satellite's attitude in ECI coordinates | no | `sat-attitude` |


### Outputs

| Index | TypeEnum | Name | Description | Optional | Flag | Flag Value |
|-------|----------|------|-------------|----------|------|------------|
| 0 | `Vec3` | position | The satellite's position in ECI coordinates | no | - | - |


### Flags
Instances of `CRA` must be passed the following flags when run from the command line.

| Index | Name | Format | Description | Optional |
|-------|------|--------|-------------|----------|
| 0 | Points | `--points [Vec2List]` | The list of edge points on the image | no |
| 1 | Focal Length | `--camera-focal-length [Float]` | The focal length of the camera in *meters* | no |
| 2 | Pixel Size | `--camera-pixel-size [Float]` | The width of one pixel in *meters* (square pixel assumed) | no |
| 3 | Resolution | `--camera-resolution [Vec2]` | The (width, height) of an image the camera takes in pixels | no |
| 4 | Satellite Attitude | `--sat-attitude [Quaternion]` | The attitude of the satellite in ECI coordinates | no |
| 5 | Camera Orientation | `--camera-orientation [Quaternion]` | The orientation of the camera in local bodyframe coordinates (LOST cam positive z, FOUND cam positive y) | no |
| 6 | Artemis Offset | `--earth-radius-offset [Float]` | Pretend the Earth's radius is bigger by this many meters to mitigate the effects of the atmosphere | yes |


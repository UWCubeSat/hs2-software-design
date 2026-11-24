# FOUND::Feature Mapping

`Feature Mapping` is a position determination algorithm. This algorithm is an extended success for the HS2 mission and serves as an alternative to horizon detection as a means of determining our position in orbit.

The details of the algorithm are still going through iteration and refinement. As it exists today, `Feature Mapping` begins with SuperPoint, which produces a feature dictionary containing keypoint coordinates and descriptors identified within a given image. After removing invalid coordinates, LightGlue compares these keypoints to coordinates within our photo database to identify which feature we are able to see. Since we know the positions of these features, we can then identify our position. This strategy is still being compared and tested against leveraging SIFT for feature mapping to accomplish the same objective.

`Feature Mapping` receives an image of the Earth and an optional search_prior parameter. It outputs a 3D position describing the satellite's position in ECI coordinates or requests a new photo.

## Inputs/Outputs

### Images
Instances of `Feature Mapping` may require the following images. An algorithm instance may specify which camera to use for each image. 

| Index | ImageType | Name |  Description | Optional |
|-------|----|---------|-------------|----------|
| 0 | EARTH | image1 | Picture of the Earth | no 

### Inputs
Instances of `Feature Mapping` may require the following input types. \
Note: EVERY input is a flag. The difference between a regular flag and an input is that regular flags may be hardcoded by the algorithm topology.

| Index | TypeEnum | Name |  Description | Optional | Flag |
|-------|----------|------|--------------|----------|------|
| 0 | ImageType::EARTH | image | This is the image of the Earth. | no | `image` |
| 1 | List | search_prior | Defines a general position or attitude if known. | yes | `search_prior` |

// Note: This is subject to change. Feature Mapping is currently exploring the idea of choosing whether it wants an image from the LOST or FOUND camera since these cameras perform differently based on the feature present in the image.

### Outputs
Instances of `Algorithm` may output the following types. Optional outputs have a corresponding flag specifying when they are used.

| Index | TypeEnum | Name | Description | Optional | Flag | Flag Value |
|-------|----------|------|-------------|----------|------|------------|
| 0 | Vec3 | position | The satellite's position in ECI coordinates. | no | - | - |

### Flags
// Note: Feature Mapping is still under development, so flags have not yet been determined.

## Libraries
Currently, the algorithm is developed in Python. This is subject to change. Thus, these are all Python-based packages:
1. global-land-mask: Determines whether a given location is located on land or Earth.
2. cv2: SIFT, mask erosion, and image matching
3. LightGlue: runs SuperPoint and LightGlue

Other libraries used are not relevant to the algorithm itself, but rather how the code is written.

## References
https://arxiv.org/pdf/2409.02825: Highlights SIFT and SuperPoint as two methods for feature mapping low-earth orbit satellite images.

## Testing
// Note: Feature Mapping is still under development, so a test procedure have not yet been determined.

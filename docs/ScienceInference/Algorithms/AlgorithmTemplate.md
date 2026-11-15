# SUBSYSTEM::Algorithm

`Algorithm` [(Expansion if acronym)] is a [position determination/attitude determination/position refinement etc.] algorithm

[General description of algorithm]

`Algorithm` receives [at least 3 image points; an image of the stars; a quaternion describing the satellite's attitude; etc.] and outputs a [position; updated position; filtered image; etc.]

## Inputs/Outputs

Note: Optional means an algorithm *instance* is allowed to declare it does not accept or output the corresponding thing. Nothing is ever optional once an algorithm instance is specified.

### Images
Instances of `Algorithm` may require the following images. An algorithm instance may specify which camera to use for each image. \
Images are required before the algorithm instance is reached in the topology, but not necessarily before the first algorithm in the topology is run.
For example, the algorithm topology may use one of algorithm1's outputs to specify the orientation of an image that algorithm3 uses.

[Put non-optional images first]
| Index | ImageType | Name |  Description | Optional |
|-------|----|---------|-------------|----------|
| 0 | Type | image1 | This image is an image of an image | no |
| 1 | Type | image2 | ... | yes/no |
| 2 | Type | image3 | ... | yes/no |
| ... | ... | ... | ... |


### Inputs
Instances of `Algorithm` may require the following input types. \
Note: EVERY input is a flag. The difference between a regular flag and an input is that regular flags may be hardcoded by the algorithm topology.

[Use types from AlgorithmEnums.md]
| Index | TypeEnum | Name |  Description | Optional | Flag |
|-------|----------|------|--------------|----------|------|
| 0 | Type | input1 | This input is an input| no | `input1` |
| 1 | Type | input2 | ... | yes/no | ... |
| 2 | Type | input3 | ... | yes/no | ... |
| ... | ... | ... | ... | ... |


### Outputs
Instances of `Algorithm` may output the following types. Optional outputs have a corresponding flag specifying when they are used.

| Index | TypeEnum | Name | Description | Optional | Flag | Flag Value |
|-------|----------|------|-------------|----------|------|------------|
| 0 | Type | output1 | This output is an output | no | - | - |
| 1 | Type | output2 | ... | yes | `flag` | `value` |
| 2 | Type | output3 | ... | yes/no | ... | ... |
| ... | ... | ... | ... | ... | ... | ... |

### Flags
Instances of `Algorithm` may be passed the following flags when run from the command line.

| Index | Name | Format | Description | Optional |
|-------|------|--------|-------------|----------|
| 0 | input1 | `--input1 [data]` | This flag specifies input1 | no |
| 1 | flag2 | ... | ... | yes |
| 2 | flag3 | ... | ... | yes/no |
| ... | ... | ... | ... | ... | ... | ... |

## Libraries
The libraries used along with links

## References
Any papers used to develop this algorithm

## Testing
How this algorithm was or is planned to be tested before launch


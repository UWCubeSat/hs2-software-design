## Inputs/Outputs

Data values that an algorithm can either take in as input from experiments.csv/a previous algorithm or give as output to a subsequent algorithm.
These data values are stored in a per-experiment csv file that is deleted upon experiment completion.
"Format" specifies how the data is stored in the algorithm data file.

Inputs and outputs are not required to be passed to neighboring algorithm nodes; i.e. algorithm 3 may use an output from algorithm 1.

| Enum | Name | Format | Description |
|------|------|--------|-------------|
| 0 | Int | `x` | `x` is a signed integer. |
| 2 | Float | `x` | `x` is a double precision float. |
| 4 | Vec2 | `x:y:z` | `x`,`y` are double precision floats. |
| 6 | Vec3 | `x:y:z` | `x`,`y`,`z` are double precision floats. |
| 8 | Quaternion | `x:y:z:w` | Most commonly used to describe satellite attitude or camera orientation. `x`,`y`,`z`,`w` are double precision floats. |
| 9 | QuaternionList | `num_x0:y0:z0:w0_x1:y1:z1:w1_[...]` | Describes a list of Quaternions. `num` is an unsigned integer describing the number of elements. `x[n]`,`y[n]`,`z[n]`, `w[n]` are double precision floats. |
| (2n+1) | [nth enum]List | `num_[data0]_[data1]_[...]` | Any odd numbered type is a list of the previous type. `num` is an unsigned integer specifying how many elements there are. QuaternionList has been provided as an example. |
***

## Image Types

Image subjects that an algorithm can request from a specific camera. Each image type (except ANY) corresponds to a unique orientation algorithm that `DataCollectionApplication` uses to point a camera before taking an image. However an image may satisfy more than one type requirement (i.e. a PRECISE image may also be a HORIZON image, and every image is an ANY image).
If two images are required simultaneously for an experiment, `DataCollectionApplication` will guarantee both cameras satisfy their ImageType requirement before collecting data.

| Enum | Name | Description | Orientation Algorithm |
|------|------|-------------|-----------------------|
| 0 | ANY | Any image | none |
| 1 | STARS | Any image that does not contain Earth, the moon, or the sun. | --placeholder-- |
| 2 | HORIZON | An image that contains a lit limb of Earth | --placeholder-- |
| 3 | PRECISE | An image taken from a specific attitude and camera orientation, specified in `placeholder.file` by ScienceApplication | --placeholder-- |
***

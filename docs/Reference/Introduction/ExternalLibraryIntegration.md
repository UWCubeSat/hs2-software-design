# External Library Integration

HuskySat-2 uses external C++ libraries for the optical-navigation and star-catalog processing mission. The libraries are invoked by Level 3 application components rather than by drivers or hardware managers.

| Library | Purpose | Integration |
|---------|---------|-------------|
| LOST | Lost-in-space star identification | C++ library invoked by `ScienceInferenceApplication` |
| FOUND | Follow-up optical navigation | C++ library invoked by `ScienceInferenceApplication` |
| SCOPE | Camera calibration from star image processing | C++ library invoked by `ScienceInferenceApplication`; uses LOST during calibration processing |

The libraries are built as CMake dependencies and wrapped by application-level interfaces. Science results are written through F' data-product services. Flagged raw images are retained on the external SSD and backed up to the microSD card when selected by the storage policy, subject to the final capacity and data-retention configuration.

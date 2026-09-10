# External Library Integration

HuskySat-2 uses external C++ libraries for the optical-navigation and star-catalog processing mission. The libraries are invoked by Level 3 application components rather than by drivers or hardware managers.

| Library | Purpose | Integration |
|---------|---------|-------------|
| LOST | Lost-in-space star identification | C++ library invoked by `ScienceInferenceApplication` |
| FOUND | Follow-up optical navigation | C++ library invoked by `ScienceInferenceApplication` |
| SCOPE | Star-catalog optical processing | C++ library invoked by `ScienceInferenceApplication`; uses LOST during calibration processing |

The libraries are built as CMake dependencies and wrapped by application-level interfaces. Science results are written through F' data-product services. Flagged raw images are retained in external flash for later downlink, subject to the storage and data-retention design that remains TBA.

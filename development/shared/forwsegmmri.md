---
title:
---

- Reslice the MRI volume in order to obtain homogeneous voxels (cubic)
- Segment the scalp with smooth/threshold operations
- Segment the brain compartments with SPM/Freesurfer
- With [development:fwdarch:#Morphology operators](/development/fwdarch/#Morphology operators) obtain the filled volumes
- Obtain the brain compartment as brain = grey+white+csf and then apply smooth/threshold operators

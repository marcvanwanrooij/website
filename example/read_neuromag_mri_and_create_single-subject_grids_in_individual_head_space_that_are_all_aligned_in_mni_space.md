---
title: Doing source-analysis with the created headmodel
tags:
---

{% include markup/danger %}
The below example code is hopelessly outdated (thus deprecated) and will probably not work anymore. This page is kept in place just for reference. If you ended up on this page because you are curious to learn about the creation of dipole grids from .fif mri, please look at [this](/example/create_single-subject_grids_in_individual_head_space_that_are_all_aligned_in_mni_space) example script.
{% include markup/end %}

This example script relies on the example script [Create mni-aligned grids in individual head_space](/example/create_single-subject_grids_in_individual_head_space_that_are_all_aligned_in_mni_space). But for Neuromag data there are some differences. First make a mni template as is done in the above mentioned example script.

    %%batch create_headmodel_neuromag
    %==================================================================
    %Loading mri of single subject and make a single shell head model
    %==================================================================

    filename_mri       = 'mri.fif';
    mri                = ft_read_mri(filename_mri);
    save mri mri

    filename_hdr       = 'data.fif';
    hdr                = ft_read_header(filename_hdr);

    %--------------
    %Neuromag specific
    %--------------
    %voxel coordinates of fiducials. can be  taken from the Neuromag GUI for MRI-MEG Integration
    %but x and y coordinates need to be swapped!

    rpa                = [y x z];
    nas                = [y x z];
    lpa                = [y x z]

    %----realign the mri to the correct coordinate system.
    %determine head coordinate system

    cfg                = [];
    cfg.fiducial.rpa   = rpa;
    cfg.fiducial.nas   = nas;
    cfg.fiducial.lpa   = lpa;
    cfg.coordsys       = 'neuromag';
    cfg.method         = 'fiducial';
    mri_real           = ft_volumerealign(cfg,mri);

    %segment the brain into gray, white and csf matter to later make a single
    %shell model.
    cfg                = [];
    cfg.template       = 'T1.nii'; %spm8
    cfg.coordsys       = 'neuromag';
    cfg.write          = 'no';
    cfg.name           = 'temp';
    [segmentedmri]     = ft_volumesegment(cfg, mri_real)

    %check how it looks; does the segmented mri fit into the mri? Probably not
    %because of neuromag coordinates (x and y are swapped) and a bug in volume
    %segment.
    cfg                = [];
    test               = segmentedmri;
    test.avg.pow       = test.gray+test.white;
    test.anatomy       = mri_real.anatomy;
    cfg.funparameter   = 'avg.pow';
    cfg.interactive    = 'yes';
    ft_sourceplot(cfg,test);

** Figure 1 The segmented mri**
{% include image src="/assets/img/example/read_neuromag_mri_and_create_single-subject_grids_in_individual_head_space_that_are_all_aligned_in_mni_space/segmri.jpg" %}

    %make the single_shell headmodel
    cfg                = [];
    hdm                = ft_prepare_singleshell(cfg,segmentedmri);

    %make the leadfield normalised to the mni template
    %normalise the mri first
    cfg                = [];
    cfg.template       = '/T1.nii'; %is in MNI coordinates, from templates/spm8
    cfg.downsample     = 2;
    cfg.coordsys       = 'neuromag';
    cfg.nonlinear      = 'no';
    norm               = ft_volumenormalise(cfg,mri);

    %make the leadfield
    %Use the final transform matrix of norm to make the grid.pos (and the template_grid)
    load Template_brain/template_grid.mat
    grid               = [];
    grid.pos           = warp_apply(inv(norm.cfg.final), template_grid.pos, 'homogenous')/10; %in cm
    grid.inside        = template_grid.inside;
    grid.outside       = template_grid.outside;
    grid.dim           = template_grid.dim;
    clear template_grid

    % also remember the normalization transformation matrix
    grid.transform     = norm.cfg.final;

    %to see if everything has worke
    %make a figure of the single subject headmodel, sensor positions and grid positions
    cfg                = [];
    cfg.vol            = hdm;
    cfg.grid           = grid;
    cfg.gradfile       = 'grad.mat';
    figure;
    ft_headmodelplot(cfg);

{% include image src="/assets/img/example/read_neuromag_mri_and_create_single-subject_grids_in_individual_head_space_that_are_all_aligned_in_mni_space/headmodel.png" %}

{% include image src="/assets/img/example/read_neuromag_mri_and_create_single-subject_grids_in_individual_head_space_that_are_all_aligned_in_mni_space/headmodel2.png" %}

## Doing source-analysis with the created headmodel

When you have then estimated the sources which happens in NM or CTF space, you have to replace the .pos field of the source or the result of **[ft_sourcedescriptives](/reference/ft_sourcedescriptives)** with the template_grid.pos to get it back into MNI space because the origins of the two spaces are different. When you then sourceinterpolate to the normalised mri, this should work!

    load f
    cfg                = [];
    cfg.grid           = grid;
    cfg.frequency      = 10;
    cfg.vol            = hdm;
    cfg.gradfile       = 'grad.mat';
    cfg.projectnoise   = 'yes';
    cfg.keeptrials     = 'no';
    cfg.keepfilter     = 'yes';
    cfg.keepcsd        = 'yes';
    cfg.keepmom        = 'yes';
    cfg.lambda         = 0.1 * mean(f.powspctrm(:,nearest(cfg.frequency)),1);
    cfg.method         = 'dics';
    cfg.feedback       = 'textbar';
    source             = ft_sourceanalysis(cfg,f);

    save source source

    load grid
    cfg                = [];
    source.dim         = grid.dim;
    sd                 = ft_sourcedescriptives(cfg,source);
    %-----MNI SPECIFIC
    %Because sourceanalysis worked in NM coordinates and the interpolation goes
    %in MNI coordinates, we have to replace the sd.pos by the
    %template_grid.pos
    load Template_brain/template_grid
    sd.pos             = template_grid.pos;
    %-----END

    load norm
    cfg                = [];
    cfg.parameter      = 'avg.nai';
    cfg.voxelcoord     = 'no';
    cfg.interpmethod   = 'linear';
    sdint              = ft_sourceinterpolate(cfg, sd, norm);

    %plotting the results
    cfg                = [];
    cfg.surfdownsample = 2;
    cfg.downsample     = 2;
    cfg.funparameter   = 'avg.nai';
    cfg.anaparameter   = 'anatomy';
    cfg.funcolormap    = 'jet';
    cfg.method         = 'ortho';

    figure;
    ft_sourceplot(cfg,sdint)

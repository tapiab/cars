Command line usage
==================

`cars_cli.py  <../../bin/cars_cli.py>`_ is the unique entry point for CARS command line usage. 
It enables two main steps : `prepare` and `compute_dsm` described in the following sections. 

.. code-block:: bash

    usage: cars_cli.py <command> [<args>]

    The cars_cli.py commands are:
       prepare                  Preparation for compute_dsm producing stereo-rectification
                                grid as well as an estimate of the disparity to explore.
       compute_dsm              Tile-based, concurent resampling in epipolar geometry, disparity
                                estimation, triangulation and rasterization


Prepare DSM production
======================

The prepare part will perform the following steps:

1. Compute the stereo-rectification grids of the input pair's images
2. Compute sift matches between the left and right images in epipolar geometry
3. Derive an optimal disparity range from the matches
4. Derive a bilinear correction model of the right image's stereo-rectification grid in order to minimize the epipolar error
5. Apply the estimated correction to the right grid
6. Export the left and corrected right grids


Command Description
-------------------

.. code-block:: bash

  usage: cars_cli.py prepare [-h]
                           [--loglevel {DEBUG,INFO,WARNING,ERROR,CRITICAL}]
                           [--epi_step EPI_STEP]
                           [--disparity_margin DISPARITY_MARGIN]
                           [--epipolar_error_upper_bound EPIPOLAR_ERROR_UPPER_BOUND]
                           [--epipolar_error_maximum_bias EPIPOLAR_ERROR_MAXIMUM_BIAS]
                           [--elevation_delta_lower_bound ELEVATION_DELTA_LOWER_BOUND]
                           [--elevation_delta_upper_bound ELEVATION_DELTA_UPPER_BOUND]
                           [--mode {dask,local}] [--nb_workers NB_WORKERS]
                           [--walltime WALLTIME] [--check_inputs]
                           injson outdir

  positional arguments:
    injson                Input json file
    outdir                Output directory

  optional arguments:
    -h, --help            show this help message and exit
    --loglevel {DEBUG,INFO,WARNING,ERROR,CRITICAL}
                        Logger level (default: INFO. Should be one of DEBUG,
                        INFO, WARNING, ERROR, CRITICAL)
    --epi_step EPI_STEP   Step of the deformation grid in nb. of pixels
                        (default: 30, should be > 1)
    --disparity_margin DISPARITY_MARGIN
                        Add a margin to min and max disparity as percent of
                        the disparity range (default: 0.02, should be in range
                        [0,1])
    --epipolar_error_upper_bound EPIPOLAR_ERROR_UPPER_BOUND
                        Expected upper bound for epipolar error in pixels
                        (default: 10, should be > 0)
    --epipolar_error_maximum_bias EPIPOLAR_ERROR_MAXIMUM_BIAS
                        Maximum bias for epipolar error in pixels (default: 0,
                        should be >= 0)
    --elevation_delta_lower_bound ELEVATION_DELTA_LOWER_BOUND
                        Expected lower bound for elevation delta with respect
                        to input low resolution DTM in meters (default: -1000)
    --elevation_delta_upper_bound ELEVATION_DELTA_UPPER_BOUND
                        Expected upper bound for elevation delta with respect
                        to input low resolution DTM in meters (default: 1000)
    --mode {pbs_dask,local_dask}   Parallelization mode
    --nb_workers NB_WORKERS
                        Number of workers (default:8, should be > 0)
    --walltime WALLTIME   Walltime for one worker (default: 00:59:00, should be
                          formatted as HH:MM:SS)
    --check_inputs        Check inputs consistency

`Command line usage:`

.. code-block:: bash

    $ cars_cli.py prepare preproc_input.json outdir

Input json file
---------------

The prepare input file (``preproc_input.json``) file is formatted as follows:

.. code-block:: json

    {
        "img1" : "/tmp/cars/tests/data/input/phr_reunion/left_image.tif",
        "color1" : "/tmp/cars/tests/data/input/phr_reunion/left_image.tif",
        "img2" : "/tmp/cars/tests/data/input/phr_reunion/right_image.tif",
        "mask1" : "/tmp/cars/tests/data/input/phr_reunion/left_mask.tif",
        "mask2" : "/tmp/cars/tests/data/input/phr_reunion/right_mask.tif",
        "srtm_dir" : "/tmp/cars/tests/data/input/phr_reunion/srtm",
        "nodata1": 0,
        "nodata2": 0
    }


The mandatory fields of the input json file are:

* The ``img1`` and ``img2`` fields contain the paths to the images forming the pair.
* The ``srtm_dir`` field contains the path to the folder in which are located the srtm tiles covering the production.
* ``nodata1`` : no data value of the image 1.
* ``nodata2`` : no data value of the image 2.

The other optional fields of the input json file are:

* ``mask1`` : external mask of the image 1 (convention: 0 is a valid pixel, other values indicate data to ignore)
* ``mask2`` : external mask of the image 2 (convention: 0 is a valid pixel, other values indicate data to ignore)
* ``color1`` : image stackable to ``img1`` used to create an ortho-image corresponding to the produced DSM. This image can be composed of XS bands in which case a PAN+XS fusion will be performed.

Input parameters
----------------

Some optional parameters of the command line impact the matching:

* ``epi_step`` parameter :  step of the epipolar grid to compute (in pixels in epipolar geometry).
* ``disparity_margin`` parameter :  Add a margin to min and max disparity as percent of the disparity range.
* ``epipolar_error_upper_bound`` parameter: expected epipolar error upper bound (in pixels).
* ``epipolar_error_maximum_bias`` parameter: value added to the vertical margins for the matching. If this parameter is different to zero then the shift produced by an potential bias on the geometrical models is compensated by taking into account the median shift computed from the img1 and img2 matches.
* ``elevation_delta_lower_bound`` parameter: expected lower bound of the altitude discrepancy with the input DEM (in meters).
* ``elevation_delta_upper_bound`` parameter: expected upper bound of the altitude discrepancy with the input DEM (in meters).

During its execution, this program creates a distributed dask cluster (except if the ``mode`` option is different than ``pbs_dask`` or ``local_dask``). In the logs, an internet address is displayed. It can be opened with firefox and displays a dashboard which enables to follow the tasks' execution in real time. The parameters ``nb_workers`` and ``walltime`` configures respectively dask cluster workers number and the maximum time of execution.

``cars_cli.py prepare`` has also a ``--check_inputs`` option which enables the check of the input data consistency, it is to say that:

* ``img1`` and ``img2`` only have one band, are readable with the OTB and have a RPC model. It is also checked that the data seem to be in the sensor geometry (positive pixel size).
* ``mask1`` has the same size as ``img1`` and, as well, that ``mask2`` has the same size as ``img2``.
* the ground intersection zone between ``img1`` and ``img2`` is not empty.
* the srtm given in input covers the ground intersection zone of ``img1`` and ``img2``. For information purposes, if it is not equal to 100%, the coverage ratio of the dem with respect to the useful zone is given in the logs.

By default this option is **deactivated** because it can be potentially time-consuming.

Input images
------------

To generate the images in epipolar geometry from the grids computed by cars and the original images, one can refer to the Orfeo Toolbox documentation `here <https://www.orfeo-toolbox.org/CookBook/recipes/stereo.html#resample-images-in-epipolar-geometry>`_ .

Output contents
---------------

After its execution, the ``outdir`` folder contains the following elements:

.. code-block:: bash

    ls outdir/
    yy-MM-dd_HHhmmm_prepare.log  dask_log                     left_envelope.dbf  left_envelope.shp  left_epipolar_grid.tif      lowres_elevation_diff.nc  matches.npy      right_envelope.dbf  right_envelope.shp  right_epipolar_grid.tif
    content.json                 envelopes_intersection.gpkg  left_envelope.prj  left_envelope.shx  lowres_dsm_from_matches.nc  lowres_initial_dem.nc     raw_matches.npy  right_envelope.prj  right_envelope.shx  right_epipolar_grid_uncorrected.tif

The ``content.json`` file lists the generated files and some numerical elements:

.. code-block:: json

    {
      "input": {
        "img1": "/tmp/cars/tests/data/input/phr_ventoux/img1.tif",
        "img2": "/tmp/cars/tests/data/input/phr_ventoux/img2.tif",
        "srtm_dir": "/tmp/cars/tests/data/input/phr_ventoux/srtm",
        "nodata1": 0,
        "nodata2": 0
      },
      "preprocessing": {
        "version": "master//xxx",
        "parameters": {
          "epi_step": 30,
          "disparity_margin": 0.02,
          "epipolar_error_upper_bound": 10.0,
          "epipolar_error_maximum_bias": 0.0,
          "elevation_delta_lower_bound": -1000.0,
          "elevation_delta_upper_bound": 1000.0
        },
        "static_parameters": {
          "sift": {
            "matching_threshold": 0.6,
            "n_octave": 8,
            "n_scale_per_octave": 3,
            "dog_threshold": 20.0,
            "edge_threshold": 5.0,
            "magnification": 2.0,
            "back_matching": true
          },
          "low_res_dsm": {
            "low_res_dsm_resolution_in_degree": 0.000277777777778,
            "lowres_dsm_min_sizex": 100,
            "lowres_dsm_min_sizey": 100,
            "low_res_dsm_ext": 3,
            "low_res_dsm_order": 3
          }
        },
        "output": {
          "left_envelope": "left_envelope.shp",
          "right_envelope": "right_envelope.shp",
          "envelopes_intersection": "envelopes_intersection.gpkg",
          "envelopes_intersection_bounding_box": [
            -58.589517087035645,
            -34.4931726206081,
            -58.58173610178845,
            -34.48677006524553
          ],
          "epipolar_size_x": 2407,
          "epipolar_size_y": 2510,
          "epipolar_origin_x": 0.0,
          "epipolar_origin_y": 0.0,
          "epipolar_spacing_x": 30.0,
          "epipolar_spacing_y": 30.0,
          "disp_to_alt_ratio": 2.5305049217664437,
          "raw_matches": "raw_matches.npy",
          "left_epipolar_grid": "left_epipolar_grid.tif",
          "right_epipolar_grid": "right_epipolar_grid.tif",
          "right_epipolar_uncorrected_grid": "right_epipolar_grid_uncorrected.tif",
          "minimum_disparity": -8.873300104758348,
          "maximum_disparity": 2.2324556746626323,
          "matches": "matches.npy",
          "lowres_dsm": "lowres_dsm_from_matches.nc",
          "lowres_initial_dem": "lowres_initial_dem.nc",
          "lowres_elevation_difference": "lowres_elevation_diff.nc",
          "corrected_lowres_dsm_from_matches": "corrected_lowres_dsm_from_matches.nc",
          "corrected_lowres_elevation_diff": "corrected_lowres_elevation_diff.nc"
        }
      }
    }


The other files are:

* ``left_epipolar_grid.tif`` : left image epipolar grid
* ``right_epipolar_grid.tif`` : right image epipolar grid with correction
* ``left_envelope.shp`` : left image envelope
* ``right_envelope.shp`` : right image envelope
* ``envelopes_intersection.gpkg`` : intersection of the right and left images' envelopes
* ``ground_positions_grid.tif`` : image with the same geometry as the epipolar grid and for which each point has for value the ground position (lat/lon) of the corresponding point in the epipolar grid
* ``matches.npy`` : matches list after filtering
* ``raw_matches.npy`` : initial matches list
* ``lowres_dsm_from_matches.nc`` : low resolution DSM computed from the matches
* ``lowres_elevation_diff.nc`` : difference between the low resolution DSM computed from the matches and the initial DEM in input of the prepare step
* ``lowres_initial_dem.nc`` : initial DEM in input of the prepare step corresponding to the two images envelopes' intersection zone
* ``corrected_lowres_dsm_from_matches.nc`` :  Corrected low resolution DSM from matches if low resolution DSM is large enough (minimum size is 100x100)
* ``corrected_lowres_elevation_diff.nc`` : difference between the initial DEM in input of the prepare step  and the corrected low resolution DSM. if low resolution DSM is large enough (minimum size is 100x100)

DSM production with compute\_dsm
================================

Once the prepare preprocessing step is done, the ``compute_dsm`` program will be in charge of:

1. **resampling the images pairs in epipolar geometry** (corrected one for the right image) by using SRTM in order to reduce the disparity intervals to explore,
2. **correlating the images pairs** in epipolar geometry
3. **triangulating the sights** and get for each point of the reference image a latitude, longitude, altitude point
4. **filtering the 3D points cloud** via two consecutive filters. The first one removes the small groups of 3D points. The second filters the points which have the most scattered neighbors. Those two filters are activated by default.
5. **projecting these altitudes on a regular grid** as well as the associated color

Command Description
-------------------

.. code-block:: bash

    usage: cars_cli.py compute_dsm [-h]
                                   [--loglevel {DEBUG,INFO,WARNING,ERROR,CRITICAL}]
                                   [--sigma SIGMA] [--dsm_radius DSM_RADIUS]
                                   [--resolution RESOLUTION] [--epsg EPSG]
                                   [--roi [ROI [ROI ...]]]
                                   [--mode {pbs_dask,local_dask,mp}]
                                   [--nb_workers NB_WORKERS] [--walltime WALLTIME]
                                   [--dsm_no_data DSM_NO_DATA]
                                   [--color_no_data COLOR_NO_DATA]
                                   [--corr_config CORR_CONFIG]
                                   [--min_elevation_offset MIN_ELEVATION_OFFSET]
                                   [--max_elevation_offset MAX_ELEVATION_OFFSET]
                                   [--output_stats] [--use_geoid_as_alt_ref]
                                   [--use_sec_disp] [--snap_to_left_image]
                                   [--align_with_lowres_dem]
                                   [--disable_cloud_small_components_filter]
                                   [--disable_cloud_statistical_outliers_filter]
                                   [injsons [injsons ...]] outdir

    positional arguments:
      injsons               Input json files
      outdir                Output directory

    optional arguments:
      -h, --help            show this help message and exit
      --loglevel {DEBUG,INFO,WARNING,ERROR,CRITICAL}
                            Logger level (default: INFO. Should be one of DEBUG,
                            INFO, WARNING, ERROR, CRITICAL)
      --sigma SIGMA         Sigma for rasterization in fraction of pixels
                            (default: None, should be >= 0)
      --dsm_radius DSM_RADIUS
                            Radius for rasterization in pixels (default: 1, should
                            be >= 0)
      --resolution RESOLUTION
                            Digital Surface Model resolution (default: 0.5, should
                            be > 0)
      --epsg EPSG           EPSG code (default: None, should be > 0)
      --roi [ROI [ROI ...]]
                            DSM ROI in final projection [xmin ymin xmax ymax] (it
                            has to be in final projection) or DSM ROI file (vector
                            file or image which footprint will be taken as ROI).
      --mode {pbs_dask,local_dask,mp}
                            Parallelization mode
      --nb_workers NB_WORKERS
                            Number of workers (default: 8, should be > 0)
      --walltime WALLTIME   Walltime for one worker (default: 00:59:00, should be
                            formatted as HH:MM:SS)
      --dsm_no_data DSM_NO_DATA
                            No data value to use in the final DSM file (default:
                            -32768)
      --color_no_data COLOR_NO_DATA
                            No data value to use in the final color image
                            (default: 0)
      --corr_config CORR_CONFIG
                            Correlator config (json file)
      --min_elevation_offset MIN_ELEVATION_OFFSET
                            Override minimum disparity from prepare step with this
                            offset in meters
      --max_elevation_offset MAX_ELEVATION_OFFSET
                            Override maximum disparity from prepare step with this
                            offset in meters
      --output_stats        Outputs dsm as a netCDF file embedding quality
                            statistics.
      --use_geoid_as_alt_ref
                            Use geoid grid as altimetric reference.
      --use_sec_disp        Use the points cloud generated from the secondary
                            disparity map.
      --snap_to_left_image  This mode can be used if all pairs share the same left
                            image. In that case it will modify lines of sights of
                            secondary images so that they all cross those of the
                            reference image.
      --align_with_lowres_dem
                            If this mode is used, during triangulation, points
                            will be corrected using the estimated correction from
                            the prepare step in order to align 3D points with the
                            low resolution initial DEM.
      --disable_cloud_small_components_filter
                            This mode deactivates the points cloud filtering of
                            small components.
      --disable_cloud_statistical_outliers_filter
                            This mode deactivates the points cloud filtering of
                            statistical outliers.

Input json file
---------------

This program takes as input a json file (or a list of N json files in the case of a N images pairs processing) which corresponds to the content.json file generated at the prepare step (cf. above). Its output is the path to the folder which will contain the results of the stereo, that is to say the ``dsm.tif`` (regular grid of altitudes) and the ``clr.tif`` (corresponding color) files.

Input parameters
----------------

Some optional parameters enable to modify the regular grid:

* ``sigma``: controls the influence radius of each point of the cloud during the rasterization
* ``dsm_radius``: number of pixel rings to take into account in order to define the altitude of the current pixel
* ``resolution``: altitude grid step (dsm)
* ``epsg``: epsg code used for the cloud projection. If not set by the user, the more appropriate UTM zone will be retrieved automatically
* ``roi``: final dsm ROI. If the ROI is given by a float quadruplet, they have to already be in the final geometry. If the ROI is given by a vector file, or an image, the conversion to the final geometry will be performed automatically.

    * example with a quadruplet: ``cars_cli.py compute_dsm content.json outdir/ --roi 0.1 0.2 0.3 0.4``
* ``dsm_no_data``: no data value of the final dsm
* ``color_no_data``: no data value of the final color ortho-image
* ``corr``: correlator to use ('pandora' (version V1.B))
* ``corr_config``: correlator's configuration file (for pandora)
* ``min_elevation_offset``: minimum offset in meter to use for the correlation. This parameter is converted in minimum of disparity using the disp_to_alt_ratio computed in the prepare step.
* ``max_elevation_offset``: maximum offset in meter to use for the correlation. This parameter is converted in maximum of disparity using the disp_to_alt_ratio computed in the prepare step.
* ``use_geoid_as_alt_ref``: controls the altimetric reference used to compute altitudes. If activated, the function uses the geoid file defined by the ```OTB_GEOID_FILE``` environment variable.
* ``use_sec_disp`` : enables to use the secondary disparity map to densify the 3D points cloud.
* ``snap_to_left_image`` : each 3D point is snapped to line of sight from left reference image (instead of using mid-point). This increases the coherence between several pairs if left image is the same image for all pairs.
* ``align_with_lowres_dem``: During prepare step, a cubic splines correction is computed so as to align DSM from a pair with the initial low resolution DEM. If this mode is used, the correction estimated for each pair is applied. This will increases coherency between pairs and with the initial low resolution DEM.
* ``disable_cloud_small_components_filter``: Deactivate the filtering of small 3D points groups. The filtered groups are composed of less than 50 points, the distance between two "linked" points is less than 3.
* ``disable_cloud_statistical_outliers_filter``: Deactivate the statistical filtering of the 3D points. For this filter the examined statistic is the mean distance of each point to its 50 nearest neighbors. The filtered points have a mean distance superior than this statistic's mean + 5 * this statistic's standard deviation.

DASK parameters
---------------
As the prepare part, During its execution, this program creates a distributed dask cluster (except if the ``mode`` option is different than ``pbs_dask`` or ``local_dask``). In the logs, an internet address is displayed. It can be opened with firefox and displays a dashboard which enables to follow the tasks execution in real time.
The following parameters can be used :
* ``mode``: parallelisation mode (``pbs_dask``, ``local_dask`` or ``mp`` for multiprocessing)
* ``nb_workers``: number of nodes to use for the computation
* ``walltime``: nodes' allocation time

To know the number of used cores, the program rests on the ``OMP_NUM_THREADS`` environment variable.
In intern, the tile size is estimated from the value of the ``OTB_MAX_RAM_HINT`` variable (expressed in MB) times the memory amount reserved for a node, it is to say ``OMP_NUM_THREADS x 5 Gb``.
For a production at full image scale (or using several images), it is recommended that ``OTB_MAX_RAM_HINT`` is set to a value high enough to fill the allocated resources. For example, for ``OMP_NUM_THREADS=8``, the allocated memory for a node is set to 20Gb, thus the ``OTB_MAX_RAM_HINT`` can be set to 10 000.
A low value of ``OTB_MAX_RAM_HINT`` leads to a higher number of generated tiles and an under-consumption of the allocated resources.

Other environment variables can impact the dask execution on the cluster:

* ``CARS_NB_WORKERS_PER_PBS_JOB``: defines the number of workers that are started for each PBS job (set to 2 by default)
* ``CARS_PBS_QUEUE``: enables to turn to another queue than the standard one (dev for example)
* ``OPJ_NUM_THREADS``, ``NUMBA_NUM_THREADS`` and ``GDAL_NUM_THREADS`` are exported on each job (all set by default to the same value as ``OMP_NUM_THREADS``, it is to say 4)

The nodes on which the computations are performed should be able to handle the opening of several files at once. In the other case, some "Too many open files" errors can happen. It is then recommended to launch the command again on nodes which have a higher opened files limit.

Output contents
---------------

The output folder contains a content.json file, the computed dsm and the color ortho-image (if the ``color1`` field is not set in the input configuration file then the ``img1`` is used).

.. code-block:: bash

    $ ls
    clr.tif  content.json  dask_log  dsm.tif

If the ``--output_stats`` is activated, the output directory will contain tiff images corresponding to different statistics computed during the rasterization.

.. code-block:: bash

    $ ls
    clr.tif  content.json  dask_log  dsm_mean.tif  dsm_n_pts.tif  dsm_pts_in_cell.tif  dsm_std.tif  dsm.tif

Those statistics are:

* The number of 3D points used to compute each cell (``dsm_n_pts.tif``)
* The elevations' mean of the 3D points used to compute each cell (``dsm_mean.tif``)
* The elevations' standard deviation of the 3D points used to compute each cell (``dsm_std.tif``)
* The number of 3D points strictly contained in each cell (``dsm_pts_in_cell.tif``)


Once the computation is done, the output folder also contains a ``content.json`` file describing the folder's content and reminding the complete history of the production.

.. code-block:: json

    {
      "input_configurations": [
        {
          "input": {
            "img1": "/tmp/cars/tests/data/input/phr_ventoux/left_image.tif",
            "img2": "/tmp/cars/tests/data/input/phr_ventoux/right_image.tif",
            "srtm_dir": "/tmp/cars/tests/data/input/phr_ventoux/srtm",
            "nodata1": 0,
            "nodata2": 0
          },
          "preprocessing": {
            "version": "master//xxx",
            "parameters": {
              "epi_step": 30,
              "disparity_margin": 0.02,
              "epipolar_error_upper_bound": 10.0,
              "epipolar_error_maximum_bias": 0.0,
              "elevation_delta_lower_bound": -1000.0,
              "elevation_delta_upper_bound": 1000.0
            },
            "static_parameters": {
              "sift": {
                "matching_threshold": 0.6,
                "n_octave": 8,
                "n_scale_per_octave": 3,
                "dog_threshold": 20.0,
                "edge_threshold": 5.0,
                "magnification": 2.0,
                "back_matching": true
              },
              "low_res_dsm": {
                "low_res_dsm_resolution_in_degree": 0.000277777777778,
                "lowres_dsm_min_sizex": 100,
                "lowres_dsm_min_sizey": 100,
                "low_res_dsm_ext": 3,
                "low_res_dsm_order": 3
              }
            },
            "output": {
              "left_envelope": "/tmp/left_envelope.shp",
              "right_envelope": "/tmp/right_envelope.shp",
              "envelopes_intersection": "/tmp/envelopes_intersection.gpkg",
              "envelopes_intersection_bounding_box": [
                -58.589517087035645,
                -34.4931726206081,
                -58.58173610178845,
                -34.48677006524553
              ],
              "epipolar_size_x": 2407,
              "epipolar_size_y": 2510,
              "epipolar_origin_x": 0.0,
              "epipolar_origin_y": 0.0,
              "epipolar_spacing_x": 30.0,
              "epipolar_spacing_y": 30.0,
              "disp_to_alt_ratio": 2.5305049217664437,
              "raw_matches": "/tmp/raw_matches.npy",
              "left_epipolar_grid": "/tmp/left_epipolar_grid.tif",
              "right_epipolar_grid": "/tmp/right_epipolar_grid.tif",
              "right_epipolar_uncorrected_grid": "/tmp/right_epipolar_grid_uncorrected.tif",
              "minimum_disparity": -8.873300104758348,
              "maximum_disparity": 2.2324556746626323,
              "matches": "/tmp/matches.npy",
              "lowres_dsm": "/tmp/lowres_dsm_from_matches.nc",
              "lowres_initial_dem": "/tmp/lowres_initial_dem.nc",
              "lowres_elevation_difference": "/tmp/lowres_elevation_diff.nc"
            }
          }
        }
      ],
      "stereo": {
        "version": "master//xxx",
        "parameters": {
          "resolution": 0.30000001192092896,
          "sigma": null,
          "dsm_radius": 1,
          "epsg": 32721
        },
        "static_parameters": {
          "rasterization": {
            "grid_points_division_factor": null
          },
          "cloud_filtering": {
            "small_components": {
              "on_ground_margin": 10,
              "connection_distance": 3.0,
              "nb_points_threshold": 50,
              "clusters_distance_threshold": null,
              "removed_elt_mask": false,
              "mask_value": 255
            }
          }
        },
        "output": {
          "altimetric_reference": "ellipsoid",
          "epsg": 32721,
          "dsm": "dsm.tif",
          "dsm_no_data": -32768.0,
          "color_no_data": 0.0,
          "color": "clr.tif"
        }
      }
    }

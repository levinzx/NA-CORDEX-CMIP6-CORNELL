# **Aims:**
Use MPI to drive WRF for a decadal simulation over the NA-CORDEX (12km res) and CONUS (4km res) regions. The simulations should:
- Represent the time varying or different land use land categories (LULC) and aerosol (AOD) between historical and future climate
- The outer domain covers the NA-CORDEX required area, and the inner domain covers the current US east and west coastal wind farm leasing areas
        + ![model_domains](https://github.com/levinzx/NA-CORDEX-CMIP6-CORNELL/blob/main/model_domain.png)
- Input data is the MPI-LR ensemble member r1i1p1f1, which has the most comprehensive list of fields available to drive the WRF model.
- Due to the limited computational resources, only a limited years of simulation over the CONUS domain is planned. As a result, we want to select years in HIST and FUTURE that have a similar magnitude of climate variabilities, thus the comparison between HIST and FUTURE is less impacted by the climate variability.
- The simulation uses WARM restarts for the NA-CORDEX domain for the entire period, then uses ndown.exe to simulate selected years for the CONUS domain with at least a 3-month spin-up period before January 01 to achieve reasonable snow and soil fields. Both domains use spectrum nudging above the PBL to avoid drifting in the WARM restart runs from the input GCM.
- The simulations will enable both general atmospheric research and various specific applications in a changing climate, including solar energy, wind energy, windstorms, deep convection, freezing rain, land surface effect, heatwave & drought, etc.

# **Overview of the Simulations:**
The WRF simulations were done on the NCAR's Derecho HPC. The simulations were submitted as yearly chunks. For each year of simulation, 12 jobs responsible for 12 individual monthly simulations were submitted with the chronological dependency enabled.

## **Preparation for the simulation:**
- Initial boundary conditions prepared for each year of simulation,
- [namelist.input](https://github.com/levinzx/NA-CORDEX-CMIP6-CORNELL/blob/main/namelist.input.1960-01-01_00) specified for each month,
- [pbs job script](https://github.com/levinzx/NA-CORDEX-CMIP6-CORNELL/blob/main/runwrf.pbs.1960-01) prepared for each month,
- Then the jobs for one one-year simulation are submitted using a [script](https://github.com/levinzx/NA-CORDEX-CMIP6-CORNELL/blob/main/script.runwrf.template).

## **Example of the namelist and scripts:**
- [namelist.wps](https://github.com/levinzx/NA-CORDEX-CMIP6-CORNELL/blob/main/namelist.wps)
- [namelist.input.1960-01-01_00](https://github.com/levinzx/NA-CORDEX-CMIP6-CORNELL/blob/main/namelist.input.1960-01-01_00)
- [runwrf.pbs.1960-01](https://github.com/levinzx/NA-CORDEX-CMIP6-CORNELL/blob/main/runwrf.pbs.1960-01)
- [script.runwrf.template](https://github.com/levinzx/NA-CORDEX-CMIP6-CORNELL/blob/main/script.runwrf.template)

# **Procedures:**
## **WRF Compilation**
- Model version: WRF V4.6.1
- Modify the Registry to make sure all needed outputs are written to the “rh” streams
    + ZNT
    + RMOL
    + SWDDIR
    + SWDDIRC
    + SWDDNI
    + SWDDNIC
    + SWDOWN
    + SWDOWNC
    + SWDDIF
- WRF does not have buckets for frozen hydrometers, which causes round-off errors in long-term simulations. Modify the code to add “graup_acc_nc” and “hail_acc_nc” to the “prec_acc_dt” option. And add the buckets for frozen hydrometer types: “i_snownc”, “i_graupelnc”, “i_hailnc”
    + Files involved:
        * phys/module_diag_misc.F
        * phys/module_diagnostics_driver.F
        * Registry/Registry.EM_COMMOM
- Note, ACSHFLSM, ACLHFLSM, and other Noahmp accumulated values are done over the period of noahmp_acc_dt = 60, and no buckets are needed.
- About long-term simulation: WRF uses single precision for most variables, including XTIME. The round-off error would occur after about 32 years in precipitation buckets, and the buckets may be tipping off too soon, resulting in negative values. The round-off error in XTIME also affects the calculation of the nudging coefficient, and a problem would occur around 23.5 years of simulation. See [Corrections for tipping bucket and nudging in very long simulations](https://github.com/wrf-model/WRF/pull/2063)
- Compile the WRF using PNETCDF and enable the feature in the namelist.input to speed up the simulation.

## **WPS and input preparation:**
1)	Download the MPI fields from various sources (various data formats) to gather the list of mandatory fields needed to run WRF.
2)	Since most of the MPI data are in NetCDF format, and it is a real hassle to convert NetCDF to GRIB then create a Vtable for MPI variables (especially the soil properties at different levels), we convert MPI data directly to WPS intermediate format instead of using ungrib.exe.
    - Note that the default MPI fillvalue is positive and does not work with the default metgrid.exe interpolation setting. One needs to replace the fillvalue to a negative one, e.g. -1E30, in the WPS intermediate files.
3)	Since the SST extrapolated over the lakes has large biases, the temporally averaged land surface skin temperature is used to approximate lake temperature.
    - Instead of using WPS util to generate TAVGSFC, we manually calculate and add TAVGSFC to the intermediate WPS format files
    - See the description of TAVGSFC and how WRF will work with this property in the WRF User’s Guide.
    - TAVGSFC is calculated as follows,
        - a.	A 30-day average ahead of the given time is considered to accommodate the seasonal lag of lake surface temperature to around land surface temperature.
        - b.	Modify the produced TAVGSFC to 273.15K whenever TAVGSFC < 273.15 to avoid unphysical water temperatures below the freezing point.
4)	Create a METGRID.TBL for MPI variables. Add the variables unique to the MPI dataset (soil properties at different levels) and specify their interpolation methods. Also, check (and modify if necessary) the interpolation methods for SEAICE and SST to make sure that the resulting SEAICE and SST in the met_em files have:
    - Reasonable values and spatial distribution (avoid strange spatial patterns like the coarse resolution features along the coastlines, avoid strange values outside the reasonable range, must have reasonable values over the lakes)
    - Make sure the interpolation generates the lake surface temperature, or the lake effect will not exist. And the resulting open lake temperature - lake ice contrast has to be reasonable, otherwise the lake effect will be messed up.
    - Well correspondence between SST and SEAICE over the ocean (e.g., use the same interpolation methods to produce fields over the ocean. The interpolation methods can be different over lakes, but preferably the same).
5)	For future simulations, the changes in each LULC category are added to the historical LULC fields since WRF and MPI have very different land surface categories.
    - MPI land categories are mapped to the WRF MODIS 21 land categories before the perturbations between historical and future climates are calculated.

## **WRF preparation:**
1)	Enables PNETCDF feature in the namelist.input
2)	Enables spectrum nudging above PBL
3)	Enables varying SST
4)	Enables runtime diagnostic variables (saves effort in post-processing)
5)	Specify what variables to write to the wrfout at runtime.
6)	The file’s metadata generated by ndown.exe has the map projection and land time messed up. This information must be corrected before running wrf.exe. See the discussion on [ndown P_TOP error](https://forum.mmm.ucar.edu/threads/ndown-p_top-error-please-help.17476/#post-47491)
7)	Make sure that “CAMtr_volume_mixing_ratio” is linked to the correct scenarios for the future simulations.


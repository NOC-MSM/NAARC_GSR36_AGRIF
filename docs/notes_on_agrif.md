## Notes for setting up agrif domain

Using instructions here:
https://github.com/JMMP-Group/PROMOTE-WP7/wiki/Building-the-PROMOTE%E2%80%90WP7-AGRIF-zoom

###1) Set the AGRIF_FixedGrids.in as: 

```
1
2944 3434 2713 3212 3 3 3
0
```

###2) Build the zoom with exactly the same bathymetry of the parent

Beginning with a set of files. The "file set" is located on ARCHER2 here:
/work/n01/n01/atb299/NAARC_AGRIF
(Best to start fresh each time).

Edit namelist_cfg to use the desired parent domain_cfg.nc.

Copy the namelist_ref.
```
cp namelist_ref 1_namelist_ref
```

To make the 1_namelist_cfg, execute :
```
python make_namelist.py namelist_cfg domain_cfg_zps.nc
```

Then edit &namdom in 1_namelist_cfg such that:
```
!-----------------------------------------------------------------------
&namdom        !   space and time domain (bathymetry, mesh, timestep)
!-----------------------------------------------------------------------
   ln_read_cfg = .false.
   nn_bathy = 3
                           ! = 1 read the bathymetry file
                           ! = 2 compute from external bathymetry
                           ! = 3 compute from parent (if "key_agrif")
   nn_interp = 1
   ln_lin_int  = .true.                        ! linear (T) or near neighb (F) interp (nn_bathy =3)
   cn_domcfg   =  ''
   cn_fcoord   =  ''             ! external coordinates file (jphgr_msh = 0)
   cn_topo = ''
   cn_bath = ''
   cn_lon = ''
   cn_lat = ''
   rn_scale = 1
   rn_bathy    =    0.     !  value of the bathymetry. if (=0) bottom flat at jpkm1
   nn_msh      =    0      !  create (=1) a mesh file or not (=0)
```

To build the domain, build_domains.slurm executes make_domain_cfg.exe so first link to a compiled make_domain_cfg.exe from NEMO TOOLS. This has been compiled with NEMO v5.0 (namelist) and key_agrif in cpp_DOMAINcfg.fcm. The compiler on ARCHER2 was arch-X86_ARCHER2-Cray-xios3.fcm.

```
ln -s /work/n01/n01/benbar/GSR36_AGRIF/NEMO_v5/nemo/tools/DOMAINcfg/make_domain_cfg.exe make_domain_cfg.exe

sbatch build_domains.slurm
```

It should be possible in 20 minutes to make and rebuild the domain_cfg or mesh_mask but not both and will quickly make a bunch of netcdf files that are later rebuilt. At the end this should make a 1_domain_cfg.nc and a 1_mesh_mask.nc. If it just makes the 1_layout.dat something isn't working. I've had problems with MES coords which are probably because MES won't work, only pure SCO or ZPS apparently.

Plot and check domain.

###3) Update the bathymetry to GEBCO_2021

On to step (3). Update bathy to gebco. 

Install pyogcm python environment but using pip for most parts but conda for proj related libraries which can have conflicts (because it takes ages to solve with just conda).
```
git clone https://github.com/oceandie/agrif_mo.git
cd agrif_mo
conda create -n pyogcm python=3.12 numpy pip
conda activate pyogcm
conda install cartopy matplotlib
pip install xarray netCDF4
```

Use the bathy_regrid_horiz.py in the "file set" or edit ./src/domain/bathymetry/bathy_regrid_horiz.py to insert below at line 119:
```
grid_data = grid_data.rename_dims({'time_counter': 't', 'nav_lev': 'z'})
```

Set environment variables to point to files and run bathy_regrid_horiz.py:
```
batinp="GEBCO_2021_sub_ice_topo.nc"
tmask="1_mesh_mask.nc"
batout="1_bathy_meter_3m.nc"

python bathy_regrid_horiz.py -B ${batinp} -S gebco -M ${tmask} -m -d3.0 -F -o ${batout}
```

The CDFTOOLS routines generally expect the _FillValue attribute to be set to zero:

```
inp="1_bathy_meter_3m.nc"
out="1_bathy_meter_3m_FillZero.nc"
ncatted -a _FillValue,Bathymetry,m,d,0.0 ${inp} ${out}
```

Follow the rest of step (3) or
Link the executable:
```
ln -s /work/n01/n01/atb299/CDFTOOLS_4.0_ISF/bin/cdfsmooth cdfsmooth

sbatch run_smooth.sh
```

Follow step 4 to re-generate the 1_domain_cfg.nc with new bathymetry.

Generate forcing weights.
```
source /work/n01/n01/benbar/agrif/bin/activate

python create_weights.py
```

Generate initial conditions for tests, constant and gradient.

## For compiling

Add key_agrif CPP file.
Add NST in work_cfgs.txt in setup_script.

## For setting up run directory.

Forcing weights need to be in forcing folder with 1_ at start of file name.
In 1_namelist forcing weights do not need 1_.

Inputs folder needs approproate 1_ files for the required inputs like 1_bfr_cdmin_2d.nc and 1_domain_cfg.nc. Also inlcude the AGRIF_FixedGrids.in.

AGRIF_FixedGrids.in 1_bfr_cdmin_2d.nc and 1_domain_cfg.nc specifically need linking to the run directory.

Run directory needs 1_namelist_cfg, 1_namelist_ref, 1_namelist_ice_cfg, 1_namelist_ice_ref, 1_context_nemo.xml.
1_context_nemo.xml can be a copy of the parent. iodef.xml needs to edited for the extra context by appending this line:
<context id="1_nemo" src="./1_context_nemo.xml"/>

The 1_namelist_cfg need editing in accordance with the 1_ input files.


## Notes for setting up agrif domain

Using instructions here:
https://github.com/JMMP-Group/PROMOTE-WP7/wiki/Building-the-PROMOTE%E2%80%90WP7-AGRIF-zoom

The "file set" is located on ARCHER2 here:
/work/n01/n01/atb299/NAARC_AGRIF

Follow step (2) to make and edit the 1_namelist_cfg. Copy the namelist_ref.
```
cp namelist_ref 1_namelist_ref
```

build_domains.slurm executes make_domain_cfg.exe so first link to a compiled make_domain_cfg.exe from NEMO TOOLS. This has been compiled with NEMO v5.0 (namelist) and key_agrif in cpp_DOMAINcfg.fcm. The compiler on ARCHER2 was arch-X86_ARCHER2-Cray-xios3.fcm.
```
ln -s /work/n01/n01/benbar/GSR36_AGRIF/NEMO_v5/nemo/tools/DOMAINcfg/make_domain_cfg.exe make_domain_cfg.exe

sbatch build_domains.slurm
```

This should make a 1_domain_cfg.nc and a 1_mesh_mask.nc.

Plot and check domain.

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
source /work/n01/n01/benbar/agrif/bin/activate
python create_weights.py


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


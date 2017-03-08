# CPM tutorial

I'll show you how we can extract CPM photometry of K2 Campaign 9 data. 
I'm marking in __bold__ the parts that require immediate attention. 

### CPM_PART1

The event I'm interested in is OGLE-2016-BLG-0241 (ob160241 for short). The 
event coordinates are 18:02:31.76 -27:31:46.6 (or 270.6323333 -27.5296111 
if your mind works in degrees). In the ground-based data it 
peaked at HJD'=7491.4 or on April 13th, 2016. This is before the first part of 
K2 Campaign 9 started, hence we're interested in campaign "91" (as opposed to 
"92"). On some plot I see that this coordinates correspond to channel 49 
(this can't work that way __this needs to be fixed__). Now it's time to find 
which TPF and which pixel is closest to the event -- this is only rough estimate 
that is based on WCS information in TPF files. We start python and do:

```python
from K2CPM import wcsfromtpf
import K2CPM
wcsfromtpf.dir_here = K2CPM.__file__[:K2CPM.__file__.rfind('__init__.py')] + "../../"

ra = 270.6323333
dec = -27.5296111
campaign = 91
channel = 49

wcs = wcsfromtpf.WcsFromTpf(channel, campaign)
nearest_pixel = wcs.get_nearest_pixel_radec(ra, dec)
(pix_x, pix_y, _, _, epic_id, distance) = nearest_pixel

print("pix_x = {:}\npix_y = {:}\nepic_id = {:}\ndistance = {:.1f} arcsec".format(pix_x, pix_y, epic_id, distance.value))
```
```
pix_x = 119
pix_y = 1022
epic_id = 200070511
distance = 1.8 arcsec
```

The second line above is a way of hacking some fixed directory path 
(__this also needs to be fixed__). Opps, I'm not sure what x and y mean here. 
Kepler and K2 documents don't use (x,y) they use (column,row) or (row,column). 
Here x runs from 13 to 1112, and y runs from 21 to 1044 
(__the format of coordinates has to be strictly specified__). Fortunately, the 
distance given above is < 4 arcsec, so the coordiantes are from the right 
channel. 

We know which pixel we're interested in so now its time to prepare predictor matix:

```python
from K2CPM.cpm_part1 import run_cpm_part1
from K2CPM import matrix_xy
import numpy as np

n_predictor = 400
n_pca = 0
distance = 16
exclusion = 1
flux_lim = (0.1, 2.0)
tpf_dir = 'tpf/'
pixel_list = np.array([[pix_y, pix_x]])

(predictor_matrix_list, predictor_epoch_masks) = run_cpm_part1(
		int(epic_id), campaign, n_predictor, n_pca, distance, exclusion, 
        flux_lim, tpf_dir, pixel_list, return_predictor_epoch_masks=True)

stem = "{:}_{:}_{:}_{:}".format(campaign, channel, pix_y, pix_x)
np.savetxt(stem+"_predictor_epoch_mask.dat", predictor_epoch_masks[0], fmt='%r')
matrix_xy.save_matrix_xy(predictor_matrix_list[0], stem+"_pre_matrix_xy.dat")
```

We asked for 400 pixels in predictor matrix, no use of Principal Component Analysis 
(n_pca=0), and some standard exclusion limits. The tpf_dir is where we keep 
downloaded TPF files. If you already have some of those files, then just make 
symbolic link. Note that ```pixel_list = np.array([[pix_y, pix_x]])``` reverts 
the order of coordiantes. 

At this point we have two files ready, we need two more:

```python
from K2CPM import tpfdata

tpfdata.TpfData.directory = tpf_dir

tpf = tpfdata.TpfData(epic_id=epic_id, campaign=campaign)

tpf.save_pixel_curve_with_err(pixel_list[0][0], pixel_list[0][1], file_name=stem+"_pixel_flux.dat")

np.savetxt(stem+"_epoch_mask.dat", tpf.epoch_mask, fmt='%r')
```

At this point you should have 4 files saved:
```
91_49_1022_119_epoch_mask.dat
91_49_1022_119_pixel_flux.dat
91_49_1022_119_predictor_epoch_mask.dat
91_49_1022_119_pre_matrix_xy.dat
```
In most cases *_epoch_mask.dat and *_predictor_epoch_mask.dat will be the same, 
but this is not guaranteed. The largest file is *_predictor_epoch_mask.dat and 
its size depends on n_predictor.

You may exit python shell at this point.

### CPM_PART2

We can proceed to second part of CPM. Open python shell once more and import 
required modules, and load the files:

```python
import numpy as np
from K2CPM import cpm_part2, matrix_xy

stem = "91_49_1022_119"

predictor_matrix = matrix_xy.load_matrix_xy(stem+"_pre_matrix_xy.dat")
epoch_mask = cpm_part2.read_true_false_file(stem+"_epoch_mask.dat")
predictor_epoch_mask = cpm_part2.read_true_false_file(stem+"_predictor_epoch_mask.dat")
(tpf_time, tpf_flux, tpf_flux_err) = np.loadtxt(stem+"_pixel_flux.dat", unpack=True)
```

Next step is just running the cpm_part2:

```python
l2 = 1000.
res = cpm_part2.cpm_part2(tpf_time, tpf_flux, tpf_flux_err, epoch_mask, predictor_matrix, l2)
```

Now I see that cpm_part2 in fact does not use predictor_epoch_mask - 
__this should be corrected__. BTW, this is thanks to a recent change. 

(C) Radek Poleski March 7th, 2017
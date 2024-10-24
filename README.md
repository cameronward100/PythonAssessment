# ROSAT-data-filtering
Applying filters to the Second ROSAT All-Sky Survey Point Source Catalog using Python and TOPCAT, in order to generate a list of potential candidates for X-ray Dim Isolated Neutron Stars (XDINs) in the ROSAT data

1) Downloading the ROSAT catalog

Access the ROSAT catalog using the following link: https://heasarc.gsfc.nasa.gov/W3Browse/rosat/rass2rxs.html.

You can choose which columns to download by clicking on the 'browse this table...' button in the top left corner of the page. Here, keep all the default selected columns, but also tick the following columns too: hardness_ratio_1_error, hardness_ratio_2_error, bb_flux, x_pixel, x_pixel_error, y_pixel, y_pixel_error.

Scroll down to the bottom of the page to where it says 'Do you want to change your current query settings?', and change the 'Limit Results to' section to 'No Limit', and the 'Ouput Format' to be a FITS file.

Click Start Search and download the data file which should contain 135118 objects.

2) Applying initial filters in Python

Open a new Python document, and write the following code:

    from astropy.io import fits
    from astropy.table import Table
    import numpy as np

Open and access the FITS file downloaded previously, you can do this as follows:

    fits_file_path = "insert the file directory here, plus a filename here"      
    #skip this step if you have the file in the same directory as the Jupiter Notebook
    
    initial_data = fits.open(fits_file_path)
    data = initial_data[1].data

2.1) Remove anything with a Hardness Ratio greater than 0. 

This step is taken due to the fact that only 'soft' x-ray emitters are wanted since this is a property of an X-ray Dim Isolated Neutron Star. Do this for both the hardness_ratio_1 and hardness_ratio_2 columns. After these steps there should be 76077 objects remaining. You can do this as follows by selecting these columns from the imported data:

    column1 = 'HARDNESS_RATIO_1'
    column2 = 'HARDNESS_RATIO_1_ERROR'
    max_value = 0
    filtered_data = data[(data[column1] - data[column2]) <= max_value]

    column1 = 'HARDNESS_RATIO_2'
    column2 = 'HARDNESS_RATIO_2_ERROR'
    max_value = 0
    filtered_data = data[(data[column1] - data[column2]) <= max_value]

2.2) Remove anything with a positional uncertainty greater than 15 arcseconds.

An uncertainty of 15 arcseconds is chosen due to the fact that the 7 known XDINS all have low positional uncertainties, so the assumption is made that this is the general case. Firstly, convert the X and Y pixel uncertainties by 45 in order to convert from pixels to arcseconds, then combine them into a positional uncertainty by adding these X and Y uncertainties in quadrature. Additionally, save this new positional uncertainty as a new column of data in the FITS file. After this step there should be 28748 objects remaining. You can do this step as follows:

    column1 = 'X_PIXEL_ERROR'
    column2 = 'Y_PIXEL_ERROR'
    max_difference = 15

    converted_column1 = filtered_data[column1]*45
    converted_column2 = filtered_data[column2]*45
    position_uncertainty = np.sqrt((converted_column1**2)+(converted_column2**2))

    filtered_data = Table(filtered_data)
    filtered_data['POSITIONAL_UNCERTAINTY'] = position_uncertainty

    filtered_data = filtered_data[(position_uncertainty) <= max_difference]

2.3) Remove anything with a log x-ray to optical flux ratio less than 2.5. 

This step is taken due to the fact that of the known XDINS (Magnificent 7), the lowest log optical-to-x-ray flux ratio is 2.63, so an appropriate cutoff value of 2.5 is chosen. When performing this step, a minute correction within the log(x-ray flux) is required so not to obtain an error code within Python due to the fact that several of the x-ray flux inputs are zero - this correction is small enough to have no impact on the final output data. The visual magnitude to be used is the visual magnitude of the SDSS all-sky survey. After this step there should be 9358 objects remaining in the data. You can do this step as follows:

    column = 'BB_FLUX'
    max_difference = 2.5
    Visual_mag = 22.5
    
    flux_ratio = np.log10(filtered_data[column]+(1*(10**-100)))+(Visual_mag/2.5)+5.37

    filtered_data = filtered_data[(flux_ratio) >= max_difference]

This filtered data set can now be exported for further analysis in TOPCAT. You can export it as a new FITS file as follows:

    output_file = "Insert new file directory here, including the file name"      #on Windows, use double slashes to avoid getting an error
    fits.writeto(output_file, np.asarray(filtered_data), overwrite=True)

3) Applying filters in TOPCAT - removing known objects from other all-sky surveys.

Download TOPCAT by using the following link: https://www.star.bris.ac.uk/~mbt/topcat/. From here, scroll down to the downloads section and select either the 'topcat-full.jar' Standard Version under the Standalone Jar File subheading if you have Windows or Linux, or if you have MacOS select 'topcat-all.dmg' under the MacOS subheading. If you run into any trouble when attempting to access TOPCAT, check that you have the newest version of JAVA installed and that there are not multiple versions installed either, as this can cause issues with the running of TOPCAT.

Once installed, download the outputted Python FITS file onto TOPCAT by selecting 'Open new table' in the top left corner, then 'System browser', and select the FITS file.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/97892d8c-ff75-41de-ac9c-ff99569c64fb)

3.1) Remove any object not within 3 arcminutes of an SDSS DR16 source. 

Select 'Sky crossmatch' (the blue cross on the top toolbar). Select SDSS DR16 as the remote table, and the imported FITS file as the local table, with 'RA' and 'DEC' selected in units of degrees. In the 'Match Parameters' section, set a radius of 3 arcminutes, with 'Best' selected as the mode. All other variables leave as default. Select 'Go' and TOPCAT will filter out data within these established parameters. A new table will be generated on the TOPCAT page on the left-hand side containing 4022 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/206f7d41-6731-408f-988f-8d7e72daf2bf)

3.2) Remove anything within 2x its positional uncertainty of an SDSS DR16 source. 

Select the matching rows button (two red matches on the top toolbar). In the 'Match criteria' section, select the 'Sky with errors' algorithm and set the scale to 10 arcminutes. Select the table produced in part 3.1) in both the 'Table 1' and 'Table 2' sections. In Table 1, select the RA and DEC columns with units of degrees, and the positional uncertainty column with units of arcseconds. In Table 2, select the RA_ICRS and DE_ICRS columns with units of degrees, and the positional uncertainty column with units of arcseconds. In the 'Output Rows' section, change the join type to be '1 not 2', then click 'Go' at the bottom to generate a new table containing 172 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/1cf02ea3-5779-4d11-a411-3e15f647145f)

3.3) Remove anything within 2x its positional uncertainty of a GAIA EDR3 source. 

Repeat 3.1), instead using GAIA EDR3 as the remote table and the table generated in 3.2) as the local table. Repeat 3.2), instead setting table 1 and 2 to be the GAIA EDR3 crossmatch table just generated, with table 1 using the RA and DEC columns in degrees, and the positional uncertainty column in arcseconds, and table 2 using the ra_epoch2000 and dec_epoch2000 columns in degrees, and the positional uncertainty column in arcseconds. Generate the new table containing 89 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/9f9a251d-f00f-456c-8a2e-a0411512f5c9)
![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/e0834a3d-9c3e-4b73-bf8a-58b35bdea247)

3.4) Remove anything within 5 arcseconds of a PANSTARRS DR1 source.

Repeat 3.1), instead using PANSTARRS DR1 as the remote table and the table generated in 3.3) as the local table. Using the matching rows button in the toolbar, select the 'Sky' algorithm with a maximum error of 5 arcseconds. Set table 1 and 2 to be the crossmatch table just generated, with table 1 using the RA and DEC columns, and table 2 using the RAJ2000 and DEJ2000 columns. Set the join type to '1 not 2' and generate the new table containing 72 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/21cd333b-d301-4697-b05b-b2d763f0f20d)
![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/886cce81-71cc-400c-90be-853825193740)

3.5) Accessing the TYCHO-2 survey.

Select the VO drop-down menu at the top of the TOPCAT page, then click the 'VizieR Catalogue Service'. In 'Row Selection', ensure that 'All Rows' is selected, and that the maximum row count in unlimited. In 'Catalogue Selection', search by keyword for TYCHO-2, and select the one with name I/259 'The TYCHO-2 Catalogue'. Then click 'OK' at the bottom of the page. Be aware that this is a large catalogue and will take several minutes to upload onto TOPCAT.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/0afeb747-733f-448f-9126-afc81149be73)
![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/84b481c7-b91c-4a48-9f42-823d5ed76bc3)

3.6) Remove anything within 10 arcseconds of a TYCHO-2 source.

In matching rows, select the 'Sky' algorithm with a maximum error of 10 arcseconds. Set table 1 to be the table generated in 3.4), using the RA and DEC columns, and set table 2 to be the 'I_259_tyc2' table imported from VizieR, using the _RAJ2000 and _DEJ2000 columns. Set the join type to be '1 not 2', and generate the new table containing 67 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/8b2444f3-2f42-4c3a-bd20-f8b6d0048ae8)

3.7) Accessing the Veron-Cetty survey.

Repeat 3.5), instead searching by keyword for Veron-Cetty, selecting the one with the name VII/258 'Quasars and Active Galactice Nuclei (13th Ed.)(Veron+ 2010)'. This may take several minutes to upload onto TOPCAT.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/b20b9af0-a4fc-4184-9d34-530903d6352d)

3.8) Removing anything within 10 arcseconds of a Veron-Cetty source.

Repeat 3.6), instead selecting table 1 to be the one generated in 3.6), using the RA and DEC columns, and selecting table 2 to be the 'VII_258_vv10' table imported from VizieR, using the _RAJ2000 and _DEJ2000 columns. Generate the new table cotaining 57 objects. 

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/842914a8-c07e-4d4f-a03a-b801d5b1327b)

3.9) Removes any object within 10 arcseconds of a TYCHO-1 and Hipparcos source that is not included in the TYCHO-2 catalog

Repeat 3.6), instead selecting Table 1 as the one generated in 3.8), using the RA and DEC columns, and selecting Table 2 to be the 'l_259_suppl_1' table from VizieR when importing TYCHO-2, using the _RAJ2000 and _DEJ2000 columns. Generate the new table containing 56 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/09e815fd-764f-4f88-9d77-4de0919c713d)

Finally, check that at least 3 of the Magnificent 7 known XDINS within the SDSS all-sky survey remain in the generated list of 56 objects by comparing their RA and DEC. This can be done manually downloading the file you obtained as a CSV file and opening it in Excel. You can find the RA and DEC of the Magnificent 7 online, but this paper is a good place to start: https://arxiv.org/pdf/2407.00275.


4) Removing saturated images

In this list of 56 objects, several images appear saturated on the SDSS image viewer. These objects need to be removed as they are not distinguishable from the object saturating them and are thus not viable candidates.

4.1) Getting magnitude columns for each object

Perform a crossmatch of the 56 candidate XDINS with TYCHO-2, setting a radius of 0.6 arcminutes with a 'Find Mode' of 'Each'. Select Go, and the objects in the list within 0.6 arcminutes of a TYCHO-2 source have the respective TYCHO-2 columns added on to them. In this new table, select the 'Display Column Metadata' symbol on the top display bar, and select only the coordinate and positional uncertainty data, the hardness ratios and x-ray to optical flux ratios, as well as any magnitude columns contained within the data set. Finally, save and export the new list as a CSV file.

4.2) Filling in the blank magnitude columns

Open the new CSV file in Excel. For each magnitude column in the data, sort from smallest to largest. You can scroll down to the bottom of the column; any rows without data in that column will be blank. In these blank columns, input '=1e99' and enter. Repeat this for all magnitude columns in the dataset so that there are no blank spaces in the magnitudes of the 56 candidates. Click on 'save as' and save this new data over the existing file, then open the CSV in TOPCAT and export again as a .fits file.

4.3) Removing the saturated images in Python

Open the FITS in Python. Select all the magnitude columns. The largest TYCHO-2 magnitude in the data is 11.802. It is assumed that an XDIN will appear at approximately the same magnitude in all optical magnitudes, therefore set the limiter at 11.802 and remove any object with a magnitude less than this in any of the available magnitude columns. This should reduce the list of candidates to 40 (37 excluding the M7). Export this list as a new .fits file.

5) Removing objects close to an SDSS source (again)

In the SDSS image viewer it can be seen that 5 of the candidate objects are in fact white dwarf stars rather than XDINS as they appear too close to 'blue pea'-like objects. In order to remove these objects, conduct a crossmatch with SDSS DR16 as before, but then rather doing a sky with errors search, do a sky search within 7 arcseconds. This limit is chosen due to the fact that any higher value removes some of the M7 from the list, which is not desired. Perfoming this search removes 6 objects; the 5 white dwarfs as well as another object which is apparently a faint galaxy. Export this final list of 31 candidate XDINS.

If you run into any issues, have any queries, or spot any mistakes, I am contactable on sac48@sussex.ac.uk.

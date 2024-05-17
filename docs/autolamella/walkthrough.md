# Walkthrough

The following page provides a detailed walkthrough for each method. As the methods contain significant overlap, only the unique parts will be described. 

Each walkthrough assumes you have already [configured your microscope](./../openfibsem/getting_started.md#configuring-your-microscope), loaded your sample and peformed initial setup (linked the stage, platinum deposition, etc.)


## On Grid Method

The following walkhrough is for the on-grid lamella method. The default protocol is protocol-on-grid.yaml.

## Setup Experiment

1. Connect to the Microscope (Connection Tab -> Select Configuration -> Connect to Microscope)
2. Create an Experiment (File Menu -> Create Experiment -> Select Directory -> Select Experiment Name)
3. Load a Protocol (File Menu -> Load Protocol -> protocol-on-grid.yaml). We recommend starting with this protocol, and editting it to suit your microscope and sample.
4. Move the stage to the sample grid. You can save this position for future use in the Movement Tab (Movement Tab -> Add Position -> Give your position a name (e.g. Grid 01) -> Export Position (save to file))
5. Move the stage to the milling angle. This is the tilt angle you want to mill your lamella at. You can use the Movement Tab to move to this tilt angle. 
6. Acquire Images. Use the Image Tab to acquire images with both beams. Images are required to display milling patterns.
7. Make you beams coincident / eucentric. Use the movement controls to align the coincidence of the beams. First double click on a feature in the electron beam to centre it. Then press Alt + double click in the ion beam align the same feature coincident. The same feature should now be centred in both beams.  

### Setup Positions

After you have setup your experiment, and microscope, you can now start selecting the lamella positions. 

1. Add a Lamella. Change to the Experiment tab, and Add Lamella. You can use the same movement controls to navigate around the sample to find your region of interest. It may also help to increase the field of view to get your bearings.  
2. Once you have found your region of interest, move the milling patterns with the milling controls. Press shift + left click to move the selected pattern, or cntrl + shift + left click to move all the patterns (keeping the same orientation). Patterns cannot be placed outside the image. You can adjust the dimensions and settings for the patterns in the Milling tab.  For best results, make sure your beams are coincident and the lamella and fiducial are placed along the horizontal centre of the image (aligned with the tilt axis).
3. Once you are happy with your pattern positions, press Save Position (in the Experiment tab). The button will switch from orange to green and indicate 'Position Ready'.
4. Repeat steps 1 - 5 until you have selected all your lamella positions.
5. When you are ready, press Run Setup AutoLamella.

### Setup AutoLamella

After pressing Setup AutoLamella:

1. The software will move to the first selected lamella position and if supervision is enabled ask you to confirm the the milling patterns. Press Continue to confirm, or use the milling controls to adjust.
2. Next, the alignment fiducial will be milling. The area around the fiducial will be used to align the lamella at each step. If supervision is enabled, ask you to confirm the milling operation. Press Run Milling to mill the fiducial, and then press Continue when happy with the fiducial.  
3. After the fiducial has been milled, an alignment image is acquired.
4. Steps 1 - 3 will be repeated for each selected lamella position.

### Run AutoLamella

After pressing Run AutoLamella

1. The software will move back to the first selected lamella position, and re-align using the alignment image.
2. The stress relief features (microexpansion, waffle notch) will be milled.  If supervision is enabled, ask you to confirm the milling operation. Press Run Milling to mill the pattern, and then press Continue when happy with the milling.
3. The rough trench milling patterns will be milled (all lamella trenches except the final polishing).  If supervision is enabled, ask you to confirm the milling operation. Press Run Milling to mill the pattern, and then press Continue when happy with the milling.  
4. Steps 1 - 4 will be repeated for each selected lamella position.
5. Once rough milling has been completed for each lamella, the software will go back to the first selected lamella position, realign the alignment image, and begin final polishing.  If supervision is enabled, ask you to confirm the milling operation. Press Run Milling to mill the pattern, and then press Continue when happy with the milling.  
6. Step 5 is repeated for all selected lamella positions.
7. Once final polishing is completed the software will return to the Experiment Tab, where you can see the status of all selected lamella. You are free to select more lamella and run again, but it is recommended to remove your sample after final polishing to prevent contamination build up. 

## Waffle Method

The following walkthrough is for the waffle method. The default protocol is protocol-waffle.yaml.

## Setup Experiment

1. Connect to the Microscope (Connection Tab -> Select Configuration -> Connect to Microscope)
2. Create an Experiment (File Menu -> Create Experiment -> Select Directory -> Select Experiment Name)
3. Load a Protocol (File Menu -> Load Protocol -> protocol-waffle.yaml). We recommend starting with this protocol, and editting it to suit your microscope and sample.
4. Move the stage to the sample grid. You can save this position for future use in the Movement Tab (Movement Tab -> Add Position -> Give your position a name (e.g. Grid 01) -> Export Position (save to file))
5. Move the stage flat to the ion beam. This is the rotation and tilt that makes the sample perpendicular to the ion beam. You can use the Movement Tab -> Move Flat to Ion Beam button to move to this position. If this does not move you to the current position, please check you have configured your microscope correctly.  
6. Acquire Images. Use the Image Tab to acquire images with both beams. Images are required to display milling patterns.
7. Make you beams coincident / eucentric. Use the movement controls to align the coincidence of the beams. First double click on a feature in the electron beam to centre it. Then press Alt + double click in the ion beam align the same feature coincident. The same feature should now be centred in both beams.  

### Setup Trench Positions

After you have setup your experiment, and microscope, you can now start selecting the trench positions. 

1. Add a Lamella. Change to the Experiment tab, and Add Lamella. You can use the same movement controls to navigate around the sample to find your region of interest. It may also help to increase the field of view to get your bearings.  
2. Once you have found your region of interest, move the milling patterns with the milling controls. Press shift + left click to move the selected pattern, or cntrl + shift + left click to move all the patterns (keeping the same orientation). Patterns cannot be placed outside the image. You can adjust the dimensions and settings for the patterns in the Milling tab. 
3. Once you are happy with your pattern positions, press Save Position (in the Experiment tab). The button will switch from orange to green and indicate 'Position Ready'.
4. Repeat steps 1 - 5 until you have selected all your lamella positions.
5. When you are ready, press Run Waffle Trench Milling AutoLamella.

##### Using the Minimap

As an alternative to the above, you can also use the minimap to acquire a tiled image flat to the ion beam and select positions from there. To access the minimap, use the Tools Menu -> Open Minimap Tool. 

1. Set the image beam type to Ion, and select your imaging settings. Recommended settings are Grid Size = 2000um, Tile Size = 500um, resolution 1024px, Dwell Time = 1us, Autocontrast, Autogamma. The save path and file name will be generated for you experiment automatically. 
2. Press Run Tile Collection. The software will acquire a tileset, and stitch it together. Currently overlap is not supported. 
3. Once the stitching is complete, you can add a new lamella position with Alt + Left Click. 
4. Change to the Position Tab and enable Display Pattern (and select trench) to see an overlay of the trench pattern in the image. 
5. To move an existing pattern, go to the Position Tab -> Select the Position and press Shift + Left Click on the image to move it. You can also move the stage to the position by pressing Move to Position Name or by Double Left Clicking on the image. 
6. Optionally, if you have correlation data (fluroescence images) you can load them in the Correlation Tab and correlate them with your overview image. 
7. Once you are happy with the positions go back to the AutoLamella UI. 
8. In the AutoLamella UI, go through each selected position and press move to position. Once you are happy with your pattern positions, press Save Position (in the Experiment tab). The button will switch from orange to green and indicate 'Position Ready'. IMPORTANT: make sure you go to the position before pressing Save Position (otherwise it will save the position as wherever the stage currently is, we are working on making this more streamlined). 
9. When you are ready, press Run Waffle Trench Milling to start milling trenches. 

### Trench Milling

After pressing Trench Milling:

1. The software will move to the first selected trench position, and restore the microscope state. 
2. The trench milling pattern (trench) will be milled. If supervision is enabled, ask you to confirm the milling operation. Press Run Milling to mill the pattern, and then press Continue when happy with the milling.
3. Charge neutralisation will be run, this is to disapate ion charge built up during trench milling. 
4. Reference images will be acquired of the final trench. 
5. Steps 1 - 4 will be repeated for each selected position.
6. Once all trenches have been milled, the software will return to the Experiment Tab and undercut milling will be enabled. Press Run Waffle Undercut Milling to start milling undercuts. 

### Undercut Milling

After pressing Undercut Milling.

1. The software will move the the first trench position, and restore the microscope state. 
2. The stage will rotate 180 degrees, and tilt flat to the electron beam to prepare for undercut milling. 
3. The software will use machine learning feature detection to align the lamella (trench) centre in the electron and ion beams. If supervision is enabled, it will ask you to confirm the feature detections. Drag the feature to move it to the correct location (if required), and then press Continue to proceed with the workflow. 
4. The stage will tilt down by the amount specified in the protocol protocol["options"]["undercut_tilt_angle"] (default is -5.0 deg). 
5. The software will detect the top of the lamella to be used to place the undercut pattern. If supervision is enabled, it will ask you to confirm the feature detections. Drag the feature to move it to the correct location (if required), and then press Continue to proceed with the workflow.  
6. The undercut milling pattern (undercut) will be milled. If supervision is enabled, ask you to confirm the milling operation. Press Run Milling to mill the pattern, and then press Continue when happy with the milling.
7. Steps 4 - 6 will repeat for the number of iterations specified in protocol["milling"]["undercut"]["stages]
8. The software will detect the centre of the lamella to re-align it to the centre of the image in both beams to restore coincidence after tilting. If supervision is enabled, it will ask you to confirm the feature detections. Drag the feature to move it to the correct location (if required), and then press Continue to proceed with the workflow. 
9. Reference images will be acquired of the final undercut. 
10. Steps 1 - 9 will be repeated for each selected position.
11. Once all undercuts have been milled, the software will return to the Experiment Tab and Setup AutoLamella will be enabled. Press Run Setup AutoLamella to setup the final lamella milling. 

### Setup and Run AutoLamella

From here, the workflow is exactly the same as the on-grid method. For more details please see [On-Grid Setup AutoLamella](#setup-autolamella)

The same as for the on-grid method, after the final polishing workflow completes the software will return the Experiment Tab and display the status of each lamella. It is recommended that you remove your sample after final polishing to prevent contamination build up. 

## Liftout Method

[Under Construction]

## Setup Protocol

Both liftout method require additional protocol setup. This step will walkthrough configuring the protocol for your system. 

### Scan Rotation

Liftout was developed with ion beam scan rotation set to 0 degrees. I've done my best to make it agnostic to the scan rotation (e.g. fliping the direction of movements), but can't promise i've considered everything. I know scan rotation = 180 deg is more common, but it's recommened to use 0 degrees for running liftout. I will update this note, when I have validated it works for 180 deg rotation too. If you use 180 degrees rotation, and encountered specific issues, please let me know at patrick@openfibsem.org.

### Named Positions

You will need to define two named positions, to specify which grid is for milling and which is for landing. Respectively these are called trench_start_position and landing_start_position in the protocol.  

To specify these named positions, you first need to add them to the positions.yaml file. To add the named positions.

1. Open OpenFIBSEM UI / AutoLamella UI / AutoLiftout UI, connect to the microscope.
2. Change to the Movement Tab.
3. Move the stage to the grid you want to use for milling. Go flat to the Ion beam.
4. Press Add Position. Rename the position, and press update. You can call it whatever you like, but recommended is something like liftout-grid-01-milling.
5. Move the stage to the grid you want to use for landing. Go to the landing angle.
6. Press Add Position. Rename the position, and press update. You can call it whatever you like, but recommended is something like liftout-grid-02-landing.
7. Press Export Positions and select the positions.yaml. Choose to overwrite and append to the file. 
8. Reload AutoLiftout UI, connect to the microscope, and create/load your experiment/protocol to see these named positions in the Protocol tab.
9. Set the Lamella Start Position and Landing Start Position to your named position. Update Protocol and Save Protocol (File Menu -> Save Protocol).

### Setup AutoLiftout

The Setup AutoLiftout workflow will walk you through selecting milling positions and their corresponding landing positions. 

1. The software will move the stage to the position specified as Lamella Start Position.  
2. Navigate to the region of interest, and select the position to mill the trenches. You can also adjust the milling patterns in the milling tab. The trench overlay is only for navigation and selection and won't be milled yet. Press Continue when you are happy. 
3. You will be asked if you want to continue selecting milling positions. Press Yes and Step 2 will be repeated for a new milling position. Continue selecting positions until you are happy. Press No to continue to select landing positions.
4. The software will move the stage to the position specified as Landing Start Position. 
5. Navigate to the landing post / grid, and mill the surface flat. Press Run Milling to mill the flatten pattern, and then press Continue when happy with the milling.
6. Step 5 will be repeated for each selected milling position.
7. Once you have selected and milled each landing position, the software will return to the Experiment Tab. You should see the status of each lamella. 
8. When you are ready, press Run AutoLiftout to begin the workflow. 

### Run AutoLiftout

The AutoLiftout workflow consists of four tasks; trench milling, undercut milling, liftout and landing. Trenches and Undercuts are the same workflow as in the waffle method, and can be completed in batch, but liftout and landing have to be completed sequentially for each lamella. 

For details on the trench and undercut milling, please see the above [walkthrough](#trench-milling). The only different between these workflows is the milling patterns for the trench and undercut which are defined in the protocol. 

#### Liftout

Before starting Liftout you will be asked if you want to Continue Lamella-01-XX from LiftoutLamella. You will need to complete both Liftout and Landing for each laemlla in sequence, and this is the last point you can interrupt without losing your lamella. Once you are ready, press Continue to start Liftout. Press Skip to continue to the next lamella. 

1. The software will move the stage to the undercut position, and restore the microscope state. 
2. The software will use machine learning feature detection to align the lamella (trench) centre in the electron and ion beams. If supervision is enabled, it will ask you to confirm the feature detections. Drag the feature to move it to the correct location (if required), and then press Continue to proceed with the workflow. 
3. The manipulator will be inserted above and to the left of the lamella trench. The software will use machine learning feature detection to detect the manipulator tip and the left edge of the lamella. If supervision is enabled, it will ask you to confirm the feature detections. Drag the feature to move it to the correct location (if required), and then press Continue to proceed with the workflow. 
4. Step 3 will be repeated several times in different beams to align the manipulator to the left of the lamella edge. If supervision if enabled you will be asked to confirm the position of the manipulator. Use the manipulator controls to adjust if required. 
5. The sample will be charged with the ion beam. The software will use machine learning feature detection to detect the manipulator tip and the left edge of the lamella and move them to contact. If supervision is enabled, it will ask you to confirm the feature detections. Drag the feature to move it to the correct location (if required), and then press Continue to proceed with the workflow. 
6. If using brightness based contact detection, the manipulator will be iteratively moved towards the lamella until contact is detected. If not, no additional movement is made. 
7. The manipulator is moved slightly up to avoid bottoming out, and make better surface contact with the lamella.
8. If liftout weld is selected, the manipulator is welded to the lamella with a milling redeposition pattern. If no weld is selected, the software continues on. 
9. The software will use machine learning feature detection to detect the right edge of the lamella. If supervision is enabled, it will ask you to confirm the feature detections. Drag the feature to move it to the correct location (if required), and then press Continue to proceed with the workflow. 
10. The sever milling pattern (sever) will be milled. If supervision is enabled, ask you to confirm the milling operation. Press Run Milling to mill the pattern, and then press Continue when happy with the milling.
11. The manipulator is removed from the trench, and then retracted fully.
12. The software will then ask you to continue with Land Lamella. It is recommended you complete landing immediately.

#### Landing

1. The softwre moves the stage to the landing position selected earlier and aligns to the reference image. 
2. The manipulator will be inserted above and to the left of the landing post. The software will use machine learning feature detection to detect the right edge of the lamella and the landing post. If supervision is enabled, it will ask you to confirm the feature detections. Drag the feature to move it to the correct location (if required), and then press Continue to proceed with the workflow.
4. Step 2 will be repeated several times in different beams to land the lamella right edge on the landing post.
5. After landing, if supervision is enabled, the software will ask you to confirm the lamella has landed. Press Continue if it has landed to continue with the workflow, or press Repeat to repeat the last step.
6. The software detects the lamella right edge to place the weld milling pattern. If supervision is enabled, it will ask you to confirm the feature detections. Drag the feature to move it to the correct location (if required), and then press Continue to proceed with the workflow.
7. The weld milling pattern (weld) will be milled. If supervision is enabled, ask you to confirm the milling operation. Press Run Milling to mill the pattern, and then press Continue when happy with the milling.
8. The sample will be discharged, and optionally this procedure can be repeated. If the manipulator was welded to the lamella, the cut is performed here. 
9. The manipulator moves back from the landing post. If supervision is enabled, you will be asked to confirm if the landing was successful. Press Continue to continue with the workflow, or Repeate to retry the landing attempt (deposition free attachment only).
10. You can optionally reset (re-sharpen, mill) the manipulator. This is not required for deposition free attachment method.

The software will now repeate Liftout and Landing for each selected position. Once all landings are completed (or you exit early), you can move onto polishing lamella with AutoLamella workflow. 

### Liftout - Run AutoLamella

From here, the workflow is exactly the same as the on-grid method. For more details please see [On-Grid Setup AutoLamella](#setup-autolamella). The only difference is the stage will move to a different angle for lamella milling as specified in the protocol under protocol["options"]["lamella_tilt_angle"]

The same as for the on-grid method, after the final polishing workflow completes the software will return the Experiment Tab and display the status of each lamella. It is recommended that you remove your sample after final polishing to prevent contamination build up.

## Serial Liftout Method

[Under Construction]
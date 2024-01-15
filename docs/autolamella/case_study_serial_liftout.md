# Case Study - Serial Liftout

Based on Step-by-Step Towards Successful Serial Lift-Out
[Supplementary of Serial Liftout Paper](
https://www.nature.com/articles/s41592-023-02113-5#Sec27)

Acknowledgement
not possible with data from mpi
discussion with sk and ohs about serial liftout method

Changes to workflows stages for serial liftout implementation

Code examples for completing steps throughout the workflow.

## Getting Started

For information on how to configure your microscope for use with openfibsem, please see [Getting Started](../openfibsem/getting_started.md)


### Connecting to the Microscope

Once you have configured your microscope, you should be able to sucessfully connect using your configuration.

```python

from fibsem import utils

# connect to microscope
microscope, settings = utils.setup_session(config_path="path/to/configuration.yaml", 
                                            protocol_path="path/to/protocol-serial-liftout.yaml")

```
The microscope object is the client connection to the microscope server, and settings is the configured settings of the microscope, for example the default imaging settings are available with settings.image.

You can access the protocol dictionary from settings.protocol.

## Terminology

We will use the following terminology in this guide. Please see the [Concepts Page](../openfibsem/concepts.md) for additional information. 

### Beam Coincidence

Beam coincidence refers to when the same feature is centred in both beams. Practically there will always be a small shift, but we want to minimise this where possible. Eucentric height is also refered to, as the position of the stage such that features stay in the same position when the stage is rotated. 


### Flat to Beam

When the stage is flat to a beam (e.g. flat to the electron beam) it is perpendicular to the imaging plane. Based on the microscope configuration (manufactuer, stage, and shuttle pre-tilt), we calculate the orientation required (rotation, and tilt) to move the stage to these positions. In the serial liftout appendix, the positions are given for a 45 deg shuttle-pre-tilt, and the term relative rotation is used. These map to the following flat to beam positions. 

- Flat to Electron Beam: 0 relative rotation,  shuttle-pre-tilt deg tilt
- Flat to Ion Beam: 180deg relative rotation, 52 - shuttle-pre-tilt deg tilt


### Movement Modes

- Stable Movement (Stage): Stable movements move along the sample plane, and maintain the coincidence of the beams. They correct for the stage tilt, shuttle pre-tilt and the imaging perspective. 
- Vertical Movement (Stage): Vertical movements move the stage vertically in the chamber, corrected for the stage tilt, pre-tilt and imaging perspective. They are used to realign beam coincidence (i.e. align ion to electron). 
- Corrected Movement (Manipulator): Corrected manipulator movements move only single axes at a time. The axes correspond to the imaging perspectives. Electron beam x and y directions map to the x, y axes and the Ion Beam x and y directions map to the x, and z axes. This allows you to move a single manipulator axes, without moving its position in the other beam.

As these movements all correct for the imaging perspective, they also take the beam type as a parameter. The beam type is where the imaging perspective correction is calculated from. Imaging perspective refers to the distortion when the imaging plane is not parallel to the sample plane. For example, imaging in the ion beam when the sample is flat to the electron beam causes a perspective distortion. 

### Microscope Configuration

The microscope configuration refers to how the microscope is initially configured, including specifying metatdata about the manufacturer, hardware and settings used. 

Protocol

The protocol refers to method specific parameters that are used to control how workflows are executed, select options, and define milling parameters. 

## Serial Liftout Dataset and Model

The MPI team generously provided a dataset from their serial liftout experiments. From this data we have labelled ~400 images from a workflow, and trained a segmentation model. 

```yaml
checkpoint: 'autolamella-serial-liftout-20240107.pt'
```
You can access both the dataset and model through the huggingface api. For more information on both the dataset and models, please see [Machine Learning Page](.ml.md).

We will be using examples from the dataset, and model inference throughout this guide. 

## AutoLamella Implementation

We have a work in progress implementation of Serial Liftout in AutoLamella
The current implementation in AutoLamella is slightly different than this example, due to additional experiment management integrations, logging and user interface interaction. You should be able to map these examples onto the implementation pretty closely however. 

You can try this in the AutoLiftout UI by selecting the autolamella/protocol-serial-liftout.yaml protocol, or selecting the autoliftout-serial-liftout method and configuring your protocol in the user interface. 

The specific workflow code is located:

- Core: autolamella/workflows/core.py
- Serial Liftout: autolamella/workflows/serial.py

If you want to try out this implementation workflow feel free, and if you would like any assistance please contact Patrick  on Github (@patrickcleeve2) or via [email](mailto:patrick@openfibsem.com).


## Serial Liftout Workflow

We will work through the explanatory protocol document, and demonstrate how we can implement the steps using the openfibsem api. We will start from the section Procedure:Preparatory Steps on page 16. This is the start of the FIBSEM operation, after sample preparation and vitrification.

This guide was written for a Thermo Fisher Hydra Plasma FIB, but should be general for other systems. The guide is intened more as an introduction to using openfibsem, rather than a complete automated workfloww. For current implementation in the user interface, please see [AutoLamella Implementation](#autolamella-implementation).

### Preparatory Steps

The goal of the preparatory steps is to prepare the manipulator and grids for the workflow. 

#### Step 1 - Prepare Manipulator

In this step we have to prepare and calibrate the manipulator. 

A. Focus and Link Stage
Currently it is recommend to manually focus and link the stage before starting openfibsem. Once you are more confident with the system, you can restore to a saved position to automatically skip this step. 

B. Beam Coincidence

To align the beams coincident, we can use the following steps:

1. Detect a Feature in Electron Beam
2. Move the Feature the centre of in Electron Beam (Stable Movement)
3. Detect the Feature in the Ion Beam
4. Move the stage vertically to move the Feature to the centre of the Ion Beam. (Vertical Movement)

We support multiple different ways of doing this coincident alignment, including manually via user input, alignment with reference images, and feature detection (ml) based alignment (discussed later). You can also perform this correction manually in the user interface by centred a feature with double click in the electron beam, then centring the same feature with alt + double click in the ion beam (to move vertically).

To start, we recommend you manually align the coincidence using the FIBSEM User Interface controls (Double Click to centre feature in Electron, Alt + Double Click to centre feature in Ion). 

C. Move the Shuttle Down
We move the shuttle down to avoid the manipulator making contact with the stage. All Fibsem stage positions are in the raw coordinate system (z positive is up). This coordinate system is independent of the linked (specimen) coordinate system, which is linked to the SEM working distance. 

```python

from fibsem.structures import FibsemStagePosition

# move the stage down
microscope.move_stage_relative(FibsemStagePosition(z=-2e-3))

```

D. Insert Manipulator

We insert the manipulator to examine its condition. Depending on when your system was last used and the manipulator was calibrated, this position may vary a lot. We will calibrate the manipulator in the next few steps. 

```python

# insert manipulator
microscope.insert_manipulator(name="PARK")

```

E. Prepare Manipulator Surface

We mill the bottom surface of the manipulator flat to prepare for attaching the copper adaptor. 

We can define our milling protocol as follows:

```yaml 

flatten:
    cleaning_cross_section: true
    depth: 1.0e-05
    height: 2.5e-6
    width: 20.0e-06
    hfw: 150.e-6
    milling_voltage: 30.0e+3
    milling_current: 2.8e-08
    rotation: 0.0
    scan_direction: BottomToTop
    application_file: "autolamella"
    type: "Rectangle"

```

We can use the model to detect the manipulator tip, and then place our milling pattern

```python

# detect points in ion beam at low mag
settings.image.hfw = 400e-6
settings.image.beam_type = BeamType.ION
features = [NeedleTip()]
det = detection.take_image_and_detect_features(
    microscope=microscope,
    settings=settings,
    features=features,
)

# offset detection, so we cut into manipulator
point = det.features[0].feature_m   # position of feature in metres (microscope image coordinates)
point.y -= 5e-6
point.x -= 5e-6

# get milling stages from protocol
stage = _get_milling_stages("flatten", settings.protocol, point)

# mill stages
milling.mill_stages(microscope, settings, stages)

```

F. Manipulator Calibration

We provide a manipulator calibration tool to assist in calibrating the EasyLift. Due to API limitations, the user still has to activate the calibration procedure in xTUI and then can follow the instructions in the tool to calibrate their EasyLift each day. The tool is available in the AutoLiftout UI via the Tools -> Calibrate Manipulator menu.  The tool uses the machine learning model to calibrate the manipulator. 


```python
from fibsem import calibration

# manipulator calibration
calibration._calibrate_manipulator_thermo(microscope, settings)

```

After calibrating, we can confirm our calibration was successful, by re-inserting to the saved positions, and checking the positions.

```python
from fibsem.structures import FibsemStagePosition

# make sure you move the stage down first
microscope.move_stage_relative(FibsemStagePosition(z=-1e-3))

# insert to parking position (~180um above stage)
microscope.insert_manipulator(name="PARK")

# insert to eucentric position (centre of both beams)
microscope.insert_manipulator(name="EUCENTRIC")

# retract manipulator
microscope.retract_manipulator()
```

#### Step 2, 3 - Clip and Load the Receiver Grid

These steps clip and and load the receiver grid into the FIBSEM. This is not something we can help with at the moment :D.

#### Step 4 - Copper Block Attachment

The steps prepare the copper adaptor block, and attach it to the manipulator. Figures https://www.nature.com/articles/s41592-023-02113-5/figures/7 show the process.

A. Mill Copper Bar

We move to the milling orientation, and mill the grid bar to be ~20um thick. 

```yaml

copper-bar-clean:
    cleaning_cross_section: false
    depth: 2.0e-05
    height: 5.0e-6
    width: 80.0e-06
    hfw: 150.e-6
    milling_voltage: 30.0e+3
    milling_current: 65.0e-9
    rotation: 0.0
    scan_direction: TopToBottom
    application_file: "autolamella"
    type: "Rectangle"

```

```python

import numpy as np

# first move flat to electron
microscope.move_flat_to_beam(settings, BeamType.ELECTRON)

# move to milling angle
milling_position = FibsemStagePosition(t=np.deg2rad(18))
microscope._safe_absolute_stage_movement(milling_position)

# get milling stages
stages = _get_milling_stages("copper-bar-clean", settings.protocol)

# run milling 
milling.mill_stages(microscope, settings, stages)

```

B. Rotate Flat to Ion

We rotate around flat to the ion beam.

```python

# move flat to ion
microscope.move_flat_to_beam(settings, BeamType.ION)

```

C. Mill Chain of Blocks

We mill a chain of blocks into the copper bar, leaving them attached to each other on the side. 

```yaml

copper-block-removal:
    cleaning_cross_section: false
    depth: 2.0e-05
    height: 20.0e-6
    width: 10.0e-06
    hfw: 150.e-6
    milling_voltage: 30.0e+3
    milling_current: 65.0e-9
    rotation: 0.0
    scan_direction: TopToBottom
    application_file: "autolamella"
    type: "Rectangle"
copper-block-removal-top:
    cleaning_cross_section: false
    depth: 2.0e-05
    height: 10.0e-6
    width: 110.0e-06
    hfw: 150.e-6
    milling_voltage: 30.0e+3
    milling_current: 65.0e-9
    rotation: 0.0
    scan_direction: TopToBottom
    application_file: "autolamella"
    type: "Rectangle"

```

```python

from fibsem import utils
from fibsem.patterning import _get_milling_stages
from fibsem.ui.utils import _draw_milling_stages_on_image 
from fibsem.structures import Point
from copy import deepcopy
import numpy as np

# get evenly spaced points
width = 100e-6
n_patterns = 4
pos_x = np.linspace(-width/2, width/2, n_patterns)
points = [Point(x, 0) for x in pos_x]

# acquire image for visualiation
settings.image.hfw = 150e-6
image = microscope.acquire_image(settings.image)

block_stages = []
for i, pt in enumerate(points):

    # get milling stages, at each position
    stage = _get_milling_stages("copper-block-removal", deepcopy(settings.protocol), deepcopy(pt))[0]
    stage.name = f"Copper-Block-Removal-{i:02d}"
    block_stages.append(deepcopy(stage))

# get top removal stage
top_stages= _get_milling_stages("copper-block-removal-top", settings.protocol, Point(0, 10e-6))

stages = top_stages + block_stages

# draw stages on image
fig = _draw_milling_stages_on_image(image, stages)

# run milling
milling.mill_stages(microscope, settings, stages)

```

TODO: draw patterns

D. Move Flat to Ion Beam

```python

# move flat to ion beam
microscope.move_flat_to_beam(settings, BeamType.ION)

```

E. Insert Maipulator

We insert the manipulator, and move it onto one of the blocks. This moving is best done manually at this point. 

```python

# insert manipulator
microscope.insert_manipulator(name="PARK")

```

F. Manipulator Contact

We make contact with the block face. We can run another milling stage to polish the face to ensure they are flat.

G. Attach the Block

H. Release the Block

I. Remove Manipulator

```python
# move manipulator up
microscope.move_manipulator_corrected(dx=0, dy=10e-6, beam_type=BeamType.ION)

```

J. Retract Manipulator
```python

# retract manipulator
microscope.retract_manipulator()

```

#### Step 5 - Prepare the Receiver Grid

We prepare for the double sided attachment.


```yaml
grid-lines:
    cleaning_cross_section: 0
    depth: 5.0e-05
    height: 1.0e-6
    width: 500.0e-06
    hfw: 900.0e-6
    milling_voltage: 30.0e+3
    milling_current: 2.8e-08
    rotation: 0.0
    scan_direction: TopToBottom
    application_file: "autolamella"
    type: "Rectangle" # TODO: update to Line

```

```python
# move flat to ion
microscope.move_flat_to_beam(settings, BeamType.ION)

# get milling stages
stages = _get_milling_stages("grid-lines", settings.protocol)

# mill stages
milling.mill_stages(microscope, settings, stages)

```

The manipulator and grid are now prepared. 



## Trench Milling

### Steps 1 - 6 Manual Setup

Steps 1 through 6 involve setting up the microscope, clearing contamination, platinum deposition and image correlation. At the moment these setup steps (focus and link, decontamination) are best performed manually  or not fully supported yet (correlation). 

### Step 7 - Low Magnification, High Resolution Image at SEM

You can use the movement and imaging api to move to the required orientations, and acquire reference images using the following:

```python

# move to imaging orientation -> flat to electron
microscope.move_flat_to_beam(settings, BeamType.ELECTRON)

# set imaging parameters
settings.image.beam_type = BeamType.ELECTRON
settings.image.resolution = [6144, 4096]
settings.image.dwell_time = 2e-6
settings.image.hfw = 2000e-6                            # size of grid
settings.image.label = f"ref_mapping_high_res_electron" # filename
settings.image.save = True

# acquire the image
image = acquire.new_image(microscope, settings.image)

```

### Step 8 - Platinum Deposition

You can use the deposition api, but it was developed for a system that had a multi-chem, I haven't been able to test it on a regular gis system. You can also access the deposition tool via the AutoLamella UI -> Tools -> Cryo Deposition.

Example cryo deposition api.

```python

from fibsem import gis

# define gis deposition protocol
gis_protocol = {
    "application_file": "cryo_Pt_dep",
    "gas": "Pt cryo",
    "position": "cryo",
    "hfw": 3.0e-05 ,
    "length": 7.0e-06,
    "beam_current": 1.0e-8,
    "time": 30.0,
}

# move to milling orientation -> flat to ion
microscope.move_flat_to_beam(settings, BeamType.ION)

# run cryo deposition at the current stage position
# the stage will move down by 1mm to avoid collision before sputtering.
gis.cryo_deposition(microscope, gis_protocol)

# run cryo deposition at the a named stage position
# You will need to define this named position through the Movement Tab (positions.yaml)
gis.cryo_deposition(microscope, gis_protocol name="cryo-deposition-position")
```

### Step 9 - Low Magnification, High Resolution Image Post platinum deposition

We can also acquire images after platinum deposition.

```python

# move to imaging orientation -> flat to electron
microscope.move_flat_to_beam(settings, BeamType.ELECTRON)

# set imaging parameters
settings.image.beam_type = BeamType.ELECTRON
settings.image.resolution = [6144, 4096]
settings.image.dwell_time = 2e-6
settings.image.hfw = 2000e-6                                # size of grid
settings.image.label = f"ref_mapping_high_res_electron_pt"  # filename
settings.image.save = True

# acquire the image
image = acquire.new_image(microscope, settings.image)

```

### Steps 10 - 11 Correlation

At the moment, correlation is best performed using external software.

You can acquire tilesets using the Minimap UI. It is available through the user interface -> Tools -> Open Minimap


### Step 12 - Move to Trench Milling Orientation

We can move to the trench milling orientation (flat to ion beam), with the following code:

```python
# move perpendicular to ion beam
microscope.move_flat_to_beam(settings, BeamType.ION)

```

### Step 13 - Align Reference Image

We can align the SEM reference image to the ION image with the following code. 

```python

# load reference image
ref_image = FibsemImage.load("path/to/reference_image.tif")

# rotate the reference 
ref_image_electron = image_utils.rotate_image(ref_image_electron)

# acquire ion image using same imaging settings
settings.image = ImageSettings.fromFibsemImage(ref_image)
settings.image.beam_type = BeamType.ION
new_image = acquire.new_image(microscope, settings.image)

# align reference
# NOTE: there are additional options for masking, and changing filters available
alignment.align_using_reference_images(microscope, settings, ref_image, new_image)
```

### Step 14 - Align Features Coincident

To align the beams to the feature of interest is coincident, we can use the following steps:

1. Detect a Feature in Electron Beam
2. Move the Feature the centre of in Electron Beam (Stable Movement)
3. Detect the Feature in the Ion Beam
4. Move the stage vertically to move the Feature to the centre of the Ion Beam. (Vertical Movement)

We support multiple different ways of doing this coincident alignment, including manually via user input, alignment with reference images, and feature detection (ml) based alignment (discussed later). You can also perform this correction manually in the user interface by centred a feature with double click in the electron beam, then centring the same feature with alt + double click in the ion beam (to move vertically).

```python


# example: pseudocode
# we are flat to the ion and want to rotate around 180 to be flat to the electron. then we need to re-align coincidence.
# assuming the beams were coincident prior to a rotation. We can take reference images before rotation, then cross align them after rotation to restore coincidence. 
# this is just pseudo code, real examples require more tuning and parameters to make it work repeatedly. For this reason, we prefer using the ml version.

# acquire reference images
ref_image_electron, ref_image_ion = acquire.take_reference_images(microscope, settings.image)

# rotate flat to electorn
microscope.move_flat_to_beam(settings, BeamType.ELECTRON)

# acquire new images
new_image_electron, new_image_ion = acquire.take_reference_images(microscope, settings.image)

# rotate references
ref_image_electron = image_utils.rotate_image(ref_image_electron)
ref_image_ion = image_utils.rotate_image(ref_image_ion)

# stable movement (step 1, 2)
alignment.align_using_reference_images(microscope, settings, ref_image_1, new_image_1)

# vertical movement (step 3, 4)
alignment.align_using_reference_images(microscope, settings, ref_image_2, new_image_2, constrain_vertical=True)

# the beams should now be coincident again. 

```


### Step 14 - Region of Interest

The region of interest is determined manually. Once the stage is moved to the correct position, we can save the state of the microscope with the following code:

```python
# save trench milling position
milling_state = microscope.get_current_microscope_state()

# we can restore back to this position / state at any time using:
microscope.set_microscope_state(milling_state)

# we can also save the state to file, to be reloaded later
utils.save_yaml("path/to/milling_state.yaml", milling_state.__to_dict__())

```

### Step 15 - Trench Milling

We can define the trench milling protocol as follows, we use a two stage milling protocol. The first stage mills the large trenches at high current, and the second polishes the contact surface. 

```yaml title="protocol-serial-liftout.yaml"
trench:
    stages:
    -   depth: 25.0e-6
        hfw: 400e-06
        height: 180.0e-06
        width: 4.5e-05
        milling_voltage: 30.0e+3
        milling_current: 3.0e-9
        rotation: 0.0
        scan_direction: TopToBottom
        side_trench_width: 5.0e-06
        top_trench_height: 30.0e-6
        application_file: "autolamella"
        type: "HorseshoeVertical"
        preset: "30 keV; 20 nA"
    -   depth: 25.0e-6
        hfw: 8.0e-05
        height: 2.5e-06
        width: 4.5e-05
        milling_voltage: 30.0e+3
        milling_current: 300.0e-12
        rotation: 0.0
        scan_direction: TopToBottom
        application_file: "autolamella"
        type: "Rectangle"

```

We can then run the trench milling with the following code.

```python
from fibsem import milling
from fibsem.patterning import _get_milling_stages

# move the polishing pattern to the top of the volume block
polishing_offset = Point(0, settings.protocol["trench"]["stages"][0]["height"] / 2)

# get milling stages from the protocol
stages = _get_milling_stages("trench", settings.protocol, [None, polishing_offset])

# run milling operations
milling.mill_stages(microscope, settings, stages)

```

We can also draw the milling stages on an image to see them before milling.
```python
from fibsem.ui.utils import _draw_milling_stages_on_image

# acquire image
settings.image.hfw = 400e-6
settings.image.beam_type = BeamType.ION
image = acquire.new_image(microscope, settings.image)

# draw milling stages
fig = _draw_milling_stages_on_image(image, stages)

```

### Step 16 - Acquire Reference Image

We can acquire the final trench reference images with the following code. 

```python

# move to imaging orientation -> flat to electron
microscope.move_flat_to_beam(settings, BeamType.ELECTRON)

# set imaging parameters
settings.image.beam_type = BeamType.ELECTRON
settings.image.resolution = [6144, 4096]
settings.image.dwell_time = 2e-6
settings.image.hfw = 2000e-6                                # size of grid
settings.image.label = f"ref_trench_milling_final"          # filename
settings.image.save = True

# acquire the image
image = acquire.new_image(microscope, settings.image)

```

## Liftout

The liftout steps attach the volume block to the manipulator, and extract it from the rest of the sample bulk.

### Steps 1 - 5 Manual Setup

Similar to Trench milling, steps 1 to 5 should be completed manually.

### Step 6 - Restore Milling State

We can restore our previously saved milling position / state.

```python

# load milling state from disk
milling_state = utils.load_yaml("path/to/milling_state.yaml")

# restore microscope state
microscope.set_microscope_state(milling_state)

```

## Landing

The landing steps attach the lamella to the landing grid, and sever it from the rest of the volume block. 

### Step 1 - Setup Stage

*"Move the stage to position the receiver grid in the field of view."*

We can restore to a previously saved position, such a starting position by first saving it, and then restoring it by name.

To save and restore a position:

```python

## save position
# get the current stage position
stage_position = microscope.get_stage_position()

# give your position a name
stage_position.name = "my-position-grid-01"

# save position to positions.yaml
# default path: fibsem/config/positions.yaml
utils.save_positions([stage_position])

## restore position
# you can restore these positions by loading the position by name:
stage_position = utils._get_position("my-position-grid-01")

# move to position (safely)
microscope._safe_absolute_stage_movement(stage_position)

```

### Step 2 - Move to Landing Orientation

*"Set the stage to lamella milling orientation (0° relative rotation to loading angle, 18° stage tilt)
and adjust the stage rotation to make sure that the pins or 400 mesh grid bars are aligned
vertical."*

We can restore to a previously defined position as shown before. In the autolamella application, the landing orientation is defined in the protocol as options/landing_start_position. You can define this position as shown above, or via the Movement Tab. 

```yaml title="protocol-serial-liftout.yaml"
options:
    landing_start_position: pre-tilt-35-deg-grid-02-landing
```

### Step 3 - Setup Landing Positions

*"Set up the coincidence points for all positions to be used for section attachment. Place them in the middle of the field of view and save the positions. If corrections of rotation are necessary to have perfectly vertically running 400-mesh grid bars or pins, perform them during this step and save them with the stage positions. Note: assuming the receiver grid is perfectly loaded, saving a single coincidence point per row may suffice."*

We will break this up into the following steps:

1. Select initial landing position
2. Generate landing position grid

We will generate a grid of landing positions, based on these protocol values. 

```yaml
options:
    landing_grid:
        x: 100.0e-6     # grid spacing in x
        y: 400.0e-6     # grid spacing in y
        rows: 4         # number of rows to generate
        cols: 10        # number of columns to generate
```

```python 

def generate_landing_positions(microscope, settings) -> list[FibsemStagePositions]:
    """Generate a grid of landing positions starting at the top left corner. Positions are 
    generated along the sample plane, based on the current orientation of the stage."""

    # base state = top left corner
    base_state = microscope.get_current_microscope_state()

    # get the landing grid protocol
    landing_grid_protocol = settings.protocol["options"]["landing_grid"]
    grid_square = Point(landing_grid_protocol['x'], landing_grid_protocol['y'])
    n_rows, n_cols = landing_grid_protocol['rows'], landing_grid_protocol['cols']

    positions = []

    for i in range(n_rows):
        for j in range(n_cols):
            _new_position = microscope._calculate_new_position( 
                settings=settings, 
                dx=grid_square.x*j, 
                dy=-grid_square.y*i, 
                beam_type=BeamType.ION, 
                base_position=base_state.absolute_position)            
            
            # position name is number of position in the grid
            _new_position.name = f"Landing Position {i*n_cols + j:02d}"
            
            positions.append(_new_position)
    
    return positions
```

TODO: show generated grid


Once we have selected our initial landing position, we can generate this grid of landing positions as follows:

```python

# user moves to initial landing position
initial_landing_position = 

# generate landing positions
positions = generate_landing_positions(microscope, settings)

# save landing positions to file
utils.save_positions("path/to/saved-landing-positions.yaml")

```

As these generated positions use the stable movement api (stage moves along the sample plane, coincidence is maintained), the positions should be relatively coincident across the entire grid. However, sample variation and damage to the grid can mean that the sample plane is not completely flat, breaking this assumption. 

### Step 4 - Move to Landing Position

"*Go back to the first section attachment position"*

This is the point we begin the landing workflow. The initial landing position is selected during setup, and then we use the generated landing position.  

```python

def create_lamella(microscope, experiment: Experiment, positions: list) -> Lamella:
    """Create a new lamella object, ready for landing at the next available landing position"""

    # create a new lamella 
    num = max(len(experiment.positions) + 1, 1)
    lamella = Lamella(experiment.path, num)

    # get the number of previously landed lamella
    _counter = Counter([p.state.stage.name for p in experiment.positions])
    land_idx = _counter[AutoLamellaStage.LandLamella.name]

    # set the state of the lamella, ready for landing
    lamella.state.stage = AutoLamellaStage.LiftoutLamella
    lamella.state.microscope_state = microscope.get_current_microscope_state()
    lamella.state.microscope_state.absolute_position = deepcopy(positions[land_idx])
    lamella.landing_state = deepcopy(lamella.state.microscope_state)

    return lamella

def landing_workflow(microscope, settings, experiment) -> Experiment:

    # generated positions
    positions = utils._get_positions("path/to/saved-landing-positions.yaml")

    # continue landing until exhausted
    continue_landing = True

    while continue_landing:

        # create a new lamella position
        lamella = create_lamella(microscope, experiment, positions)

        # land lamella
        lamella = land_lamella(
            microscope=microscope,
            settings=settings,
            lamella=lamella,
        )
        
        # continue with landing if user confirms, and enough material
        continue_landing = (
            ask_user(f"Continue Landing?") and
            validate_volume_block_size(microscope, settings)
        )

```


### Step 5 - Insert Manipulator

*"Re-insert the needle to which the extracted volume is attached."*

```python

# insert manipulator to park position (high above landing grid)
microscope.insert_manipulator(name="PARK")

```


### Step 6 - 

*"Lower the needle so that the extracted volume is about 10 µm above the first position."*

We break this down into the following steps:

1. Detect the bottom of the volume block, and the centre of the landing grid. Landing grid refers to the individual landing grid we intend to land on. 
2. Set an offset of 10um between these two points
3. Move the manipulator by the distance between these two points.


```python
# detect points in ion beam at low mag
settings.image.hfw = 400e-6
settings.image.beam_type = BeamType.ION
features = [VolumeBlockBottomEdge(), LandingGridCentre()]
det = detection.take_image_and_detect_features(
    microscope=microscope,
    settings=settings,
    features=features,
)

# set the offset y=10um
det._offset = Point(0, -10e-6)

# move based on detection
detection.move_based_on_detection(microscope, settings, det, beam_type=BeamType.ION)
```

### Step 7 - Position Extraction Volume

*"Position the extracted volume using the SEM channel.
a. For double-sided attachment, adjust y to align the leading edge of the volume with the
previously milled line pattern and adjust x to place the volume precisely between the
two grid bars. "*


```python
# detect points in electron beam at low mag
settings.image.hfw = 150e-6
settings.image.beam_type = BeamType.ELECTRON
features = [VolumeBlockBottomEdge(), LandingGridCentre()]
det = detection.take_image_and_detect_features(
    microscope=microscope,
    settings=settings,
    features=features,
)

# move based on detection
detection.move_based_on_detection(microscope, settings, det, beam_type=BeamType.ELECTRON)
```



### Step 8 - Lower Extraction Volume

*"Lower the extraction volume into place by adjusting z. Use the FIB channel as guidance.
Double-check intermittently for alignment with line/corner landmarks using the SEM channel.
a. For double sided attachment:
i. If the extraction volume is too wide to fit between the grid bars, mill off excess
material using regular cross sections or patterns (30kV, 300 pA). The extracted
volume should fit close to perfectly into the mesh.
ii. Align the lower front edge of the extraction volume with the previously milled
line.*"


We break this down into the following steps:

1. Check the size of the volume block and the landing grid.
2. If larger, mill the sides away. If not, continue.
3. Detect the bottom edge of the volume block, and the centre of the landing grid.
4. Move the manipulator down based on the detection (z-axis only).


```python


# check volume block size
# see if wider than grid bars gap 
# TODO:

# detect points in electron beam at low mag
settings.image.hfw = 150e-6
settings.image.beam_type = BeamType.ION
features = [VolumeBlockBottomEdge(), LandingGridCentre()]
det = detection.take_image_and_detect_features(
    microscope=microscope,
    settings=settings,
    features=features,
)

# move based on detection
detection.move_based_on_detection(microscope, settings, det, move_x=False, beam_type=BeamType.ION)
```


### Step 9 - Landing Attachment

*"Attach the extracted volume to the grid bars by redposition milling. Place the milling start of the patterns at on the interface of the grid bar and extraction volume."*

*"For double-sided attachment mill on both adjacent grid bars for attachment. Mill a vertical array of regular cross-sections (single pass, width 4.0 µm, height 0.5 µm, zdepth 10 µm, vertical spacing 0.25 µm, 30 kV, 1 nA) directed away from the extracted volume."*


We break this down into the following steps:

1. Detect corners of the volume block
2. Offset them by the height we want our lamella
3. Get milling stages from protocol
4. Mill the welds

First we need to define our weld milling protocol. We already support these kind of milling patterns, under the type "Spot Weld". A spot weld consists of a set of equally horizontal patterns, and is used as the name suggests for redeposition welds. Similar to attaching the  volume block to the adapter, setting passes = 1 is important so we don't mill over our redeposited material.

```yaml title="protocol-serial-liftout.yaml"
weld:
    stages:
    # left weld
    -   height: 0.5e-6
        width: 4.0e-6
        depth: 10.0e-6
        distance: 0.25e-6
        number: 5
        rotation: 0.0
        passes: 1.0
        milling_current: 300.0e-12
        milling_voltage: 1.0e-9
        hfw: 150.0e-6
        application_file: "autolamella"
        scan_direction: "RightToLeft"
        type: "SpotWeld"
    # right weld
    -   height: 2.5e-6
        width: 5.0e-6
        depth: 4.0e-6
        distance: 0.25e-6
        number: 5
        rotation: 0.0
        passes: 1.0
        milling_current: 1.0e-9
        milling_voltage: 30.0e+3
        hfw: 150.0e-6
        application_file: "autolamella"
        scan_direction: "LeftToRight"
        type: "SpotWeld"
```

This protocol gives us the following milling patterns.





We can now write the code for detecting the corners, offseting the patterns, and milling the welds. 

```python

# detect points in ion beam
settings.image.beam_type = BeamType.ION
features = [VolumeBlockBottomLeftCorner(), VolumeBlockBottomRightCorner()]
det = detection.take_image_and_detect_features(
    microscope=microscope,
    settings=settings,
    features=features,
)

# get the points
left_corner = det.features[0].feature_m 
right_corner = det.features[1].feature_m

# add some offset in y
v_offset = 2e-6  # half of recommended 4um height (step 11)
left_corner.y  +=  v_offset
right_corner.y +=  v_offset

# get weld milling stages
stages = milling._get_milling_stages("weld", settings.protocol, [left_point, right_point])

# mill stages
milling.mill_stages(microscope=microscope, settings=settings, stages=stages)
```

### Step 10 - Move Manipulator

*"Move the needle up by a step of 50-100 nm to create strain."*

```python

# move manipulator up 50-100 nm to create strain
dy = 100e-9
microscope.move_manipulator_corrected(dx=0, dy=dy, beam_type=BeamType.ION)

```

### Step 11 - Release Section

*"Release the section by milling a line pattern across the extracted volume at the desired sectioning distance from the lower edge of the extracted volume (30kV, 1 nA, z-depth 20 µm). Sectioning at 4 µm is recommended to begin with, but sections down to 1 µm can be obtained."*

We break this down into the following steps:

1. Detect bottom edge of the volume block
2. Offset them by the height we want our lamella
3. Get milling stages from protocol
4. Mill the sever

Milling protocol

```yaml title="protocol-serial-liftout.yaml"

landing_sever:
    cleaning_cross_section: 0.0
    depth: 20.0e-06
    height: 0.25e-06
    hfw: 150.0e-6
    milling_current: 1.0e-09
    milling_voltage: 30.0e+3
    rotation: 0.0
    scan_direction: LeftToRight
    width: 50.0e-06
    application_file: "autolamella"
    type: "Rectangle" # TODO: change to Line
```

```python

# detect points, in ion beam
settings.image.beam_type = BeamType.ION
features = [VolumeBlockBottomEdge()]
det = detection.take_image_and_detect_features(
    microscope=microscope,
    settings=settings,
    features=features,
)

# add some offset in y
v_offset = 4e-6  # recommended 4um height 
point = det.features[0].feature_m
point.y += v_offset

# get weld milling stages
stages = milling._get_milling_stages("landing_sever", settings.protocol, point)

# mill stages
milling.mill_stages(microscope=microscope, settings=settings, stages=stages)
```

### Step 12 - Validate Release

*"Check whether the section has been released by moving the needle up by 1-3 steps of 50 nm.
If the section moves with the needle, repeat the line pattern milling."*


We break this down into the following steps

1. Move the manipulator up by 50-100nm.
2. Detect the bottom edge of the volume block, and the top edge of the lamella.
3. Measure the distance between these points
4. If the distance is greater than threshold continue. If not, repeat the milling. 


```python

confirm_sever = False
while confirm_sever is False:

    # Step 11 - Mill Sever 
    # HERE

    # move the manipulator up a small amount
    for i in range(3):
        microscope.move_manipulator_corrected(dx=0, dy=50e-9, beam_type=BeamType.ION)

    # detect the distance between volume block and lamella
    settings.image.beam_type = BeamType.ION
    features = [VolumeBlockBottomEdge(), LamellaTopEdge()]
    det = detection.take_image_and_detect_features(
        microscope=microscope,
        settings=settings,
        features=features,
    )

    # check if the distance is greater than threshold 
    threshold = 0.5e-6
    confirm_sever = (abs(det.distance.y) > threshold)

# Step 13 - Move Manipulator up

```

### Step 13 - Move Manipulator Up

*"Once the section is released, carefully maneuver the extraction volume up. As soon as there is some distance to the section (~1 µm), increase the step size or jog. Move the volume up
slightly below the edge of the FIB image in lowest magnification"*

```python

# move up slowly
dy = 100e-9
for i in range(20):
    microscope.move_manipulator_corrected(dx=0, dy=dy, beam_type=BeamType.ION)

# move up
microscope.move_manipulator_corrected(dx=0, dy=100e-6, beam_type=BeamType.ION) # question: is this too much force?

```

### Step 14 - Retract Manipulator

*"Retract the needle"*.

```python
# retract the manipulator
microscope.retract_manipulator()
```

### Step 15 - Repeat

*"Move the stage to the next section attachment position and continue from step 5. Repeat this, until the extracted volume has been completely sectioned." 

At this point we would end our workflow stage, and continue to the next landing position to repeat the next landing. 

We also need to develop an automated stopping condition, to determine when the "volume has been completely sectioned". Again we can use the model to measure the remaining volume.

We break down this down into the following steps:

1. Assume the manipulator is inserted, or we are at a position where we can see the entire volume block. For example, at the start of the landing workflow after the manipulator is inserted. 
2. Detect the bounding box of the volume block.
3. Measure the size of the volume block bounding box
4. If the size is above a threshold, we continue to land more. If below, the volume block has been exhausted, and we need to clean the adapter and reset. 

```python

def validate_volume_block_size(microscope, settings) -> bool:
    """Validates if the volume block has enough material to continue landing"""

    # detect volume block features at low mag in ion beam
    settings.image.hfw = 400e-6
    settings.image.beam_type = BeamType.ION
    features = [VolumeBlockTopEdge(), VolumeBlockBottomEdge()]
    det = detection.take_image_and_detect_features(
        microscope=microscope,
        settings=settings,
        features=features,
    )

    # get distance
    volume_block_height = abs(det.distance.y) # distance between features

    # check threshold
    threshold = settings.protocol["options"].get("minimum_volume_size", 10e-6)

    continue_landing
    if volume_block_height >= threshold:
        continue_landing = True

    return continue_landing

```

## Section Thinning

Standard Polishing Workflow







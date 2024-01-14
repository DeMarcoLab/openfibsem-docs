# Case Study - Serial Liftout

Based on Step-by-Step Towards Successful Serial Lift-Out
[Supplementary of Serial Liftout Paper](
https://www.nature.com/articles/s41592-023-02113-5#Sec27)

Acknowledgement
not possible with data from mpi
discussion with sk and ohs about serial liftout method

Changes to workflows stages for serial liftout implementation

with code

## Terminology

Flat to Beam
Image Coincidence
Feature Detection



## Serial Liftout Model



## Workflow Changes


## Setup

### Select Positions

Select a single position

### Select Landing Positions

Select an initial landing position
Generating landing positions

### Liftout

Lift-Out

## Landing

Landing


### Step 1 - Setup Stage

*"Move the stage to position the receiver grid in the field of view."*

Not applicable has handled in setup stage. 

### Step 2 - Move to Landing Orientation

*"Set the stage to lamella milling orientation (0° relative rotation to loading angle, 18° stage tilt)
and adjust the stage rotation to make sure that the pins or 400 mesh grid bars are aligned
vertical."*

Not applicable as handled in setup stage.

The landing orientation is defined by
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
    positions = experiment.landing_positions

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
det._offset = Point(0, 10e-6)

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
    volume_block_height = det.distance # distance between features

    # check threshold
    threshold = settings.protocol["options"].get("minimum_volume_size", 10e-6)

    continue_landing
    if volume_block_height >= threshold:
        continue_landing = True

    return continue_landing

```

## Section Thinning

Standard Polishing Workflow







# Examples

The example directory contains a few example code snippets to help you get started with OpenFIBSEM.

The scripts are setup to run on the simulated microscope (manufactuer="Demo") by default. You can run them on a real microscope by specifying the manufacturer and ip_address in setup_session. Please becareful and understand the code before running on a real microscope.


Before running these scripts, make sure you have installed openfibsem, the manufacturers api, and activated your environment.

To activate your environment

```bash
conda activate fibsem
```

## Basic Example

This basic example script demonstrates how to connect to the microscope (setup_session), and take an image with both beams. The images are then displayed with matplotlib.

To run the script:

```python
python example/example.py
```

Code:

```python
from fibsem import utils, acquire

import matplotlib.pyplot as plt
import matplotlib
matplotlib.use('TkAgg', force=True) # Activate 'agg' backend for off-screen plotting.


def main():

    # connect to microscope
    microscope, settings = utils.setup_session(manufacturer="Demo", ip_address="localhost")

    # take image with both beams
    eb_image, ib_image = acquire.take_reference_images(microscope, settings.image)

    # show images
    fig, ax = plt.subplots(1, 2, figsize=(10, 5))
    ax[0].imshow(eb_image.data, cmap="gray")
    ax[0].set_title("Electron Beam Image")
    ax[0].axis("off")
    ax[1].imshow(ib_image.data, cmap="gray")
    ax[1].set_title("Ion Beam Image")
    ax[1].axis("off")
    plt.show()


if __name__ == "__main__":
    main()
```







## Imaging

The imaging example demonstrate how to take an image with both beams, and change the imaging settings. 

To run the script:

```python
python example/example_imaging.py
```

Code:

```python
import matplotlib
import matplotlib.pyplot as plt

from fibsem import acquire, utils
from fibsem.structures import BeamType
import logging

matplotlib.use('TkAgg', force=True) # Activate 'agg' backend for off-screen plotting.


"""
This script will take an image with the electron beam, an image with the ion beam, and an image with both beams. 
The images are then displayed in a matplotlib figure.

The settings for images are stored in the settings.image struct, and can be modified before taking an image.

For more detail on the settings, see the documentation for the ImageSettings class.

"""

def main():
    
    # connect to the microscope
    microscope, settings = utils.setup_session(manufacturer="Demo", ip_address="localhost")

    # info about ImageSettings
    logging.info(f"\nAcquiring Images Example:")
    logging.info(f"The current image settings are: \n{settings.image}")

    # take an image with the electron beam
    settings.image.beam_type = BeamType.ELECTRON
    eb_image = acquire.new_image(microscope, settings.image)

    # take an image with the ion beam
    settings.image.beam_type = BeamType.ION
    ib_image = acquire.new_image(microscope, settings.image)

    # take an image with both beams with increased hfw
    settings.image.hfw = 400e-6       
    ref_eb_image, ref_ib_image = acquire.take_reference_images(microscope, settings.image)

    # show images

    fig, ax = plt.subplots(2, 2, figsize=(10, 7))
    ax[0][0].imshow(eb_image.data, cmap="gray")
    ax[0][0].set_title("Electron Image 01")
    ax[0][1].imshow(ib_image.data, cmap="gray")
    ax[0][1].set_title("Ion Image 01")
    ax[1][0].imshow(ref_eb_image.data, cmap="gray")
    ax[1][0].set_title("Electron Image 02 (Reference)")
    ax[1][1].imshow(ref_ib_image.data, cmap="gray")
    ax[1][1].set_title("Ion Image 02 (Reference)")
    plt.show()


if __name__ == "__main__":
    main()

```

## Movement

The movement example

To run the script

```python
python example/example_movement.py
```

Code:

```python
from fibsem import utils
from fibsem.structures import  FibsemStagePosition
import numpy as np
import logging


"""
This script demonstrates how to get the current stage position, and how to move the stage to a new position.

The basic movement methods are absolute_move and relative_move. 
- Relative move moves the stage by a certain amount in the current coordinate system.
- Absolute move moves the stage to a new position in the absolute coordinate system. 

This script will move the stage by 20um in the x direction (relative move), and then move back to the original position (absolute move).

Additional movement methods are available in the core api:
- Stable Move: the stage moves along the sample plane, accounting for stage tilt, and shuttle pre-tilt
- Vertical Move: the stage moves vertically in the chamber, regardless of tilt orientation

"""

def main():

    # connect to microscope
    microscope, settings = utils.setup_session(manufacturer="Demo", ip_address="localhost")
    
        # info about ImageSettings
    logging.info("---------------------------------- Current Position ----------------------------------\n")

    # get current position
    intial_position = microscope.get_stage_position()
    logging.info(f"\nStage Movement Example:")
    logging.info(f"Current stage position: {intial_position}")
    

    logging.info("\n---------------------------------- Relative Movement ----------------------------------\n")

    #### Moving to a relative position ####
    relative_move = FibsemStagePosition(x=20e-6,            # metres
                                        y=0,                # metres
                                        z=0.0,              # metres
                                        r=np.deg2rad(0),    # radians
                                        t=np.deg2rad(0))    # radians
    
    input(f"Press Enter to move by: {relative_move} (Relative)")
    
    # move by relative position    
    microscope.move_stage_relative(relative_move)
    current_position = microscope.get_stage_position()
    logging.info(f"After move stage position: {current_position}")


    logging.info("\n---------------------------------- Absolute Movement ----------------------------------\n")

    #### Moving to an absolute position ####
    stage_position = intial_position # move back to initial position

    # uncomment this if you want to move to a different position 
    # be careful to define a safe position to move too
    # relative_move = FibsemStagePosition(x=0,                # metres
    #                                     y=0,                # metres
    #                                 z=0.0,                  # metres
    #                                     r=np.deg2rad(0),    # radians
    #                                     t=np.deg2rad(0))    # radians

    input(f"Press Enter to move to: {stage_position} (Absolute)")

    # move to absolute position
    microscope.move_stage_absolute(stage_position) 
    current_position = microscope.get_stage_position()
    logging.info(f"After move stage position: {current_position}")


    logging.info("---------------------------------- End Example ----------------------------------")
   

if __name__ == "__main__":
    main()
```

## Milling

The milling example demonstrates how the define milling patterns, and run ion beam milling. Note: at the moment only ion beam milling is supported, we hope to add electron beam patterning in the future. 

```python
python example/example_milling_.py
```

```python

from fibsem import utils
from fibsem.structures import FibsemPatternSettings, FibsemPattern, FibsemMillingSettings
from fibsem import milling
import logging

"""
This script demonstrates how to use the milling module to mill a rectangle and two lines.

The script will:
    - connect to the microscope
    - setup milling
    - draw a rectangle and two lines
    - run milling
    - finish milling (restore ion beam current)

"""

def main():

    # connect to microscope
    microscope, settings = utils.setup_session(manufacturer="Demo", ip_address="localhost")

    # rectangle pattern
    rectangle_pattern = FibsemPatternSettings(
        pattern = FibsemPattern.Rectangle,
        width = 10.0e-6,
        height = 10.0e-6,
        depth = 2.0e-6,
        rotation = 0.0,
        center_x = 0.0,
        center_y = 0.0,
    )

    # line pattern one
    line_pattern_01 = FibsemPatternSettings(
        pattern = FibsemPattern.Line,
        start_x = 0.0,
        start_y = 0.0,
        end_x = 10.0e-6,
        end_y = 10.0e-6,
        depth = 2.0e-6,
    )

    # line pattern two (mirror of line pattern one)
    line_pattern_02 = line_pattern_01
    line_pattern_02.end_y = -line_pattern_01.end_y

    logging.info(f"""\nMilling Pattern Example:""")

    logging.info(f"The current milling settings are: \n{settings.milling}")
    logging.info(f"The current rectangle pattern is \n{rectangle_pattern}")
    logging.info(f"The current line pattern one is \n{line_pattern_01}")
    logging.info(f"The current line pattern two is \n{line_pattern_02}")
    logging.info("---------------------------------- Milling ----------------------------------\n")
    # setup patterns in a list
    patterns = [rectangle_pattern, line_pattern_01, line_pattern_02]

    # setup milling
    milling.setup_milling(microscope, settings.milling)

    # draw patterns
    for pattern in patterns:
        milling.draw_pattern(microscope, pattern)

    # run milling
    milling.run_milling(microscope, settings.milling.milling_current, milling_voltage=settings.milling.milling_voltage)

    # finish milling
    milling.finish_milling(microscope, settings.system.ion.current)


if __name__ == "__main__":
    main()
```

## AutoLamella

The autolamella script is a minimal recreation of the original autolamella program in ~150 lines of code using OpenFIBSEM. For the original paper please see [AutoLamella V1 Paper](https://doi.org/10.1016/j.jsb.2020.107488)

To run the script:

```python
python example/autolamella.py
```

Code: 

```python

import logging
import os
from dataclasses import dataclass
from pathlib import Path
from pprint import pprint

import numpy as np
from fibsem import acquire, alignment, calibration, milling, movement, utils
from fibsem.structures import BeamType, MicroscopeState,  FibsemImage, FibsemStagePosition


@dataclass
class Lamella:
    state: MicroscopeState
    reference_image: FibsemImage
    path: Path

def main():

    PROTOCOL_PATH = os.path.join(os.path.dirname(__file__), "protocol_autolamella.yaml")
    microscope, settings = utils.setup_session(protocol_path=PROTOCOL_PATH)
    
    # move to the milling angle
    stage_position = FibsemStagePosition(
        r=np.deg2rad(settings.protocol["stage_rotation"]),
        t=np.deg2rad(settings.protocol["stage_tilt"])
    )
    microscope.move_stage_absolute(stage_position) # do need a safe version?

    # take a reference image    
    settings.image.label = "grid_reference"
    settings.image.beam_type = BeamType.ION
    settings.image.hfw = 900e-6
    settings.image.save = True
    acquire.take_reference_images(microscope, settings.image)

    # select positions
    experiment: list[Lamella] = []
    lamella_no = 1
    settings.image.hfw = 80e-6
    base_path = settings.image.save_path

    while True:
        response = input(f"""Move to the desired position. 
        Do you want to select another lamella? [y]/n {len(experiment)} selected so far.""")

        # store lamella information
        if response.lower() in ["", "y", "yes"]:
            
            # set filepaths
            path = os.path.join(base_path, f"{lamella_no:02d}")
            settings.image.save_path = path
            settings.image.label = f"ref_lamella"
            acquire.take_reference_images(microscope, settings.image)

            lamella = Lamella(
                state=microscope.get_current_microscope_state(),
                reference_image=acquire.new_image(microscope, settings.image),
                path = path
            )
            experiment.append(lamella)
            lamella_no += 1
        else:
            break

    # sanity check
    if len(experiment) == 0:
        logging.info(f"No lamella positions selected. Exiting.")
        return

    # setup milling
    settings.application_file = settings.protocol.get("application_file", "autolamella")
    milling.setup_milling(microscope = microscope,
        mill_settings = settings.milling)

    # mill (fiducial, trench, thin, polish)
    for stage_no, milling_dict in enumerate(settings.protocol["lamella"]["protocol_stages"], 1):
        
        logging.info(f"Starting milling stage {stage_no}")

        lamella: Lamella
        for lamella_no, lamella in enumerate(experiment):

            logging.info(f"Starting lamella {lamella_no:02d}")

            # return to lamella
            microscope.set_microscope_state(lamella.state)

            # realign
            alignment.beam_shift_alignment(microscope, settings.image, lamella.reference_image)
                       
            if stage_no == 0:
                logging.info("add microexpansion joints here")

            # mill trenches
            milling.draw_trench(microscope, milling_dict)
            milling.run_milling(microscope, milling_dict["milling_current"], milling_dict["milling_voltage"])
            milling.finish_milling(microscope)

            # retake reference image
            settings.image.save_path = lamella.path
            settings.image.label = f"ref_mill_stage_{stage_no:02d}"
            lamella.reference_image = acquire.new_image(microscope, settings.image)

            if stage_no == 3:
                # take final reference images
                settings.image.label = f"ref_final"
                acquire.take_reference_images(microscope, settings.image)
   
    logging.info(f"Finished autolamella: {settings.protocol['name']}")


if __name__ == "__main__":
    main()


```
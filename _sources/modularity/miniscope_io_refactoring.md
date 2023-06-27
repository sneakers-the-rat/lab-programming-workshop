# Example: Miniscope IO

**Scenario:** We have a bunch of different miniscopes that may each have their own firmware and format when writing acquired videos to disk. We would like to make a package to standardize the format and provide a simple I/O wrapper around it.

Follow along with the code in the package:

https://github.com/Aharoni-Lab/miniscope-io

## Problems

There is a bug in the way that videos are acquired that is likely a hardware bug, but the software isn't helping us understand it at all:

<video src="../_images/wirefree.mp4" title="Wire-Free Miniscope Example" controls><a href="../_images/wirefree.mp4">Wire-Free Miniscope Example</a></video>

We also have the problem where we have implemented the same I/O routines in multiple places with various functionality!

- Wireless-Miniscope
    - https://github.com/Aharoni-Lab/Wireless-Miniscope/blob/master/image_acquisition/offline_DAQ.ipynb
- Miniscope-v4-Wire-Free
    - Reading: https://github.com/Aharoni-Lab/Miniscope-v4-Wire-Free/blob/master/Miniscope-v4-Wire-Free-Python%20DAQ%20Interface/Load%20raw%20data%20from%20SD%20card%20and%20write%20video%20-%20WireFree%20V4%20Miniscope.ipynb
    - Writing: https://github.com/Aharoni-Lab/Miniscope-v4-Wire-Free/blob/master/Miniscope-v4-Wire-Free-Python%20DAQ%20Interface/Set%20recording%20parameters%20-%20WireFree%20V4%20Miniscope.ipynb
- Miniscope-Wire-Free-DAQ
    -  https://github.com/Aharoni-Lab/Miniscope-Wire-Free-DAQ/blob/master/Wire-Free-DAQ-Software/Wire-Free-DAQ.ipynb

And the format is loosely coupled to how the data is written by the capturing firmware

- https://github.com/Aharoni-Lab/Miniscope-v4-Wire-Free/blob/1a3ba1163786f86421c3b2a484c4462427ef9917/Miniscope-v4-Wire-Free-MCU-Firmware/Miniscope-v4-Wire-Free/definitions.h#L39
- https://github.com/Aharoni-Lab/Miniscope-SAMD-Framework/blob/main/include/MS_definitions.h

## Overview

What can we do!?

Much like the [previous](modularity) lesson, we are going to 

- Break up the logic into smaller pieces
- Move Notebook code into a package
- Create interfaces between the format specification and the I/O class

## Format

### Current Status!

We first notice some inconsistencies between a format from 3 years ago and the current version!

`````{tab-set}
````{tab-item} New Version
```python
# SD Card sector information
headerSector =          1022 # Holds user settings to configure Miniscope and recording
configSector =          1023 # Holds final settings of the actual recording
dataStartSector =       1024 # Recording data starts here
sectorSize =            512

WRITE_KEY0 =                0x0D7CBA17
WRITE_KEY1 =                0x0D7CBA17
WRITE_KEY2 =                0x0D7CBA17
WRITE_KEY3 =                0x0D7CBA17

# SD Card Header Sector positions
HEADER_GAIN_POS =               4
HEADER_LED_POS =                5
HEADER_EWL_POS =                6
HEADER_RECORD_LENGTH_POS =      7
HEADER_FRAME_RATE =             8

# SD Card Config Sector positions
CONFIG_BLOCK_WIDTH_POS =                0
CONFIG_BLOCK_HEIGHT_POS =               1
CONFIG_BLOCK_FRAME_RATE_POS =           2
CONFIG_BLOCK_BUFFER_SIZE_POS =          3
CONFIG_BLOCK_NUM_BUFFERS_RECORDED_POS = 4
CONFIG_BLOCK_NUM_BUFFERS_DROPPED_POS =  5

# Data Buffer Header positions
BUFFER_HEADER_HEADER_LENGTH_POS =           0
BUFFER_HEADER_LINKED_LIST_POS =             1
BUFFER_HEADER_FRAME_NUM_POS =               2
BUFFER_HEADER_BUFFER_COUNT_POS =            3
BUFFER_HEADER_FRAME_BUFFER_COUNT_POS =      4
BUFFER_HEADER_WRITE_BUFFER_COUNT_POS =      5
BUFFER_HEADER_DROPPED_BUFFER_COUNT_POS =    6
BUFFER_HEADER_TIMESTAMP_POS =               7
BUFFER_HEADER_DATA_LENGTH_POS =             8
BUFFER_HEADER_WRITE_TIMESTAMP_POS =         9
```
````
````{tab-item} Old Version
```python
# SD Card sector information
headerSector =          1023 # Holds user settings to configure Miniscope and recording
configSector =          1024 # Holds final settings of the actual recording
dataStartSector =       1025 # Recording data starts here
sectorSize =            512

WRITE_KEY0 =                0x0D7CBA17
WRITE_KEY1 =                0x0D7CBA17
WRITE_KEY2 =                0x0D7CBA17
WRITE_KEY3 =                0x0D7CBA17

# SD Card Header Sector positions
HEADER_GAIN_POS =               4
HEADER_LED_POS =                5
HEADER_EWL_POS =                6
HEADER_RECORD_LENGTH_POS =      7
HEADER_FRAME_RATE =             8

# SD Card Config Sector positions
CONFIG_BLOCK_WIDTH_POS =                0
CONFIG_BLOCK_HEIGHT_POS =               1
CONFIG_BLOCK_FRAME_RATE_POS =           2
CONFIG_BLOCK_BUFFER_SIZE_POS =          3
CONFIG_BLOCK_NUM_BUFFERS_RECORDED_POS = 4
CONFIG_BLOCK_NUM_BUFFERS_DROPPED_POS =  5

# Data Buffer Header positions
BUFFER_HEADER_HEADER_LENGTH_POS =           0
BUFFER_HEADER_LINKED_LIST_POS =             1
BUFFER_HEADER_FRAME_NUM_POS =               2
BUFFER_HEADER_BUFFER_COUNT_POS =            3
BUFFER_HEADER_FRAME_BUFFER_COUNT_POS =      4
BUFFER_HEADER_WRITE_BUFFER_COUNT_POS =      5
BUFFER_HEADER_DROPPED_BUFFER_COUNT_POS =    6
BUFFER_HEADER_TIMESTAMP_POS =               7
BUFFER_HEADER_DATA_LENGTH_POS =             8
```
````
`````

The new version changes the location of the header sectors, and adds an additional field for a write timestamp.

Additionally, in the [formatting code](https://github.com/Aharoni-Lab/Miniscope-v4-Wire-Free/blob/master/Miniscope-v4-Wire-Free-Python%20DAQ%20Interface/Set%20recording%20parameters%20-%20WireFree%20V4%20Miniscope.ipynb) bundled with the new version of the reading code, we have inconsistent header fields:

`````{tab-set}
````{tab-item} Reading
```python
# SD Card Header Sector positions
HEADER_GAIN_POS =               4
HEADER_LED_POS =                5
HEADER_EWL_POS =                6
HEADER_RECORD_LENGTH_POS =      7
HEADER_FRAME_RATE =             8
```
````
````{tab-item} Writing
```python
# SD Card Header Sector positions
HEADER_GAIN_POS =               4
HEADER_LED_POS =                5
HEADER_EWL_POS =                6
HEADER_RECORD_LENGTH_POS =      7
HEADER_FRAME_RATE =             8
HEADER_DELAY_START_POS =        9
HEADER_BATT_CUTOFF_POS =        10
```
````
`````

So even within the same microscope, over time we have format drift! We would like to be able to account for this drift, as well as make it easier to add, remove, and move fields for different microscopes. There is also some zombie code - the `WRITE_KEY`s above don't seem to be used, but there's no way for me to be sure!

We can make something that we can rely on rather than needing to juggle notebook code!

### Format abstraction

We're going to use our trusty friend [pydantic](https://docs.pydantic.dev/latest/) to make some data models that mimic the above configurations.


```{note}
We are just transcribing code at this point rather than rearchitecting it - ideally we would have some kind of data format that would allow us to unambiguously address the header and other metadata without needing to compute from sector size and etc. but we will get there!
```

The current code mixes information about the format with the code used to read it. We are instead going to separate this out into three parts

- Abstract specification of format - the possible values that a format spec can use, as well as some defaults
- Specific formats - The concrete values used by a particular miniscope
- I/O - a reader/writer that uses a format configuration to read and write video data.

### Sector Config

First, we want to configure the highest-level of the format: where the different sections are, and how large a sector is on the disk.

```python
from pydantic import BaseModel

class SectorConfig(BaseModel):
    """
    Configuration of sector layout on the SD card.

    For each sector, one can retrieve the position with the attribute *_pos,

    eg.

    >>> sectors = SectorConfig(header=1023, config=1024, data=1025, size=512)
    >>> sectors.header
    1024
    >>> # should be 1023 * 512
    >>> sectors.header_pos
    523776
    """

    header: int = 1023
    """
    Holds user settings to configure Miniscope and recording
    """
    config: int = 1024
    """
    Holds final settings of the actual recording
    """
    data: int = 1025
    """
    Recording data starts here
    """
    size: int = 512
    """
    The size of an individual sector
    """

    def __getattr__(self, item:str) -> int:
        """
        Get positions by multiplying by sector size
        (__getattr__ is only called if the name can't be found, so we don't need to handle
        the base case of the existing attributes)
        """
        split = item.split('_')
        if len(split) == 2 and split[1] == "pos":
            return getattr(self, split[0]) * getattr(self, 'size')
        else:
            raise AttributeError()

```

The top part illustrates the basic use of pydantic for data models:

- Declare the **fields** of this model with class attributes
- Each field has a type annotation, in this case all `int`s. Pydantic will validate these when the model is instantiated
- Each field also has a default value that will be used if none is provided explicitly.
- We use `"""docstrings"""` to explain what each parameter is (copied from the original notebook), and that will be included in any generated documentation we use, as well as the JSON-schema that is generated by pydantic.

We do a little something extra for quality of life. Since each position is a number of sectors from the beginning of the drive, we override the `__getattr__` method so that each field (eg. `header`) has a "virtual" `header_pos` field that returns `header` * `size`. This prevents us from needing to repeat ourselves every time we need to do multiplication, and also creates a clear semantic meaning to the different numbers.

### SD Header

Rather than having a flat series of attributes, we can bundle together the different sets of disk positions. For example, the header for the whole SD card:

```python
from typing import Optional

class SDHeaderPositions(BaseModel):
    """
    Positions in the header for the whole SD card
    """
    gain: int = 4
    led: int = 5
    ewl: int = 6
    record_length: int = 7
    fs: int = 8
    """Frame rate"""
    delay_start: Optional[int] = None
    battery_cutoff: Optional[int] = None
```

This model is the same, except we introduce the notion of optional parameters, by default set to `None`. We will write the code that consumes this object in such a way that it can handle unset values, since we introduced the `delay_start` as a feature after the original format, but don't want to break it!

### All Together

We combine several of these sub-models together in the top-level model:

```python
class SDLayout(BaseModel):
    """
    Data layout of an SD Card.

    Used by the :class:`.io.SDCard` class to tell it how data on the SD card is laid out.
    """
    sectors: SectorConfig
    write_key0: int = 0x0D7CBA17
    write_key1: int = 0x0D7CBA17
    write_key2: int = 0x0D7CBA17
    write_key3: int = 0x0D7CBA17
    """
    These don't seem to actually be used in the existing reading/writing code, but we will leave them here for continuity's sake :)
    """
    word_size: int = 4
    """
    I'm actually not sure what this is, but 4 is hardcoded a few times in the existing notebook and it
    appears to be used as a word size when reading from the SD card.
    """

    header: SDHeaderPositions = SDHeaderPositions()
    config: ConfigPositions = ConfigPositions()
    buffer: BufferHeaderPositions = BufferHeaderPositions()

```

Which is a tidy, abstract representation of the various fields we can have in our data structure.

Conveniently, pydantic also makes a standardized JSON-schema for our config, so we get some interoperability for free, even if there isn't quite enough information to actually make use of it:

If we call `SDLayout.schema()`, we get...

````{dropdown} JSON Schema (click to expand)
```json
{
  "title": "SDLayout",
  "description": "Data layout of an SD Card.\\n\\nUsed by the :class:`.io.SDCard` class to tell it how data on the SD card is laid out.",
  "type": "object",
  "properties":
  {
    "sectors":
    {
      "$ref": "#/definitions/SectorConfig"
    },
    "write_key0":
    {
      "title": "Write Key0",
      "default": 226277911,
      "type": "integer"
    },
    "write_key1":
    {
      "title": "Write Key1",
      "default": 226277911,
      "type": "integer"
    },
    "write_key2":
    {
      "title": "Write Key2",
      "default": 226277911,
      "type": "integer"
    },
    "write_key3":
    {
      "title": "Write Key3",
      "default": 226277911,
      "type": "integer"
    },
    "word_size":
    {
      "title": "Word Size",
      "default": 4,
      "type": "integer"
    },
    "header":
    {
      "title": "Header",
      "default":
      {
        "gain": 4,
        "led": 5,
        "ewl": 6,
        "record_length": 7,
        "fs": 8,
        "delay_start": null,
        "battery_cutoff": null
      },
      "allOf":
      [
        {
          "$ref": "#/definitions/SDHeaderPositions"
        }
      ]
    },
    "config":
    {
      "title": "Config",
      "default":
      {
        "width": 0,
        "height": 1,
        "fs": 2,
        "buffer_size": 3,
        "n_buffers_recorded": 4,
        "n_buffers_dropped": 5
      },
      "allOf":
      [
        {
          "$ref": "#/definitions/ConfigPositions"
        }
      ]
    },
    "buffer":
    {
      "title": "Buffer",
      "default":
      {
        "length": 0,
        "linked_list": 1,
        "frame_num": 2,
        "buffer_count": 3,
        "frame_buffer_count": 4,
        "write_buffer_count": 5,
        "dropped_buffer_count": 6,
        "timestamp": 7,
        "data_length": 8,
        "write_timestamp": null
      },
      "allOf":
      [
        {
          "$ref": "#/definitions/BufferHeaderPositions"
        }
      ]
    }
  },
  "required":
  [
    "sectors"
  ],
  "definitions":
  {
    "SectorConfig":
    {
      "title": "SectorConfig",
      "description": "Configuration of sector layout on the SD card.\\n\\nFor each sector, one can retrieve the position with the attribute *_pos,\\n\\neg.\\n\\n>>> sectors = SectorConfig(header=1023, config=1024, data=1025, size=512)\\n>>> sectors.header\\n1024\\n>>> # should be 1023 * 512\\n>>> sectors.header_pos\\n523776",
      "type": "object",
      "properties":
      {
        "header":
        {
          "title": "Header",
          "default": 1023,
          "type": "integer"
        },
        "config":
        {
          "title": "Config",
          "default": 1024,
          "type": "integer"
        },
        "data":
        {
          "title": "Data",
          "default": 1025,
          "type": "integer"
        },
        "size":
        {
          "title": "Size",
          "default": 512,
          "type": "integer"
        }
      }
    },
    "SDHeaderPositions":
    {
      "title": "SDHeaderPositions",
      "description": "Positions in the header for the whole SD card",
      "type": "object",
      "properties":
      {
        "gain":
        {
          "title": "Gain",
          "default": 4,
          "type": "integer"
        },
        "led":
        {
          "title": "Led",
          "default": 5,
          "type": "integer"
        },
        "ewl":
        {
          "title": "Ewl",
          "default": 6,
          "type": "integer"
        },
        "record_length":
        {
          "title": "Record Length",
          "default": 7,
          "type": "integer"
        },
        "fs":
        {
          "title": "Fs",
          "default": 8,
          "type": "integer"
        },
        "delay_start":
        {
          "title": "Delay Start",
          "type": "integer"
        },
        "battery_cutoff":
        {
          "title": "Battery Cutoff",
          "type": "integer"
        }
      }
    },
    "ConfigPositions":
    {
      "title": "ConfigPositions",
      "description": "Image acquisition configuration positions",
      "type": "object",
      "properties":
      {
        "width":
        {
          "title": "Width",
          "default": 0,
          "type": "integer"
        },
        "height":
        {
          "title": "Height",
          "default": 1,
          "type": "integer"
        },
        "fs":
        {
          "title": "Fs",
          "default": 2,
          "type": "integer"
        },
        "buffer_size":
        {
          "title": "Buffer Size",
          "default": 3,
          "type": "integer"
        },
        "n_buffers_recorded":
        {
          "title": "N Buffers Recorded",
          "default": 4,
          "type": "integer"
        },
        "n_buffers_dropped":
        {
          "title": "N Buffers Dropped",
          "default": 5,
          "type": "integer"
        }
      }
    },
    "BufferHeaderPositions":
    {
      "title": "BufferHeaderPositions",
      "description": "Positions in the header for each frame",
      "type": "object",
      "properties":
      {
        "length":
        {
          "title": "Length",
          "default": 0,
          "type": "integer"
        },
        "linked_list":
        {
          "title": "Linked List",
          "default": 1,
          "type": "integer"
        },
        "frame_num":
        {
          "title": "Frame Num",
          "default": 2,
          "type": "integer"
        },
        "buffer_count":
        {
          "title": "Buffer Count",
          "default": 3,
          "type": "integer"
        },
        "frame_buffer_count":
        {
          "title": "Frame Buffer Count",
          "default": 4,
          "type": "integer"
        },
        "write_buffer_count":
        {
          "title": "Write Buffer Count",
          "default": 5,
          "type": "integer"
        },
        "dropped_buffer_count":
        {
          "title": "Dropped Buffer Count",
          "default": 6,
          "type": "integer"
        },
        "timestamp":
        {
          "title": "Timestamp",
          "default": 7,
          "type": "integer"
        },
        "data_length":
        {
          "title": "Data Length",
          "default": 8,
          "type": "integer"
        },
        "write_timestamp":
        {
          "title": "Write Timestamp",
          "type": "integer"
        }
      }
    }
  }
}
```
````

### Specific Formats

We then use this schematic model to declare the specific values for the wirefree miniscope (conveniently, most of these are the default values, wonder why that is lmao)

In `miniscope_io/formats.py`, we find:

````{dropdown} WireFreeSDLayout Format Example
```python
from miniscope_io.sdcard import \
    SDLayout, \
    SectorConfig, \
    SDHeaderPositions, \
    BufferHeaderPositions, \
    ConfigPositions

WireFreeSDLayout = SDLayout(
    sectors=SectorConfig(
        header = 1022,
        config = 1023,
        data   = 1024,
        size   = 512
    ),
    write_key0 = 0x0D7CBA17,
    write_key1 = 0x0D7CBA17,
    write_key2 = 0x0D7CBA17,
    write_key3 = 0x0D7CBA17,
    header = SDHeaderPositions(
        gain           = 4,
        led            = 5,
        ewl            = 6,
        record_length  = 7,
        fs             = 8,
        delay_start    = 9,
        battery_cutoff = 10
    ),
    config = ConfigPositions(
        width              = 0,
        height             = 1,
        fs                 = 2,
        buffer_size        = 3,
        n_buffers_recorded = 4,
        n_buffers_dropped  = 5
    ),
    buffer = BufferHeaderPositions(
        length               = 0,
        linked_list          = 1,
        frame_num            = 2,
        buffer_count         = 3,
        frame_buffer_count   = 4,
        write_buffer_count   = 5,
        dropped_buffer_count = 6,
        timestamp            = 7,
        data_length          = 8
    )
)
```
````

## IO

Great, that gives us some abstract definition of the format, now to use it!

### Current Status!

The current code was written under duress! Or I mean, in a time crunch! Whatever! It is reproduced here not to shame, but to show where we're starting from!

```python
import numpy as np
from matplotlib import pyplot as plt
import cv2
import time

# Load up Config Sector
f.seek(configSector * sectorSize, 0)  # Move to correct sector
configSectorData = np.fromstring(f.read(sectorSize), dtype=np.uint32)

# Read Data Sectors
saveVideo = True
plotHeaderValues = False
displayVideo = True

frameNum = 0
pixelCount = 0
header = []

if saveVideo is True:
    out = cv2.VideoWriter('WFV4_Mini5_test2.avi', cv2.VideoWriter_fourcc(*'GREY'), 
                        10.0, (configSectorData[CONFIG_BLOCK_WIDTH_POS], configSectorData[CONFIG_BLOCK_HEIGHT_POS] ), 
                        isColor=False)

frame = np.zeros((configSectorData[CONFIG_BLOCK_WIDTH_POS] * configSectorData[CONFIG_BLOCK_HEIGHT_POS], 1), dtype=np.uint8)
f.seek(dataStartSector * sectorSize, 0) # Starting data location
for i in range(configSectorData[CONFIG_BLOCK_NUM_BUFFERS_RECORDED_POS]):
    dataHeader = np.fromstring(f.read(4), dtype=np.uint32) # gets header length
    dataHeader = np.append(dataHeader, np.fromstring(f.read((dataHeader[BUFFER_HEADER_HEADER_LENGTH_POS] - 1) * 4), dtype=np.uint32))

    header.append(dataHeader)
    
    numBlocks = int((dataHeader[BUFFER_HEADER_DATA_LENGTH_POS] + (dataHeader[BUFFER_HEADER_HEADER_LENGTH_POS] * 4) + (512 - 1)) / 512)
    
    data = np.fromstring(f.read(numBlocks*512 - dataHeader[BUFFER_HEADER_HEADER_LENGTH_POS] * 4), dtype=np.uint8)

    # -------------------------------------
    if (dataHeader[BUFFER_HEADER_FRAME_BUFFER_COUNT_POS] == 0):
        # First buffer of a frame
        
        if saveVideo is True:
            out.write(np.reshape(frame, (configSectorData[CONFIG_BLOCK_WIDTH_POS], configSectorData[CONFIG_BLOCK_HEIGHT_POS] )))
        
        if displayVideo is True:
            cv2.imshow('Video', np.reshape(frame, (configSectorData[CONFIG_BLOCK_WIDTH_POS], configSectorData[CONFIG_BLOCK_HEIGHT_POS] )))
            cv2.waitKey(20)
            
        frame[0:dataHeader[BUFFER_HEADER_DATA_LENGTH_POS], 0] = data
        pixelCount = dataHeader[BUFFER_HEADER_DATA_LENGTH_POS]
        frameNum = dataHeader[BUFFER_HEADER_FRAME_NUM_POS]
    else:
        # All other buffers of a frame
        # startIdx = dataHeader[BUFFER_HEADER_FRAME_BUFFER_COUNT_POS] * 50 * 512
        frame[pixelCount:(pixelCount + dataHeader[BUFFER_HEADER_DATA_LENGTH_POS]), 0] = data[:dataHeader[BUFFER_HEADER_DATA_LENGTH_POS]]
        pixelCount = pixelCount + dataHeader[BUFFER_HEADER_DATA_LENGTH_POS]

if saveVideo is True:            
    out.release()

if displayVideo is True:
    cv2.destroyWindow('Video')
    
if plotHeaderValues is True:
    temp = np.asarray(header)
    plt.plot(temp)
```

Some things we would like to improve on!

- **Hard to read!** The same variables are accessed multiple times with long indexing variable names
- **Brittle!** Some values are hardcoded (eg. the `word_size` of `4` and the `sector_size` of `512`) redundantly, meaning that if we change the configuration file at the top, the reading code wouldn't actually use it! Our configuration model is an *interface* to our reader, and if the reader doesn't respond to the inputs of the interface, then it doesn't work!
- **Monolithic!** It's one big loop, so we can't write tests for it or reuse any of it elsewhere! Multiple different kinds of code are mixed together: we can save a video and plot frames as we gather them. This logic is embedded within the I/O code, so if we want to add any additional functionality to our video or plotting functions, we need to expand the loop, making it more difficult to maintain!
- **Dangerous!** - File handling is outside of a context manager - Since this is precious data, we don't want to corrupt our drive! This is less of a problem in `rb` mode than it would be in a write mode, but we want to be sure that we *always* close the file after we are done using it.

You get the idea!

### Separate I/O Class

- using context managers to open file descriptors
- TO BE HONEST IT GOT VERY LATE WHILE I WAS WORKING ON THE CODE AND I WILL FINISH THE TUTORIAL LATER




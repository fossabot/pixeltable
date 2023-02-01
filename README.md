# Pixeltable

Pixeltable presents a dataframe-like interface to image and video data.

## Installation

1. Install Postgres

    On MacOS, [postgresapp.com](postgresapp.com) is a convenient way to do that.

2. `pip install pixeltable`

3. Install additional dependencies
   - Install PyTorch (required for CLIP): see [here](https://pytorch.org/get-started/locally/).
   - Install CLIP from [here](https://github.com/openai/CLIP).
   - If you want to work with videos, you also need to install `ffmpeg`.

## Setup

Pixeltable requires a home directory and a Postgres database, and both are created automatically the
first time you create a Pixeltable client (see below). The location of the home directory is
`~/.pixeltable` (or the value of the `PIXELTABLE_HOME` environment variable); the name of the
Postgres database is `pixeltable` (or the value of the `PIXELTABLE_DB` environment variable).

## Overview

Import convention:
```
import pixeltable as pt
```

### Creating a client
```
cl = pt.Client()
```

### Creating a database
```
db = cl.create_db('db1')
```

### Creating a table with video data
```
c1 = pt.Column('video', pt.VideoType())
c2 = pt.Column('frame_idx', pt.IntType())
c3 = pt.Column('frame', pt.ImageType())
t = db.create_table(
    'video_table', [c1, c2, c3],
    extract_frames_from='video',
    extracted_frame_col='frame',
    extracted_frame_idx_col='frame_idx',
    extracted_fps=1)
```

### Querying a table

|Action|Code|
|----|----|
| Look at 10 rows | `t.show(10)` |
| Look at all rows | `t.show(0)` |
| Look at row for frame 15 | `t[t.frame_idx == 15].show()` |
| Look at rows before index 15 | `t[t.frame_idx < 15].show(0)` |
| Look at rows before index 15 with RGB frames | `t[(t.frame_idx < 15) & (t.frame.mode == 'RGB')].show(0)` |

### Selecting and transforming columns

|Action|Code|
|----|----|
| Only retrieve the frame index and frame | `t[t.frame_idx, t.frame].show()` |
| Look at frames rotated 90 degrees | `t[t.frame.rotate(90)].show()` |
| Overlay frame with itself rotated 90 degrees | `t[pt.functions.blend(t.frame, t.frame.rotate(90))].show()` |

### Attributes and methods on image data

Images are currently represented as `PIL.Image.Image` instances in memory and support a lot of the
attributes and methods documented
[here](https://pillow.readthedocs.io/en/stable/reference/Image.html#PIL.Image.Image).

Available attributes are: `mode`, `height`, `width`.

Available methods are: `convert`, `crop`, `effect_spread`, `entropy`, `filter`, `getbands`, `getbbox`,
`getchannel`, `getcolors`, `getextrema`, `getpalette`, `getpixel`, `getprojection`, `histogram`,
`point`, `quantize`, `reduce`, `remap_palette`, `resize`, `rotate`, `transform`, `transpose`.

Methods can be chained, for example: `t.frame.resize((224, 224)).rotate(90).convert('L')`

### Computed columns

The values in a computed column are automatically filled when data is added:
```
t.add_column(pt.Column('frame_idx_plus_1', computed_with=(t.frame_idx + 1)))
```

### Functions

Functions can be used to transform data, both during querying as well as when data is added to a table.
```
add1 = pt.Function(return_type=pt.IntType(), param_types=[pt.IntType()], eval_fn=lambda x: x + 1)
```

For querying: `t[t.frame_idx, add1(t.frame_idx)].show()`

As a computed column: `t.add_column(pt.Column('c', computed_with=add1(t.frame_idx)))`

### Image similarity search

In order to enable similarity on specific image columns, create those columns with `indexed=True`.
This will compute an embedding for every image and store it in a vector index.
```
c3 = pt.Column('frame', pt.ImageType(), indexed=True)
```

Assuming `img = PIL.Image.open(...)`

Similarity search: `t[t.frame.nearest(img)].show(10)`

Keyword search: `t[t.frame.matches('car')].show(10)`

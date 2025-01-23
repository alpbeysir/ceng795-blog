### Minecraft World

We were in dire need of some things to render other than single blocks. We decided to use Minecraft worlds. 
Minecraft is a very popular voxel-based game with procedural world generation. Minecraft's world format is well-documented and easy to parse with
the help of some open source libraries. 

#### World Format

The world is composed of `.mca` files on disk.
In the `.xml` format `Chunk` tag, a list of region file paths may be provided alongside the usual `Block`'s:

```xml
<Objects>
    <Chunk>
        <Region>big/r.-1.-1.mca</Region>
        <Region>big/r.-1.1.mca</Region>
        <Region>big/r.-1.3.mca</Region>
    </Chunk>
</Objects>
```

- `.mca` files store one region.
- Regions store an area of `32x32` chunks.
- Chunks store `256` chunk sections, some of which may be unused/null. These are stacked vertically.
- Chunk sections store `16x16x16` voxels.

Simple formulas are used to get a voxel's position based on the coordinates of its region, chunk and chunks section.
The code below loads these files to memory (simplified to highlight important parts):

```cpp
FILE* fp = fopen(region_file_path.string().c_str(), "rb");
enkiRegionFile region_file = enkiRegionFileLoad(fp);
for (int i = 0; i < 1024; i++) { // we iterate each chunk in the region
    enkiChunkBlockData chunk = enkiNBTReadChunk(&stream);
    for (int j = 0; j < 256; j++) { // for each chunk section
        if (chunk.sections[j] == nullptr)
            continue;

        const auto palette = chunk.palette[j]; // for getting block type

        for (int x = 0; x < 16; x++) {
            for (int y = 0; y < 16; y++) {
                for (int z = 0; z < 16; z++) {
                    let block = Block(block_pos, 0, scene_mat_index); // convert to internal block format
                    terrain.blocks.emplace_back(block);
                }
            }
        }
    }
    enkiNBTFreeAllocations(&stream);
}
enkiRegionFileFreeAllocations(&region_file);
fclose(fp);
```

Each chunk also has a palette. This basically defines which voxels are mapped to which materials. This will later be useful in the materials section.

#### Materials & Textures

To get the voxels to look as they do in-game, we needed to extract the original texture data from the game. This is stored as JSON metadata and PNG images.
Since these are very game specific, I will choose to not go into much detail. Briefly, the metadata is keyed by tags like `minecraft:stone`, `minecraft:grass_block`. These tags can be extracted from the aforementioned chunk palette. For each metadata key, a standard diffuse material is generated.
The JSON file corresponding to the tag is used to read the correct `.png` files. Because **different sides can have different textures**, multiple images may
be assigned to the same material. 

Since the voxels don't have a true UV definition, a technique similar to triplanar rendering is used.

Using the voxel normal from the hit point data, we can determine which side of the voxel is hit:

```cpp
TextureDirection dir;
if (images.contains(TextureDirection::All)) { // exception for voxels that have all sides the same
    dir = TextureDirection::All;
}
else if (normal.y < 0.0f) {
    dir = TextureDirection::Bottom;
}
else if (normal.y > 0.0f) {
    dir = TextureDirection::Top;
}
else {
    dir = TextureDirection::Side;
}
const auto image = images.at(dir).get(); // determine which image to sample
```

The obtained image is then sampled like usual.

### Performance

| Scene          | Init Time      | Render Time    |
|----------------|----------------|----------------|
| test.xml       | 0.50s          | 3.74s          |
| big_dragon.xml | 23.08s         | 32.98s         |
| closeup.xml    | 19.88s         | 41.18s         |

Initialization time scaled linearly with the number of regions, as expected. Because of the nature of Minecraft's world files, the loading process is a bit inefficient.
`closeup.xml` has 4 regions defined, amounting to a total of `129,010,458` blocks. This takes ~20 seconds to load. 
The efficiency of the SVO algorithm is demonstrated here, even with +100 million voxels the render time is still manageable.

We also ran profiling while rendering the `closeup.xml` scene:

![image](https://github.com/user-attachments/assets/a5930521-7570-42f7-a80a-eece17925ad9)

The graph shows how much time each function took to execute at each recursion depth. 
The first huge hump is the SVO ray intersection function, the second is the SVO construction and the rest is the world loading.
A keen observer will notice the `log8` scaling of the SVO functions.









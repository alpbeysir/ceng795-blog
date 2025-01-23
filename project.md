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

#### Materials & Textures


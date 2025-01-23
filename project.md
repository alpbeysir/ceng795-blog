### Minecraft World

We were in dire need of some things to render other than single blocks. We decided to use Minecraft worlds. 
Minecraft is a very popular voxel-based game with procedural world generation. Minecraft's world format is well-documented and easy to parse with
the help of some open source libraries. 

#### World Format

The world is composed of `.mca` files on disk.

- `.mca` files store one region.
- Regions store an area of `32x32` chunks.
- Chunks store `256` chunk sections, some of which may be unused/null. These are stacked vertically.
- Chunk sections store `16x16x16` voxels.

Simple formulas are used to get a voxel's position based on the coordinates of its region, chunk and chunks section.

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

#### Materials & Textures


# CyberNoise Terrain Generation SDK


## Module Description

Our module supports generating a 2D top-down terrain generation via using gradient noise. This terrain will consist of a grid of renderables that are called tiles. These tiles can be any texture to form the look of a terrain. 

A terrain consists of multiple layers to allow for more than one pass in the terrain. For example, the first pass can be the ground (like grass) and the second pass can determine where trees are placed.

Basic functions are used to manipulate or gather information about the terrain. For example, the position (0,0) can contain a tile that is user defined as “Water” and you can manipulate this “Water” tile to instead become “Ice” using the API.

## API
### Scripts:

**terrain/tile.js:**
  - Loads tile texture, generates sprite renderable,and defines tile name/type
  
**terrain/layer.js:**
  - Generates a terrain layer. Holds all the renderables for a given layer and draws them.
  
**terrain/terrain.js:**
  - Holds all terrain layers and draws them. Has high level functions for interacting with layers like figuring out what tile a position is in.
  
**terrain/terrain_settings.js:**
  - Holds values for what settings should be used for terrain generation like scale or offset.
  
**utils/noise.js**
  - Gradient noise used in terrain generation. This helper class will take a terrain settings and use its scale/offset values to generate normalized floats which are used for generation.

### Tile

```javascript
 /**
  * Creates a new tile
  *
  * @param {String} name name of tile, used for identification
  * @param {String} textureName file path used for texture
  * @param {vec2} range the range at which the tile will appear (e.g. 0.2 < x < 0.4)
  * @param {Number[4]} setElemPositions is for selecting the right image in a texture sheet. Uses engine implementation of sprite renderable to do this
  */
 constructor(name, textureName, range, setElemPositions)
 
 /**
  * Makes a new sprite renderable at the position with a size based off the tile
  *
  * @param {vec2} position to place tile
  * @param {vec2} size of tile
  * @returns {SpriteRenderable} a new sprite renderable
  */
 makeRenderable(position, size)
 
 /** Loads the texture */  
 load()
 
 /** Unloads the texture */  
 unload()
 
 /** @returns {String} name of tile */  
 getName()
  
 /** @returns {String} name of texture file path. Texture name */  
 getTextureName()
  
 /** @returns {vec2} range at which the tile will appear */  
 getRange()
```

### Layer

```javascript
 /**
  * Layer constructor expects tiles to be loaded. Offset is set afterwards but before generation
  *
  * @param {Tile[]} tiles to be used for layer
  * @param {vec2} size in WC of the entire layer (WxH)
  * @param {vec2} scale value of each tile in WC
  * @param {(Bool *function(Layer[], Noise, Vec2))[]} conditions List of functions for validating tile placement. Null if none
  * @param {TerrainSettings} terrainSettings for current layer
  * @param {Layer[]} otherLayers previous layers that were already generated. This is for conditions if a tile placement depends on previous layers
  */
 constructor(tiles, size, scale, conditions, terrainSettings,otherLayers)
 
 /**
  * Generates the renderables for the layer with the given noise
  *
  * @param {Noise} noise used for determining tile placement
  */
 generateLayer(noise)
 
 /** @param {vec2} position to offset the layer by*/
 setOffset(position)
 
 /** @returns {TerrainSettings} terrain settings of layer*/
 getTerrainSettings()
 
 /** @param {TerrainSettings} terrainSettings new settings*/
 setNewTerrainSettings(terrainSettings)
```

### Terrain

```javascript
 /** 
  * Constructs a new terrain, expects the layers to be set later
  * @param {int} how far the terrain has zoomed in/out
  */
 constructor(zoomLevel)
 
 /** @param {Layer[]} layers array that the terrain draws */
 setLayers(layers)
 
 /**
  * Sets the terrain object and all its layers at the given position
  *
  * @param {vec2} position to place the terrain
  * @param {Noise} noise for regenerating the terrain
  */
 setPosition(position, noise)
 
 /**
  * Deletes the tile, with the given name, at the location
  *
  * @param {vec2} position the position to delete the tile at
  * @param {String} name of the new tile (to delete the correct tile in the correct layer)
  * @returns {Bool} true if successful
  */
 deleteTileAtLocation(position, name)
 
 /**
  * Sets the tile at the location for the layer
  *
  * @param {vec2} position the position to set the tile at
  * @param {String} name of the new tile
  * @param {Number} index of the layer that is being modified
  */
 setTileAtLocation(position, name, index)
 
 /**
  * Grabs the name of all tiles at the location
  *
  * @param {Number} position the position to extract the names from
  * @returns {String[]} an array of strings containing all the tile names at the location. Null if no tile found at the location
  */
 getTileNamesAtLocation(position)
 
 /**
  * Zooms in/out the terrain at the given point (depending on the direction)
  *
  * @param {vec2} center the point to zoom in/out
  * @param {Number} zoom the factor to zoom in/out
  * @param {Number} direction -1 to zoom out, 1 to zoom in
  */
 zoom(center, zoom, direction)
 
 /**
  * Recreates the terrain with a new noise seed from a noise object
  *
  * @param {Noise} noise for regenerating the terrain
  */
 regenerateLayers(noise)
 
 /**
  * Draws all the layers for the given camera
  *
  * @param {Camera} camera to draw the terrain in
  */
 draw(camera)
```

### Terrain Settings

```javascript
 /**
  * Constructs a new terrain settings with the given fields
  *
  * @param {vec2} offset of the terrain. Like moving the terrain
  * @param {Number} scale scales the terrain to be larger/smaller
  * @param {Number} roughnessLayer how many layers for generating roughness
  * @param {Number} roughnessBlend how the layers blend between each other, each corresponding layer will blend roughnessBlend times less
  */
 constructor(offset, scale, roughnessLayer, roughnessBlend)
 
 /**
  * Copy constructor
  *
  * @param {TerrainSettings} terrainSettings old terrain settings
  */
 copy(terrainSettings)
 
 /** @param {Number} scale of our terrain settings */
 setScale(scale)
 
 /** @param {Number} delta multiplies current scale by this */
 multScale(delta)
 
 /** @param {Number} delta adds current scale by this */
 addScale(delta)
 
 /** @param {vec2} offset of our terrain settings */
 setOffset(offset)
 
 /** @param {vec2} delta adds current offset by this */
 addOffset(delta)
 
 /** @returns {vec2} terrain settings offset */
 getOffset()
 
 /** @returns {Number} terrain settings scale */
 getScale()
 /** @returns {Number} terrain settings roughnessLayer */
 getRoughessLayer()
 
 /** @returns {Number} terrain settings roughnessBlend */
 getRoughnessBlend()
```

### Noise

```javascript
 /**
  * Permutates a new gradient noise with a given seed
  *
  * @param {Number} seed for noise. Same seed gives same noise values at positions
  */
 constructor(seed)
 
 /**
  * @param {vec2} position in the current noise
  * @param {TerrainSettings} settings for noise. Can be any object that has similar fields
  * @returns {Number} a normalized float while considering octaves, persistence, offset, and scale
   */
 getOctaveVal(position, settings)
 
 /**
  * @param {vec2} position in the current noise
  * @param {TerrainSettings} settings for noise. Can be any object that has similar fields
  * @returns {Number} a normalized float while considering offset, and scale
  */
 getValue(position, settings) {
```

## Demo
[https://aj213.github.io/452CyberNoise/](https://aj213.github.io/452CyberNoise/)


## Conclusion
**Strengths:**
 - Customizability: the API allows you to make multiple passes for terrain generation and allows you to implement your own functions for semantic checking of tile placement.
 - Generalized: Can support simple games wanting to make static terrain as a background, and also support a voxel engine making infinite terrain that can be edited.

**Weaknesses:**
 - Verbose: Creating layers and tiles takes up multiple lines and accepts dozens of parameters in total. Such customization and control comes at the expense of simplicity. 

**Improvements:**
 - General Performance of terrain generation.
 - Not every tile needs 2 triangles, an entire layer could consist of 2 triangles which would be much more performant.
 - Easier texture selection. Current sprite renderable implementation is difficult to use with tile sets with consistent width/height. 

## Welcome to GitHub Pages

You can use the [editor on GitHub](https://github.com/CyberVA/CyberNoiseSDKWebsite/edit/gh-pages/index.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [Basic writing and formatting syntax](https://docs.github.com/en/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/CyberVA/CyberNoiseSDKWebsite/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.

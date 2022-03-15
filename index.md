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
 constructor(name, textureName, range, setElemPositions) {
       this.mName = name;
       this.mTextureName = textureName;
       this.mRange = range; // Vec2
       this.mSetElemPositions = setElemPositions; // int[4]
 }
 
 /**
  * Makes a new sprite renderable at the position with a size based off the tile
  *
  * @param {vec2} position to place tile
  * @param {vec2} size of tile
  * @returns {SpriteRenderable} a new sprite renderable
  */
 makeRenderable(position, size){
       let sprite = new engine.SpriteRenderable(this.mTextureName);
       sprite.setElementPixelPositions(this.mSetElemPositions[0], this.mSetElemPositions[1], this.mSetElemPositions[2], this.mSetElemPositions[3]);
       sprite.getXform().setPosition(position[0], position[1]);
       sprite.getXform().setSize(size[0], size[1]);
       return sprite;
 }
 
 /** Loads the texture */  
 load() { engine.texture.load(this.mTextureName); }
 
 /** Unloads the texture */  
 unload() { engine.texture.unload(this.mTextureName); }
 
 /** @returns {String} name of tile */  
 getName() { return this.mName; }
  
 /** @returns {String} name of texture file path. Texture name */  
 getTextureName() { return this.mTextureName; }
  
 /** @returns {vec2} range at which the tile will appear */  
 getRange() { return this.mRange; }
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
 constructor(tiles, size, scale, conditions, terrainSettings, otherLayers) {
        this.mTiles = tiles;
        this.mSize = size;
        this.mScale = scale;
        this.mConditions = conditions;
        this.mTerrainSettings = terrainSettings;
        this.mOtherLayers = otherLayers;
        
        this.mPlacedTiles = {};
        this.mOffset = vec2.fromValues(0, 0);
  }
 
 /**
  * Generates the renderables for the layer with the given noise
  *
  * @param {Noise} noise used for determining tile placement
  */
 generateLayer(noise){
      this.mPlacedTiles = {};
      for(let x = 0; x < this.mSize[0]; x += this.mScale[0]){
          for(let y = 0; y < this.mSize[1]; y += this.mScale[1]){
              let currentPosition = vec2.fromValues(x, y);
              if(!this.shouldPlace(currentPosition, noise)){
                  continue;
              }
              let height = noise.getOctaveVal(vec2.fromValues(currentPosition[0] + this.mOffset[0],currentPosition[1] + this.mOffset[1]), this.mTerrainSettings);
              let tile = this.getTileTypeByHeight(height);
              if(tile == null){
                  continue;
              }
              // Configure renderable to be a the right position and scale
              let pos = vec2.fromValues(x + this.mOffset[0] + this.mScale[0]/2, y + this.mOffset[1] + this.mScale[1]/2);
              let size = vec2.fromValues(this.mScale[0], this.mScale[1]);
              let renderable = tile.makeRenderable(pos, size);
              
              let tileLocation = this.getTileLocation(pos);
              this.mPlacedTiles[tileLocation] = {renderable: renderable, name: tile.getName()};
          }
      }
  }
 
 /** @param {vec2} position to offset the layer by*/
 setOffset(position) { this.mOffset = position; }
 
 /** @returns {TerrainSettings} terrain settings of layer*/
 getTerrainSettings() { return this.mTerrainSettings; }
 
 /** @param {TerrainSettings} terrainSettings new settings*/
 setNewTerrainSettings(terrainSettings){ this.mTerrainSettings = terrainSettings; }
```

### Terrain

```javascript
 /** 
  * Constructs a new terrain, expects the layers to be set later
  * @param {int} how far the terrain has zoomed in/out
  */
 constructor(zoomValue = 0) { 
      this.mLayers = null;
      this.mCurrentZoom = zoomValue;
  }
 
 /** @param {Layer[]} layers array that the terrain draws */
 setLayers(layers) { this.mLayers = layers; }
 
 /**
  * Sets the terrain object and all its layers at the given position
  *
  * @param {vec2} position to place the terrain
  * @param {Noise} noise for regenerating the terrain
  */
 setPosition(position, noise){
      for(let i = 0; i < this.mLayers.length; i++) {
          this.mLayers[i].setOffset(position);
          this.mLayers[i].generateLayer(noise);
      }
  }
 
 /**
  * Deletes the tile, with the given name, at the location
  *
  * @param {vec2} position the position to delete the tile at
  * @param {String} name of the new tile (to delete the correct tile in the correct layer)
  * @returns {Bool} true if successful
  */
 deleteTileAtLocation(position, name){
      for(let i = 0; i < this.mLayers.length; i++){
          let curName = this.mLayers[i].getTileNameAtLocation(position);
          if(curName == name){
              this.mLayers[i].deleteTileAtLocation(position);
          }
      }
  }
 
 /**
  * Sets the tile at the location for the layer
  *
  * @param {vec2} position the position to set the tile at
  * @param {String} name of the new tile
  * @param {Number} index of the layer that is being modified
  */
 setTileAtLocation(position, name, index){
     this.mLayers[index].setTileAtLocation(position, name);
 }
 
 /**
  * Grabs the name of all tiles at the location
  *
  * @param {Number} position the position to extract the names from
  * @returns {String[]} an array of strings containing all the tile names at the location. Null if no tile found at the location
  */
 getTileNamesAtLocation(position){
      let tileNames = [];
      for(let i = 0; i < this.mLayers.length; i++){
          tileNames.push(this.mLayers[i].getTileNameAtLocation(position));
      }
      return tileNames;
  }
 
 /**
  * Zooms in/out the terrain at the given point (depending on the direction)
  *
  * @param {vec2} center the point to zoom in/out
  * @param {Number} zoom the factor to zoom in/out
  * @param {Number} direction -1 to zoom out, 1 to zoom in
  */
 zoom(center, zoom, direction) {
      let zoomValue = direction > 0 ? 1.0/zoom : zoom;
        
      this.mCurrentZoom += direction;
      for(let i = 0; i < this.mLayers.length; i++){
          let newOffset = vec2.fromValues(
              center[0]*Math.pow(zoom,this.mCurrentZoom)-center[0],
              center[1]*Math.pow(zoom,this.mCurrentZoom)-center[1]
          );
          this.mCurrentOffset = newOffset;
          this.mLayers[i].getTerrainSettings().setOffset(newOffset);
          this.mLayers[i].getTerrainSettings().multScale(zoomValue);
      }
  }
 
 /**
  * Recreates the terrain with a new noise seed from a noise object
  *
  * @param {Noise} noise for regenerating the terrain
  */
 regenerateLayers(noise){
      for(let i = 0; i < this.mLayers.length; i++){
          this.mLayers[i].generateLayer(noise);
      }
  }
 
 /**
  * Draws all the layers for the given camera
  *
  * @param {Camera} camera to draw the terrain in
  */
 draw(camera){
      for(let i = 0; i < this.mLayers.length; i++){
          if(this.mLayers[i] == null){
              continue;
          }
          this.mLayers[i].draw(camera);
      }
  }
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
 constructor(offset = vec2.fromValues(0, 0), scale = 0, roughnessLayer = 0, roughnessBlend = 0) {
    this.mOffset = offset;
    this.mScale = scale;
    this.mRoughnessLayer = roughnessLayer;
    this.mRoughnessBlend = roughnessBlend;
 }
 
 /**
  * Copy constructor
  *
  * @param {TerrainSettings} terrainSettings old terrain settings
  */
 copy(terrainSettings) {
    this.mOffset = terrainSettings.getOffset();
    this.mScale = terrainSettings.getScale();
    this.mRoughnessLayer = terrainSettings.getRoughessLayer();
    this.mRoughnessBlend = terrainSettings.getRoughnessBlend();
 }
 
 /** @param {Number} scale of our terrain settings */
 setScale(scale) { this.mScale = scale; }
 
 /** @param {Number} delta multiplies current scale by this */
 multScale(delta) { this.mScale *= delta; }
 
 /** @param {Number} delta adds current scale by this */
 addScale(delta) { this.mScale += delta; }
 
 /** @param {vec2} offset of our terrain settings */
 setOffset(offset) { this.mOffset = offset; }
 
 /** @param {vec2} delta adds current offset by this */
 addOffset(delta) { this.mOffset = vec2.fromValues(this.mOffset[0] + delta[0], this.mOffset[1] + delta[1]); }
 
 /** @returns {vec2} terrain settings offset */
 getOffset() { return this.mOffset; }
 
 /** @returns {Number} terrain settings scale */
 getScale() { return this.mScale; }
 /** @returns {Number} terrain settings roughnessLayer */
 getRoughessLayer() { return this.mRoughnessLayer; }
 
 /** @returns {Number} terrain settings roughnessBlend */
 getRoughnessBlend() { return this.mRoughnessBlend; }
```

### Noise
Credit to https://github.com/WardBenjamin/SimplexNoise/blob/master/SimplexNoise/Noise.cs for the provided simplex noise algorithm
```javascript
 /**
  * Permutates a new gradient noise with a given seed
  *
  * @param {Number} seed for noise. Same seed gives same noise values at positions
  */
 constructor(seed) {
      this.mSeed = seed;
      this._perm = new Array(
        151,160,137,91,90,15,
        131,13,201,95,96,53,194,233,7,225,140,36,103,30,69,142,8,99,37,240,21,10,23,
        190, 6,148,247,120,234,75,0,26,197,62,94,252,219,203,117,35,11,32,57,177,33,
        88,237,149,56,87,174,20,125,136,171,168, 68,175,74,165,71,134,139,48,27,166,
        77,146,158,231,83,111,229,122,60,211,133,230,220,105,92,41,55,46,245,40,244,
        102,143,54, 65,25,63,161, 1,216,80,73,209,76,132,187,208, 89,18,169,200,196,
        135,130,116,188,159,86,164,100,109,198,173,186, 3,64,52,217,226,250,124,123,
        5,202,38,147,118,126,255,82,85,212,207,206,59,227,47,16,58,17,182,189,28,42,
        223,183,170,213,119,248,152, 2,44,154,163, 70,221,153,101,155,167, 43,172,9,
        129,22,39,253, 19,98,108,110,79,113,224,232,178,185, 112,104,218,246,97,228,
        251,34,242,193,238,210,144,12,191,179,162,241, 81,51,145,235,249,14,239,107,
        49,192,214, 31,181,199,106,157,184, 84,204,176,115,121,50,45,127, 4,150,254,
        138,236,205,93,222,114,67,29,24,72,243,141,128,195,78,66,215,61,156,180,
        151,160,137,91,90,15,
        131,13,201,95,96,53,194,233,7,225,140,36,103,30,69,142,8,99,37,240,21,10,23,
        190, 6,148,247,120,234,75,0,26,197,62,94,252,219,203,117,35,11,32,57,177,33,
        88,237,149,56,87,174,20,125,136,171,168, 68,175,74,165,71,134,139,48,27,166,
        77,146,158,231,83,111,229,122,60,211,133,230,220,105,92,41,55,46,245,40,244,
        102,143,54, 65,25,63,161, 1,216,80,73,209,76,132,187,208, 89,18,169,200,196,
        135,130,116,188,159,86,164,100,109,198,173,186, 3,64,52,217,226,250,124,123,
        5,202,38,147,118,126,255,82,85,212,207,206,59,227,47,16,58,17,182,189,28,42,
        223,183,170,213,119,248,152, 2,44,154,163, 70,221,153,101,155,167, 43,172,9,
        129,22,39,253, 19,98,108,110,79,113,224,232,178,185, 112,104,218,246,97,228,
        251,34,242,193,238,210,144,12,191,179,162,241, 81,51,145,235,249,14,239,107,
        49,192,214, 31,181,199,106,157,184, 84,204,176,115,121,50,45,127, 4,150,254,
        138,236,205,93,222,114,67,29,24,72,243,141,128,195,78,66,215,61,156,180 
     );
   }
   
  /**
   * Generates more detail onto a noise function via multiple iterations
   * @param {*} x x position
   * @param {*} y y position
   * @param {*} octaves level of detail that we generate on to the noise
   * @param {*} persistence how heavy the blend between noise generations we apply
   * @returns value ranging from 0 to 1
   */
	octaveSimplex(x, y, octaves, persistence) {
		let total = 0;
		let frequency = 1;
        let amplitude = 1;
        let maxValue = 0;
		for(let i=0;i<octaves;i++) {
			total += this.simplexNoise(x * frequency, y * frequency) * amplitude;
			
            maxValue += amplitude;

			amplitude *= persistence;
			frequency *= 2;
		}
		return total / maxValue;
	}
 
 /**
  * @param {vec2} position in the current noise
  * @param {TerrainSettings} settings for noise. Can be any object that has similar fields
  * @returns {Number} a normalized float while considering octaves, persistence, offset, and scale
   */
 getOctaveVal(position, settings) {
     let x = (position[0]  + settings.getOffset()[0]) * settings.getScale() + this.mSeed;
     let y = (position[1]  + settings.getOffset()[1]) * settings.getScale() + this.mSeed;
     return (this.octaveSimplex(x, y, settings.getRoughessLayer(), settings.getRoughnessBlend()) + 1)/2.0;
 }
 
 /**
  * @param {vec2} position in the current noise
  * @param {TerrainSettings} settings for noise. Can be any object that has similar fields
  * @returns {Number} a normalized float while considering offset, and scale
  */
 getValue(position, settings) {
     let x = (position[0]  + settings.getOffset()[0]) * settings.getScale() + this.mSeed;
     let y = (position[1]  + settings.getOffset()[1]) * settings.getScale() + this.mSeed;
     return this.simplexNoise(x, y);
 }
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

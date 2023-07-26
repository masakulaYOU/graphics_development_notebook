# Cesium地形数据解析

Cesium地形数据采用了一种扩展名为`.terrain`的二进制文件结构，该结构为`quantized-mesh-1.0`格式的、简单多重采样的四叉树金字塔结构。每个瓦片的地形数据包含了当前瓦片的顶点信息、三角网信息、索引信息以及扩展信息等。访问每个瓦片的服务地址为：

```
http://example.com/tiles
```

瓦片金字塔的两个根文件的范围及服务地址为
- (-180 deg, -90 deg) - (0 deg, 90 deg) - http://example.com/tiles/0/0/0.terrain
- (0 deg, -90 deg) - (180 deg, 90 deg) - http://example.com/tiles/0/1/0.terrain

这两个根瓦片的八个子瓦片的范围以及服务地址为
- (-180 deg, -90 deg) - (-90 deg, 0 deg) - http://example.com/tiles/1/0/0.terrain
- (-90 deg, -90 deg) - (0 deg, 0 deg) - http://example.com/tiles/1/1/0.terrain
- (0 deg, -90 deg) - (90 deg, 0 deg) - http://example.com/tiles/1/2/0.terrain
- (90 deg, -90 deg) - (180 deg, 0 deg) - http://example.com/tiles/1/3/0.terrain
- (-180 deg, 0 deg) - (-90 deg, 90 deg) - http://example.com/tiles/1/0/1.terrain
- (-90 deg, 0 deg) - (0 deg, 90 deg) - http://example.com/tiles/1/1/1.terrain
- (0 deg, 0 deg) - (90 deg, 90 deg) - http://example.com/tiles/1/2/1.terrain
- (90 deg, 0 deg) - (180 deg, 90 deg) - http://example.com/tiles/1/3/1.terrain

请求瓦片时，要确保在HTTP头文件中包含以下信息
```
Accept: application/vnd.quantized-mesh,application/octet-stream;q=0.9
```

否则可能会返回一个奇怪的瓦片


## 瓦片结构

每个瓦片是一个特别编码的mesh结构，其顶点数据包含了在瓦片边界处另一个瓦片的顶点数据，其目的是为了确保相邻的瓦片在拼接时，边界的顶点保持连续，没有突然变化的情况。

地形瓦片数据是经过gzip压缩的，解压后的瓦片是一种小端格式的二进制数据，即将低序字节存储在起始位置。文件的第一部分是包含了以下格式的头信息。其中，`Doubles`数据为IEEE 754的64位的双精度浮点数，`Floats`数据为IEEE 754的32位的单精度浮点型。

```cpp
struct QuantizedMeshHeader 
{
	// 球心坐标系下的瓦片的中心位置
	double CenterX;
	double CenterY;
	double CenterZ;

	// 整张瓦片覆盖区域的最高高程和最低高程
	// 这么设计的目的是，
	// 防止在地形简化过程中丢失瓦片的高程最大和最小的顶点
	// 同时也是进行分析或者进行可视化的一个参考数据
	float MinimumHeight;
	float maximumHeight;

	// 瓦片的外接球，包括XYZ坐标和球的半径
	// XYZ坐标为球心坐标，半径以米为单位
	double BoundingSphereCenterX;
	double BoundingSphereCenterY;
	double BoundingSphereCenterZ;
	double BoundingSphereRadius;

	// 地平线遮挡点，为椭球尺寸范围内的球心坐标
	// 如果该点低于地平线，则说明整个瓦片都是低于地平线的
	// 用于判断这个瓦片可不可见
	// 详情看http://cesiumjs.org/2013/04/25/Horizon-culling/
	double HorizonOcclusionPointX;
	double HorizonOcclusionPointY;
	double HorizonOcclusionPointZ;
};
```

之后，便是顶点数据的数据头，其中`unsigned int`表示32位无符号整型，`unsigned short`位16位无符号整型。

```cpp
struct VertexData
{
	unsigned int vertexCount;
	unsigned short u[vertexCount];
	unsinged short v[vertexCount];
	unsigned short height[vertexCount];
};
```

`vertexCount`为下面`u`、`v`和`height`数组的长度。三个数组都是用zig-zag方式对数据进行了压缩，以便将其转为小整数。其中，解码方式为
```javascript
var u = 0;
var v = 0;
var height = 0;

function zigZagDecode(value) {
	return (value >> 1) ^ (-(value & 1));
}

for (i = 0; i < vertexCount; ++i) {
	u += zigZagDecode(uBuffer[i]);
	v += zigZagDecode(vBuffer[i]);
	height += zigZagDecode(heightBuffer[i]);

	uBuffer[i] = u;
	vBuffer[i] = v;
	heightBuffer[i] = height;
}
```

解码之后的数组的每个元素的含义如下：
|域|意义|
|:-|:-|
|`u`|瓦片中该顶点的水平坐标。`u`为0时，表示这个瓦片的最西边，`u`为32767时，表示这个瓦片的最东边。对于其他的数值，则是从这个瓦片的最西边的经度到这个瓦片最东边经度的线性插值|
|`v`|瓦片中该顶点的垂直坐标。`v`为0时，表示这个瓦片的最南边，`v`为32767时，表示这个瓦片的最北边。对于其他的数值，则是从这个瓦片的最南边的纬度到这个瓦片最北边纬度的线性插值|
|`height`|瓦片中该顶点的高度。`height`为0时，表示这个点的高程为头信息中的minimumHeight，`height`为32767时，表示这个点的高程为头信息中的maximumHeight。对于其他数值，则是从minimumHeight到maximumHeight的线性插值|

之后跟着的是索引数据。索引指定了这些顶点怎么互相连接，组成三角网。如果该瓦片的顶点数超过的65536，则使用`IndexData32`格式数据来编码索引，否则的话使用`IndexData16`即可。（对应javascript中的结构化数据`Uint32Array`和`Uint16Array`）。

为了使数据的字节数对其，在索引数据之前会填充一些空白，确保以下规则：如果索引类型为`IndexData16`，则保证索引数据之前的总数据量为2字节的整数倍，如果索引类型为`IndexData32`，则保证索引数据之前总数据量为4字节的整数倍。

```cpp
struct IndexData16
{
	unsigned int triangleCount;
	unsigned short indices[triangle * 3];
}

struct IndexData32
{
	unsigned int triangleCount;
	unsigned int indices[triangle * 3];
};
```

索引使用Webgl-loader中的高水位编码算法，其解码方式为

```javascript
var highest = 0;
for (var i = 0; i < indices.length; ++i) {
	var code = indices[i];
	indices[i] = highest - code;
	if (code === 0) {
		++highest;
	}
}
```

每个三角网的三角形顶点以顺时针方式进行排列.

之后，是瓦片四条边上顶点索引，这些索引是用来拼接相邻的瓦片的。

```cpp
struct EdgeIndices16
{
	unsigned int westVertexCount;
	unsigned short westIndices[westVertexCount];

	unsigned int southVertexCount;
	unsigned short southIndices[southVertexCount];

	unsigned int eastVertexCount;
	unsigned short eastIndices[eastVertexCount];

	unsigned int northVertexCount;
	unsigned short northIndices[northVertexCount];
}

struct EdgeIndices32
{
	unsigned int westVertexCount;
	unsigned int westIndices[westVertexCount];

	unsigned int southVertexCount;
	unsigned int southIndices[southVertexCount];

	unsigned int eastVertexCount;
	unsigned int eastIndices[eastVertexCount];

	unsigned int northVertexCount;
	unsigned int northIndices[northVertexCount];
}
```

## 组织规则（Tiling Scheme）和投影

默认的，瓦片数据的排列是根据瓦片地图服务（TMS）和地理坐标系来决定的，在Cesium中使用`projection`投影和`scheme`瓦片规则来指定。

`projection`可用的有`EPSG:3857`（谷歌地图使用的网络墨卡托Web Mercator）投影和`EPSG:4326`，即经纬度表示的全球地理坐标系统WGS-84。两者的不同为，`EPSG:3857`投影在根节点只有1个瓦片，而`EPSG:4326`在根节点有2张瓦片。

对于tilingScheme，有两种选择，`tms`和`slippyMap`。`tms`服务中的瓦片编码的Y是从南往北计算的，而`slippyMap`则是从北往南。即，`tms`服务的瓦片编码原点在地图左下角，而`slippyMap`则在左上角，这部分可以从谷歌瓦片的组织结构中可以看出来。

Cesium的地形瓦片可以在`layer.json`文件中指定，如果不指定，则默认为`EPSG:4326`和`tms`。

## 扩展信息

除了第一部分的信息之后，在文件结尾还可能存在额外的扩展信息。每个额外信息以一个`ExtensionHeader`开始，包含扩展id以及扩展信息，如下，其中`unsigned char`是一个8字节的无符号整型。

```cpp
struct ExtensionHeader
{
	unsigned char extensionId;
	unsigned int extensionLength;
}
```

每有一个扩展都会对应一个扩展id。如果该瓦片中没有扩展信息，那瓦片数据中就不包含`ExtensionHeader`内容。一个瓦片可以存在多个扩展，扩展的顺序由服务端提供。

扩展内容可以通过在HTTP请求头中添加，并通过`-`来连接多个扩展。如
```
Accept: 'application/vnd.quantized-mesh;extensions=octvertexnormals-watermask'
```

地形数据可以定义的扩展有以下内容

### 地形光照

- 扩展名：`Oct-Encoded Per-Vertex Normals`
- 扩展id：`1`

为地形数据添加地形光照属性。每个顶点使用oct编码来压缩xyz坐标，将一个96位（32位x3）的单精度浮点型单位向量转为只有xy的16位数据（8位x2）。`oct`编码见Cigolle 的论文A Survey of Efficient Representations of Independent Unit Vectors，http://jcgt.org/published/0003/02/01/

```cpp
struct OctEncodedVertexNormals
{
	unsigned char xy[vertexCount * 2];
}
```

如果要想请求该扩展，在请求的HTTP头中添加以下内容
```
Accept: 'application/vnd.quantized-mesh;extensions=octvertexnormals'
```

### 水面数据

- 扩展名：`Water Mask`
- 扩展id：`2`

为渲染水面效果添加海岸线数据。水面数据的长度要么是1byte，表示该瓦片全部都是陆地或者全部都是水体，要么是一个$256\times 256\times 1=65535$bytes的既包含陆地，又包含水体的掩码数据。每一位的范围是0或者255，0表示该位置为陆地，255表示该位置为水体，顺序为从北往南、从西往东。第一个字节的数据表示瓦片西北角的水面掩码值。数值也可以在0到255之间，用于表示反走样的海岸线数据。

```cpp
struct WaterMask
{
	unsigned char mask;
}
// 或者是
struct WaterMash
{
	unsigned char mask[256 * 256];
}
```

请求水面掩码数据需要在HTTP请求头中添加如下内容

```
Accept: 'application/vnd.quantized-mesh;extensions=watermask'
```

### 元数据

- 扩展名：`Metadata`
- 扩展id：`4`

元数据是一个JSON对象表示的、用于存储瓦片额外信息的数据。可能包含的数据如土地类型、子瓦片是否可见等。

```cpp
struct Metadata
{
	unsinged int jsonLength;
	char json[jsonLength];
}
```

如果要请求元数据，则需要在HTTP请求头中添加如下内容

```
Accept: 'application/vnd.quantized-mesh;extensions=metadata'
```

## Cesium解析数据源码

Cesium中解析quantized-mesh-1.0格式的地形的源码主要涉及以下几个函数。

首先，在`CesiumTerrainProvider.js`中，`CesiumTerrainProvider`类有一个实例方法，`CesiumTerrainProvider.prototype.requestTileGeometry`，该函数主要来确定使用哪一个地形服务，从添加的所有地形服务中找到第一个能够为当前瓦片提供地形的服务，然后调用外部函数`requestTileGeometery`。该部分源码如下：
```javascript
CesiumTerrainProvider.prototype.requestTileGeometry = function (
  x,
  y,
  level,
  request
) {
  //>>includeStart('debug', pragmas.debug)
  if (!this._ready) {
    throw new DeveloperError(
      "requestTileGeometry must not be called before the terrain provider is ready."
    );
  }
  //>>includeEnd('debug');

  const layers = this._layers;
  let layerToUse;
  const layerCount = layers.length;

  if (layerCount === 1) {
    // Optimized path for single layers
    layerToUse = layers[0];
  } else {
    for (let i = 0; i < layerCount; ++i) {
      const layer = layers[i];
      if (
        !defined(layer.availability) ||
        layer.availability.isTileAvailable(level, x, y)
      ) {
        layerToUse = layer;
        break;
      }
    }
  }

  return requestTileGeometry(this, x, y, level, layerToUse, request);
};
```

在`requestTileGeometry`函数中，主要用于拼接请求头，决定请求的地址，以及是否请求地形光照、水面掩码、元数据，然后拼接请求字符串，发送请求。其主要代码如下
```javascript
function requestTileGeometry(provider, x, y, level, layerToUse, request) {
  if (!defined(layerToUse)) {
    return Promise.reject(new RuntimeError("Terrain tile doesn't exist"));
  }

  const urlTemplates = layerToUse.tileUrlTemplates;
  if (urlTemplates.length === 0) {
    return undefined;
  }

  // The TileMapService scheme counts from the bottom left
  let terrainY;
  if (!provider._scheme || provider._scheme === "tms") {
    const yTiles = provider._tilingScheme.getNumberOfYTilesAtLevel(level);
    terrainY = yTiles - y - 1;
  } else {
    terrainY = y;
  }

  const extensionList = [];
  if (provider._requestVertexNormals && layerToUse.hasVertexNormals) {
    extensionList.push(
      layerToUse.littleEndianExtensionSize
        ? "octvertexnormals"
        : "vertexnormals"
    );
  }
  if (provider._requestWaterMask && layerToUse.hasWaterMask) {
    extensionList.push("watermask");
  }
  if (provider._requestMetadata && layerToUse.hasMetadata) {
    extensionList.push("metadata");
  }

  let headers;
  let query;
  const url = urlTemplates[(x + terrainY + level) % urlTemplates.length];

  const resource = layerToUse.resource;
  if (
    defined(resource._ionEndpoint) &&
    !defined(resource._ionEndpoint.externalType)
  ) {
    // ion uses query paremeters to request extensions
    if (extensionList.length !== 0) {
      query = { extensions: extensionList.join("-") };
    }
    headers = getRequestHeader(undefined);
  } else {
    //All other terrain servers
    headers = getRequestHeader(extensionList);
  }

  const promise = resource
    .getDerivedResource({
      url: url,
      templateValues: {
        version: layerToUse.version,
        z: level,
        x: x,
        y: terrainY,
      },
      queryParameters: query,
      headers: headers,
      request: request,
    })
    .fetchArrayBuffer();

  if (!defined(promise)) {
    return undefined;
  }

  return promise.then(function (buffer) {
    if (defined(provider._heightmapStructure)) {
      return createHeightmapTerrainData(provider, buffer, level, x, y);
    }
    return createQuantizedMeshTerrainData(
      provider,
      buffer,
      level,
      x,
      y,
      layerToUse
    );
  });
}
```

可以看到，在`const url = urlTemplates[(x + terrainY + level) % urlTemplates.length];`一句中拼接了请求的字符串，然后使用`Resource`类的`getDerivedResource`方法发送请求，发送请求后使用了`.fetchArrayBuffer()`方法将请求内容转换成一个`ArrayBuffer`。在Promise的请求完成后，调用了`createQuantizedMeshTerrainData`方法，解析数据。

在`createQuantizedMeshTerrainData`函数中，基本上就是按照上述的顺序一步一步来解析每一条信息。

首先预定义了一些常量，表示每个信息所用的字节数，然后使用一个`pos`变量表示当前解析位置。如，瓦片的位置信息包含xyz三个元素，每个元素是一个double类型，因此，其包含的字节数为`Float64Array.BYTES_PER_ELEMENT * 3`，一依次类推。

```javascript
let pos = 0;
  const cartesian3Elements = 3;
  const boundingSphereElements = cartesian3Elements + 1;
  const cartesian3Length = Float64Array.BYTES_PER_ELEMENT * cartesian3Elements;
  const boundingSphereLength =
    Float64Array.BYTES_PER_ELEMENT * boundingSphereElements;
  const encodedVertexElements = 3;
  const encodedVertexLength =
    Uint16Array.BYTES_PER_ELEMENT * encodedVertexElements;
  const triangleElements = 3;
  let bytesPerIndex = Uint16Array.BYTES_PER_ELEMENT;
  let triangleLength = bytesPerIndex * triangleElements;
```

然后将`ArrayBuffer`类型的信息转换为`DataView`，方便后续取数据

```javascript
const view = new DataView(buffer);
```

之后就开始解析数据了。

首先是瓦片中心点

```javascript
const center = new Cartesian3(
	view.getFloat64(pos, true),
	view.getFloat64(pos + 8, true),
	view.getFloat64(pos + 16, true)
);
pos += cartesian3Length;
```

然后为float类型的最大最小高程

```javascript
const minimumHeight = view.getFloat32(pos, true);
pos += Float32Array.BYTES_PER_ELEMENT;
const maximumHeight = view.getFloat32(pos, true);
pos += Float32Array.BYTES_PER_ELEMENT;
```

然后是瓦片的外接球，包括XYZ坐标和半径

```javascript
const boundingSphere = new BoundingSphere(
	new Cartesian3(
		view.getFloat64(pos, true),
		view.getFloat64(pos + 8, true),
		view.getFloat64(pos + 16, true)
	),
	view.getFloat64(pos + cartesian3Length, true)
);
pos += boundingSphereLength;
```

之后是地平线遮挡点

```javascript
const horizonOcclusionPoint = new Cartesian3(
	view.getFloat64(pos, true),
	view.getFloat64(pos + 8, true),
	view.getFloat64(pos + 16, true)
);
pos += cartesian3Length;
```

然后读取顶点数量和所有的压缩后的顶点信息

```javascript
const vertexCount = view.getUint32(pos, true);
pos += Uint32Array.BYTES_PER_ELEMENT;
const encodedVertexBuffer = new Uint16Array(buffer, pos, vertexCount * 3);
pos += vertexCount * encodedVertexLength;
```

之后由于索引数据的起始位置和类型会取决于顶点的数量，因此，判断了顶点数量是不是大于16位无符号短整型的最大值，如果超过最大值，则索引的类型就要使用32位无符号整型。

```javascript
if (vertexCount > 64 * 1024) {
	// More than 64k vertices, so indices are 32-bit.
	bytesPerIndex = Uint32Array.BYTES_PER_ELEMENT;
	triangleLength = bytesPerIndex * triangleElements;
}
```

在获取顶点数据之前，首先要对获取到的顶点的buffer进行解析，其中包含了uv以及高程数据。

首先将顶点buffer按照顶点数量分为三段，对应uv和height.

```javascript
// Decode the vertex buffer.
const uBuffer = encodedVertexBuffer.subarray(0, vertexCount);
const vBuffer = encodedVertexBuffer.subarray(vertexCount, 2 * vertexCount);
const heightBuffer = encodedVertexBuffer.subarray(
	vertexCount * 2,
	3 * vertexCount
);
```

然后使用zigZag解码方法对三个buffer进行解码。

```javascript
AttributeCompression.zigZagDeltaDecode(uBuffer, vBuffer, heightBuffer);
```

其解码方法与我们前文所说的一致。

```javascript
AttributeCompression.zigZagDeltaDecode = function (
  uBuffer,
  vBuffer,
  heightBuffer
) {
  //>>includeStart('debug', pragmas.debug);
  Check.defined("uBuffer", uBuffer);
  Check.defined("vBuffer", vBuffer);
  Check.typeOf.number.equals(
    "uBuffer.length",
    "vBuffer.length",
    uBuffer.length,
    vBuffer.length
  );
  if (defined(heightBuffer)) {
    Check.typeOf.number.equals(
      "uBuffer.length",
      "heightBuffer.length",
      uBuffer.length,
      heightBuffer.length
    );
  }
  //>>includeEnd('debug');

  const count = uBuffer.length;

  let u = 0;
  let v = 0;
  let height = 0;

  for (let i = 0; i < count; ++i) {
    u += zigZagDecode(uBuffer[i]);
    v += zigZagDecode(vBuffer[i]);

    uBuffer[i] = u;
    vBuffer[i] = v;

    if (defined(heightBuffer)) {
      height += zigZagDecode(heightBuffer[i]);
      heightBuffer[i] = height;
    }
  }
};

function zigZagDecode(value) {
  return (value >> 1) ^ -(value & 1);
}
```

解码完成后，开始对后续的索引数据继续进行解析。

首先，索引数据开始之前，先将当前的指针位置与数据格式对齐。即，如果是16位无符号整型，则索引数据开始之前的数据量是2字节的整数倍，如果是32位无符号整型，则索引数据开始之前的数据量是4字节的整数倍。

```javascript
// skip over any additional padding that was added for 2/4 byte alignment
if (pos % bytesPerIndex !== 0) {
	pos += bytesPerIndex - (pos % bytesPerIndex);
}
```

然后开始读取三角网中三角形的数量，之后是索引数量，索引数量是三角形数量的三倍。

```javascript
const triangleCount = view.getUint32(pos, true);
pos += Uint32Array.BYTES_PER_ELEMENT;
const indices = IndexDatatype.createTypedArrayFromArrayBuffer(
	vertexCount,
	buffer,
	pos,
	triangleCount * triangleElements
);
pos += triangleCount * triangleLength;
```

前面说过，索引的编码使用的是WEBGL的高水位编码法，因此，要对索引值进行解码。

```javascript
// High water mark decoding based on decompressIndices_ in webgl-loader's loader.js.
// https://code.google.com/p/webgl-loader/source/browse/trunk/samples/loader.js?r=99#55
// Copyright 2012 Google Inc., Apache 2.0 license.
let highest = 0;
const length = indices.length;
for (let i = 0; i < length; ++i) {
	const code = indices[i];
	indices[i] = highest - code;
	if (code === 0) {
		++highest;
	}
}
```

然后解析的是四条边上的索引数量以及索引数据。

```javascript
const westVertexCount = view.getUint32(pos, true);
pos += Uint32Array.BYTES_PER_ELEMENT;
const westIndices = IndexDatatype.createTypedArrayFromArrayBuffer(
	vertexCount,
	buffer,
	pos,
	westVertexCount
);
pos += westVertexCount * bytesPerIndex;

const southVertexCount = view.getUint32(pos, true);
pos += Uint32Array.BYTES_PER_ELEMENT;
const southIndices = IndexDatatype.createTypedArrayFromArrayBuffer(
	vertexCount,
	buffer,
	pos,
	southVertexCount
);
pos += southVertexCount * bytesPerIndex;

const eastVertexCount = view.getUint32(pos, true);
pos += Uint32Array.BYTES_PER_ELEMENT;
const eastIndices = IndexDatatype.createTypedArrayFromArrayBuffer(
	vertexCount,
	buffer,
	pos,
	eastVertexCount
);
pos += eastVertexCount * bytesPerIndex;

const northVertexCount = view.getUint32(pos, true);
pos += Uint32Array.BYTES_PER_ELEMENT;
const northIndices = IndexDatatype.createTypedArrayFromArrayBuffer(
	vertexCount,
	buffer,
	pos,
	northVertexCount
);
pos += northVertexCount * bytesPerIndex;
```
边界索引处理完后，最后要处理的就是扩展信息。

由于扩展信息的字节数是不确定的，但是扩展信息的结构是一定的，一定是8位的扩展ID+32位的扩展长度+扩展长度位的扩展信息，因此，只要按照这个结构循环读下去，直到读到文件结尾即可。

```javascript
let encodedNormalBuffer;
let waterMaskBuffer;
while (pos < view.byteLength) {
	const extensionId = view.getUint8(pos, true);
	pos += Uint8Array.BYTES_PER_ELEMENT;
	const extensionLength = view.getUint32(pos, littleEndianExtensionSize);
	pos += Uint32Array.BYTES_PER_ELEMENT;

	if (
		extensionId === QuantizedMeshExtensionIds.OCT_VERTEX_NORMALS &&
		provider._requestVertexNormals
	) {
		encodedNormalBuffer = new Uint8Array(buffer, pos, vertexCount * 2);
	} else if (
		extensionId === QuantizedMeshExtensionIds.WATER_MASK &&
		provider._requestWaterMask
	) {
		waterMaskBuffer = new Uint8Array(buffer, pos, extensionLength);
	} else if (
		extensionId === QuantizedMeshExtensionIds.METADATA &&
		provider._requestMetadata
	) {
		const stringLength = view.getUint32(pos, true);
		if (stringLength > 0) {
			const metadata = getJsonFromTypedArray(
				new Uint8Array(buffer),
				pos + Uint32Array.BYTES_PER_ELEMENT,
				stringLength
			);
			const availableTiles = metadata.available;
			if (defined(availableTiles)) {
				for (let offset = 0; offset < availableTiles.length; ++offset) {
					const availableLevel = level + offset + 1;
					const rangesAtLevel = availableTiles[offset];
					const yTiles = provider._tilingScheme.getNumberOfYTilesAtLevel(
						availableLevel
					);

					for (
						let rangeIndex = 0;
						rangeIndex < rangesAtLevel.length;
						++rangeIndex
					) {
						const range = rangesAtLevel[rangeIndex];
						const yStart = yTiles - range.endY - 1;
						const yEnd = yTiles - range.startY - 1;
						provider.availability.addAvailableTileRange(
							availableLevel,
							range.startX,
							yStart,
							range.endX,
							yEnd
						);
						layer.availability.addAvailableTileRange(
							availableLevel,
							range.startX,
							yStart,
							range.endX,
							yEnd
						);
					}
				}
			}
		}
		layer.availabilityTilesLoaded.addAvailableTileRange(level, x, y, x, y);
	}
	pos += extensionLength;
}
```

可以看到，读取地形光照和水面掩码都很简单，根据数据特征去读取一定数量的数据即可。对于元信息，由于读出的是个字符串格式的JSON文件，因此还要对这个字符串进行解析。至于如何解析这个字符串，自行查阅`getJsonFromTypedArray`函数。

最后，添加了一些构建内部类需要的数据，如裙边高程，矩形范围、外包围盒等，然后组成了一个`QuantizedMeshTerrainData`类。

```javascript
const skirtHeight = provider.getLevelMaximumGeometricError(level) * 5.0;

// The skirt is not included in the OBB computation. If this ever
// causes any rendering artifacts (cracks), they are expected to be
// minor and in the corners of the screen. It's possible that this
// might need to be changed - just change to `minimumHeight - skirtHeight`
// A similar change might also be needed in `upsampleQuantizedTerrainMesh.js`.
const rectangle = provider._tilingScheme.tileXYToRectangle(x, y, level);
const orientedBoundingBox = OrientedBoundingBox.fromRectangle(
	rectangle,
	minimumHeight,
	maximumHeight,
	provider._tilingScheme.ellipsoid
);

return new QuantizedMeshTerrainData({
	center: center,
	minimumHeight: minimumHeight,
	maximumHeight: maximumHeight,
	boundingSphere: boundingSphere,
	orientedBoundingBox: orientedBoundingBox,
	horizonOcclusionPoint: horizonOcclusionPoint,
	quantizedVertices: encodedVertexBuffer,
	encodedNormals: encodedNormalBuffer,
	indices: indices,
	westIndices: westIndices,
	southIndices: southIndices,
	eastIndices: eastIndices,
	northIndices: northIndices,
	westSkirtHeight: skirtHeight,
	southSkirtHeight: skirtHeight,
	eastSkirtHeight: skirtHeight,
	northSkirtHeight: skirtHeight,
	childTileMask: provider.availability.computeChildMaskForTile(level, x, y),
	waterMask: waterMaskBuffer,
	credits: provider._tileCredits,
});
```
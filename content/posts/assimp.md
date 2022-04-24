---
title: "Working with procedural meshes in Assimp"
date: 2021-12-11T10:18:00-06:00
draft: false
tags: ['3d','assimp','mesh','models', 'gltf', 'fbx', 'obj', 'unity', 'opengl']
---

Assimp or Open Asset Import Library is a really nice library for handling lots of various 3D model formats such as OBJ, glTF, and FBX. It can both import and export many formats.

I wanted to make a short guide on some of the more tricky things to do or less well documented parts of the library such as procedural meshes (such as making meshes from another system and wanting to export them using assimp).

## Coordinates, Handedness, and Units

Assimp is right handed with y up. There is no special consideration of units in regards but does expose things like `UnitScaleFactor`. When importing assets you can view these via the `scene->mMetaData` such as something like this:

~~~ c++
if(scene->mMetaData != nullptr)
{
	for(int i=0; i < scene->mMetaData->mNumProperties; i++)
	{
		const aiString* key;
		const aiMetadataEntry* entry;
		auto res = scene->mMetaData->Get(i,key,entry);
		if(!res)
		{
			std::cerr << "Failed to get property at index: " << i << std::endl;
			continue;
		}

		if(entry->mType == aiMetadataType::AI_BOOL)
		{
			std::cout << " bool: " << key->C_Str() << " => " << (*(bool *) entry->mData) << std::endl;
		} else if(entry->mType == aiMetadataType::AI_INT32)
		{
			std::cout << " int32: " << key->C_Str() << " => " << (*(int32_t *) entry->mData) << std::endl;
		} else if(entry->mType == aiMetadataType::AI_UINT64)
		{
			std::cout << " uint64: " << key->C_Str() << " => " << (*(uint64_t *) entry->mData) << std::endl;
		} else if(entry->mType == aiMetadataType::AI_FLOAT)
		{
			std::cout << " float: " << key->C_Str() << " => " <<(*(float *) entry->mData) << std::endl;
		} else if(entry->mType == aiMetadataType::AI_DOUBLE)
		{
			std::cout << " double: " << key->C_Str() << " => " << (*(double *) entry->mData) << std::endl;
		} else if(entry->mType == aiMetadataType::AI_AISTRING)
		{
			std::cout << " string: " << key->C_Str() << " => " << ((aiString *)entry->mData)->C_Str() << std::endl;
		} else if(entry->mType == aiMetadataType::AI_AIVECTOR3D)
		{
			auto v3 = ((aiVector3D *)entry->mData);
			std::cout << " vec3: " << key->C_Str() << " => " << v3->x << "," << v3->y << "," << v3->z << std::endl;
		} else if(entry->mType == aiMetadataType::AI_AIMETADATA)
		{
			//TODO recurse into the meta data

		} else if(entry->mType == aiMetadataType::AI_META_MAX)
		{

		}
	}
}
~~~

For example it might produce output like:

```
 int32: UpAxis => 1
 int32: UpAxisSign => 1
 int32: FrontAxis => 2
 int32: FrontAxisSign => 1
 int32: CoordAxis => 0
 int32: CoordAxisSign => 1
 int32: OriginalUpAxis => 2
 int32: OriginalUpAxisSign => 1
 float: UnitScaleFactor => 1
 float: OriginalUnitScaleFactor => 1
 int32: FrameRate => 6
 uint64: TimeSpanStart => 0
 uint64: TimeSpanStop => 0
 float: CustomFrameRate => -1
 string: SourceAsset_FormatVersion => 7400
 string: SourceAsset_Generator => FBX SDK/FBX Plugins version 2018.1
 string: SourceAsset_Format => Autodesk FBX Importer
```

In regards to the terms such as the `Axis` and `Sign` they are coming from FBX's API. For Sign +1 is forward and -1 is away or +1 is up and -1 is down or +1 is right and -1 is left. In other words if you have Y going up the the sign should be 1. In pretty much all systems Y or Z is up, X is either left or right, and Y or Z is forward/backward. If you're confused about handedness see [Practical 3D Math for Programmers]({{< ref "linear_algebra" >}}). In this case UpAxis is pointing up and down, FrontAxis is pointing towards or away, and CoordAxis is the crossproduct so the left and right. The Axis values can be 0, 1, or 2 which represent x, y, or z respectively. So if you have Up:1 UpSign:1 Front:2 FrontSign:1 and Coord:0,CoordSign:1 that would me +Y up, +X right, and +Z forward, which conversely means you're using right hand system for the imported scene. This dynamic axis for the scene is something fairly unique to FBX files, glTF has a specific coordinate handling as it's standard (+Y up, +Z forward, +X right, righ handed with units as meters, angles as radians, and b/c it's right handed rotations are counter clockwise).

When you are creating your meshes from scratch for assimp you probably want to convert them into their format (right handed, y up which follows OpenGL and Maya). In my case we are left handed which follows Unity and DirectX). There are ways to have assimp convert for you in it's post processing step (which most work in both importing and exporting) but I found it easier to just convert into their system and avoid confusion.

See also:
1. `aiProcess_MakeLeftHanded` - flips from right hand to left hand axis (y is still up)
1. `aiProcess_FlipWindingOrder` - flips from counter clockwise to clockwise
1. `aiProcess_FlipUVs` - flips the y of UVs

See also:
1. https://help.autodesk.com/view/FBX/2017/ENU/?guid=__cpp_ref_class_fbx_axis_system_html
1. https://github.com/arturoc/ofxFBX/blob/master/libs/fbxsdk/include/fbxsdk/scene/fbxaxissystem.h


## Mixing Primitives

When mixing triangles and quads (or other polygons) you might have another mesh with the same name as well for both a set of tris and quads. When you deal with this in a game engine such as Unity it will often convert all the quad and other polygons into triangles. Note, unity does allow you to keep quad data in file import settings but your mesh will still be triangulated as well in the main set of triangles. When dealing with this in a library like assimp you will have raw access to the original data without it being merged.

A triangle such as this:

![](https://i.imgur.com/gakZihb.png)

```
0: Plane mat: 0 DefaultMaterial verts: 3 faces: 1 bones: 0 triangles
```

A quad plus two triangles such as this:

![](https://i.imgur.com/WCBObpx.png)

```
0: Plane mat: 0 DefaultMaterial verts: 6 faces: 2 bones: 0 triangles
1: Plane mat: 0 DefaultMaterial verts: 4 faces: 1 bones: 0 polygons
```

## Submesh and Materials

The terminology around submeshes vs materials is a little loose between engines and DCC (Digital Content Creator). I generally like the term Unity uses; in which is a submesh contains a subset of the collection of all faces for a mesh and has one material (though the material may be null). In other words all vertices belong to the mesh and then the faces are owned by a submesh. How this gets assigned is handled by the material assignments from the DCC.

Because assimp only allows one material per mesh I was wondering how they handled the concept of a submesh (both in import and export). Essentially there will be multiple meshes with the same name repeeated for each submesh/material with a subset of the mesh primitives (vertices/uvs/etc).

An example of rectangle with two materials "Left" and "Right":

![](https://i.imgur.com/tvUzUtY.png)

```
0: Plane.001 mat: 0 Left verts: 4 faces: 1 bones: 0 polygons
1: Plane.001 mat: 1 Right verts: 4 faces: 1 bones: 0 polygons
```

Unity's importer:

![](https://i.imgur.com/IsOhmhh.png)

Unity will import this as a single mesh with 2 submeshes, 6 verts, 4 faces, 2 materials.

## Bones

Bones in assimp are quite similar to the bones you'd use in a game engine. They are represented by a series of `aiNode` for the transform data in a tree fashion along with an array of `aiBone` that reference the node not by reference or slot but instead by name. It's important that the names must match exactly. The bones array instead will be referenced by the `aiMesh`.

### Bindpose

Each bone should have an `mOffsetMatrix` that is analagous to the `Mesh.bindposes` in Unity. You need to be careful about your own coordinate system when converting the transform data into assimp. In my case our source data was coming in as a left hand y up system so I decided to rotate and swizzle my data from my source to assimp's system that way I didn't have to rely on any post processing quirks.
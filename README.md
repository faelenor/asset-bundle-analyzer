# This project has been deprecated!

It has been replaced by a much faster (25-50x) C# tool available here:
[https://github.com/Unity-Technologies/UnityDataTools](https://github.com/Unity-Technologies/UnityDataTools)

# asset-bundle-analyzer
This tool extracts useful information from Unity asset bundles and stores the information in an SQLite database.

## Getting started

You need Python 3 to run this program. You will also need a tool such as [DB Browser for SQLite](https://sqlitebrowser.org/) to query the database.

This is a command line tool with two mandatory arguments. The first one is the path of the Unity tools folder and the second is the path of the root folder containing the asset bundles to analyze. For example:

    python analyzer.py /Applications/Unity/Unity.app/Contents/Tools ~/projects/MyGame/AssetBundles

Note that you must use the same tools that have been used to build the asset bundles, otherwise it will fail.

## Optional arguments

* -p PATTERN, --pattern PATTERN: wildcard pattern used to recursively find asset bundles in the specified folder (default: * )
*  -o OUTPUT, --output OUTPUT: name of the output database file (default: database)
*  -k, --keep-temp: keep the files generated by WebExtract and binary2text in the asset bundle folder (default: False)
*  -r, --store-raw: store raw json objects in 'raw_objects' database table (default: False)

## Database structure

The main table is called *objects* and it contains a row for every object in the asset bundles. It's best to use the *object_view* view instead because it includes useful information from other tables as well. The primary key is called *id* and it doesn't represent anything, it's just a unique integer. The natural key would have been the *(file, object_id)* tuple, but dealing with composite keys complicates all queries so a simpler key is used instead.

The columns in *object_view* are:
* **object_id**: Unity object id
* **bundle**: name of the asset bundle containing this object
* **file**: name of the file (in the asset bundle) containing this object
* **class_id**: Unity class id of that object
* **type**: type name
* **name**: name of the object, if available (components don't have names)
* **game_object**: id of the parent game object, if there's one (components have a parent game object)
* **size**: size of the serialized object, before compression (the real size in the asset bundle may be smaller if LZ4 or LZMA compression is used)
* **serialized_fields**: the total number of serialized fields of this object

### Type-specific views

For some types, there is an additional view with type-specific information:
* **animation_view** for AnimationClips: legacy (0 = mecanim, 1 = legacy)
* **audio_clip_view** for AudioClips: bits_per_sample, frequency (Hz), channels, type (loading type), format
* **mesh_view** for Meshes: indices, vertices, compression (0 = uncompressed, 1 = low, 2 = medium, 3 = high), rw_enabled
* **shader_view** for Shaders: counts of properties, sub_shaders, sub_programs and unique_programs (the number of unique GPU programs that are packed with this shader as some subprograms may share the same code), keywords (all keywords that this shader supports)
* **shader_subprogram_view** for Shader subprograms: api_type, pass, hw_tier (from quality settings), prog_type (fragment, vertex, etc.), prog_keywords (keyword set for each subprogram).
* **texture_view** for Texture2D: format, width, height, mip_count, rw_enabled

### Additional views

Additional views are also provided:
* **view_breakdown_by_type**: total number and total size of all objects per type
* **view_breakdown_shaders**: for every shader, the number for instances (> 1 means duplicate), total size and list of asset bundles containing it (separated by line breaks)
* **view_mipmapped_textures**: list of all textures with mipmaps, useful to spot UI texture that should not have mipmaps
* **view_potential_duplicates**: list of all potentially duplicated objects and in which asset bundles (there may be false positives)
* **view_references_to_default_material**: list all game objects having a reference to the default material and indirectly referencing the Default shader
* **view_rw_meshes/textures**: list all meshes (or textures) with Read/Write enabled
* **view_suspicious_audio_clips**: list all clips that are streamed and should not, or the opposite

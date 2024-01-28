# DoViBaker
Bake the DoVi into your clip

This avisynth plugin reads the Base Layer, Enhancement Layer and RPU data from a profile 7 DolbyVision stream to create a clip with the DolbyVision data baked in.

## General information
This plugin uses the metadata from and RPU file or from the inside stream itself to compose the DolbyVision HDR picture out of the Base Layer (BL) and Enhancement Layer (EL). Display Management (DM) metadata will not be processed per default. It is however possible to use level 1 maximal pixel brightness data from DM by providing it to DoViCubes which can apply different LUTs depding on threshold values. 

## Feeding the plugin 
To my knowledge there are currently three source libraries that can be used. It is advisable to choose one of them in a speed test on your machine.

### [LSMASHSource](https://github.com/HomeOfAviSynthPlusEvolution/L-SMASH-Works)

example.avs:
```
bl=LWLibavVideoSource("clip.ts", format="YUV420P10", stream_index=0)
el=LWLibavVideoSource("clip.ts", format="YUV420P10", stream_index=1)
DoViBaker(bl,el)
```
### [FFmpegSource](http://avisynth.nl/index.php/FFmpegSource)

example.avs:
```
bl=FFVideoSource("clip.ts", threads=1, track=0)
el=FFVideoSource("clip.ts", threads=1, track=1)
DoViBaker(bl,el)
```

### [DGDecNV](https://www.rationalqm.us/dgdecnv/binaries/)

1. Get dovi_tool: https://github.com/quietvoid/dovi_tool/releases/tag/2.1.0
2. Extract the Base and Enhancement Layers separately from the initial profile 7 stream
3. Extract the RPU data from the Enhancement Layer using dovi_tool
4. Write a Avisynth script like the example below

example.avs:
```
bl=DGSource("blclip.dgi")
el=DGSource("elclip.dgi")
DoViBaker(bl,el,rpu="RPU.bin")
```

## Trims
Also it is possible to apply the trims available in the stream. Select which trim to apply using the `trimPq` argument and set `targetMaxNits` and `targetMinNits` as necessary. Be warned however, only the typical CM v2.9 processing is implemented thus far, and most streams have not very optimized parameters, producing suboptimal results. Thus this feature is experimental only!

## Frame Properties
The following frame properties will be set:
- `_Matrix` set to 0, representing that the output is RGB
- `_ColorRange` set to 0 in case of full range and 1 in case of limited range
- `_SceneChangePrev` set to 1 for the first frame in a scene
- `_dovi_dynamic_min_pq` the min_pq value of the current scene
- `_dovi_dynamic_max_pq` the max_pq value of the current scene
- `_dovi_dynamic_max_content_light_level` the equivalend value of maximal nits of the current scene
- `_dovi_static_max_pq` the max_pq value of the whole stream
- `_dovi_static_max_content_light_level` the value of maximal nits of the whole stream

You can get the current tonemapping value of max-content-light-level by reading the frame property `_dovi_dynamic_max_content_light_level`:
```
ScriptClip("""
mcll=propGetInt("_dovi_dynamic_max_content_light_level")
subtitle("maxcll = " + string(mcll))
""")
```

# DoViTonemap
This plugin processes the tonemapping of any HDR PQ streams to lower dynamic range targets. The implementation follows ITU-R BT.2408-7 Annex 5, with the addtion of an optional luminosity factor which scales the brightness linearily.
There are 5 arguments which control the process: `masterMaxNits`, `masterMinNits`, `targetMaxNits`, `targetMinNits` and `lumScale`. The first two arguments set the white and black brightness value of the source, and next two set the white and black brightness value of the target. The values for the master brightness can be either given explicitly or `masterMaxNits` and `masterMinNits` can both be set to `-1` which will indicate that the actual values are read from the related frame properties set by DoViBaker, leading to a dynamic tonemapping. `lumScale` changes the total brightness, this can be usefull since many HDR PQ and DV streams are actually too dark, darker then the respective SDR streams. To find the proper `lumScale` factor you might use the script `LumScaleFindHelper.avs`.

This example applies a dynamic tonemapping to a 1000nits target reading the current max brightness value off the frame properits which are set by DoViBaker. The luminosity scale used is 1.0. To increase the brightness this factor can be increased to 1.5 or 2.0 or even higher. There have been instances where factors of 4.5 where needed to match the percieved brightness of the SDR stream.
```
DoViBaker(bl,el)
DoViTonemap(lumScale=1.0, masterMaxNits=-1, targetMaxNits=1000, masterMinNits=0, targetMinNits=0)
```
## Frame Properties
The following frame properties will be consumed:
- `_dovi_dynamic_max_pq` the max_pq value of the current scene
- `_dovi_dynamic_min_pq` the min_pq value of the current scene
- `_dovi_dynamic_luminosity_scale` the luminosity of the current scene scaling factor

# DoViCubes
This plugin provides LUT processing capabilites based on the frame property `_dovi_dynamic_max_content_light_level` set by either `DoViBaker` or `DoViStatsFileReader`. Different LUTs are applied based adjustable thresholds. This is done by providing a collection of LUTs and limits of validity measured in nits of max-content-light-level. (The LUT processing implentation is based on: https://github.com/sekrit-twc/timecube).
```
DoViBaker(bl,el)
DoViCubes(cubes="lut_1000.cube;lut_2000.cube;lut_4000.cube",mclls="1010;2020",cubes_basepath="C:\")
```
This example will use the file lut_1000.cube for frames where the max-content-light-level is below or equal to 1010 nits, the file lut_2000.cube for above 1010 but below or equal 2020 nits and lut_4000.cube for all frames above 2020 nits. All cube files must be available in the path given to cubes_basepath, in this example it would be "C:\\".

## Frame Properties
The following frame properties will be consumed:
- `_dovi_dynamic_max_content_light_level` the equivalend value of maximal nits of the current scene

# DoViStatsFileLoader
This plugin reads the stats file generated by the avisynth script `StatsFileCreator.avs`. It can be used for sources which do not have any DolbyVision substream, but where a the LUT processing by DoViCubes is still necessary. Additionally it is possible to provide an alternative scene cut file, created by other means.
The format of each line of the stats file needs to be, last entry is optional:  
`<frame_number> <decision_if_frame_is_last_before_scene_cut> <frame_max_pq> <frame_lum_scale>`

The format of each line of the optional alternative scene cut file file needs to be:  
`<frame_number_of_first_scene_after_scene_cut>`

In this example the input stats file is read feeding DoViTonemap.
```
DoViStatsFileReader("statsFile.txt")
DoViTonemap(lumScale=1.0, masterMaxNits=-1, targetMaxNits=1000, masterMinNits=0, targetMinNits=0)
```

## Frame Properties
The following frame properties will be set:
- `_SceneChangePrev` set to 1 for the first frame in a scene
- `_SceneChangeNext` set to 1 for the last frame in a scene
- `_dovi_dynamic_min_pq` the min_pq value of the current scene
- `_dovi_dynamic_max_pq` the max_pq value of the current scene
- `_dovi_dynamic_max_content_light_level` the equivalend value of maximal nits of the current scene
- `_dovi_dynamic_luminosity_scale` the optional luminosity of the current scene scaling factor
- `_dovi_static_max_pq` the max_pq value of the whole stream
- `_dovi_static_max_content_light_level` the value of maximal nits of the whole stream

# StatsFileCreator.avs
This avisynth script scans through the clip and writes the stats file needed for `DoViStatsFileReader`. The stats file includes scene cuts and per-frame max brightness values. The scene cut detection algorithm is a rather simple implementation, which is however good enough for most cases.

The format of each line of the stats file created is:  
`<frame_number> <decision_if_frame_is_last_before_scene_cut> <frame_max_pq>`

The format of each line of the optional alternative scene cut file created is:  
`<frame_number_of_first_scene_after_scene_cut>`


# LumScaleHelper.avs
Used to find `lumScale` for `DoViTonemap`. This is the factor by which to mutiply the brightness of the PQ stream such that its base brightness matches that of the SDR stream.

# BetterGrayscale.avsi
Needed by `LumScaleHelper.avs` for showing a more correct and better comparable grayscale of PQ and SDR sources.

# DoViAnalyzer
This application analyzes the RPU.bin file in order to show information relevant to deciding whether it is worth to use DoViBaker or if this can be skipped completly and the Base Layer can be used directly.

```
usage: DoViAnalyzer.exe <path_to_rpu.bin_file> <optional_scenecutfile.txt>
```
The output will show the following attributes:
1. clip length
2. overall max-content-light-level
3. unusual color matrices defined by the RPU file
4. mapping non-identity introduced by the RPU file
5. enabled processing of the Enhancement Layer
6. available trims, with the trimPq of each trim being shown in brackets
7. optional warning in case that the RGB output is of limited range

Pay attention to 3-5 since these will indicate if the look of the clip will be different when DolbyVision is taken into account compared to just playing the Base Layer clip. If you are using LUTs, pay attention to 7) since in this case the LUTs provided will need to be different.

Additionally it is possible to generate a scenecutfile based on the information from the RPU file. This might be given to the encoder to improve the scene detection (using the parameter --qpfile for x265).

# AVSCube
There is also a simple implementation of AVSCube, which provides the same image processing as the integrated LUT processing as `DoViCubes`.

# Remarks concerning compilation
I had some issues linking against Timecube. I was constantly getting the following error:
```
fatal error C1083: Cannot open compiler generated file: 'x64\Release\timecube.asm': No such file or directory
```

It turnes out that this is related to the following setting: "Properies" (of the timecube project) -> "C++" -> "Output Files" -> "Assembler Output".
Setting this to "No Listing" resolves the issue.

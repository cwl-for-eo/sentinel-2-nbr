# Sentinel-2 Normalized Burn Ratio (NBR)

The goal is to produce a Sentinel-2 NBR geotif using CWl to orchestrate gdal and OTB.

The processing steps are: 

* use `gdal_translate` to clip the COG over the area of interest expressed as a bounding box for the NIR, SWIR22 and Scene Classification
* use OTB's `otbcli_BandMathX` to do the band arithmetic and generate the NBR 

## Data used

The data used are Sentinel-2 products that were converted to COG and STAC and published on publicly accessible URLs.

## Inputs

The inputs are: 

- A Sentinel-2 STAC Item URL
- An area of interest expressed as a bounding box 

## Steps

### STAC asset href resolution 

NBR requires the NIR, SWIR Sentinel-2 assets and we'll use the Scene Classification to process the normalized burn ratio only over valid pixels (e.g. avoid clouds, water, etc.).

The first workflow step is getting the B8A (NIR), B12 (SWIR22) and the Scene Classification hrefs.

This is done with the CWL `CommandLineTool`  available in the `asset.cwl` file:

```yaml
class: CommandLineTool
 
requirements:
  DockerRequirement: 
    dockerPull: terradue/jq
  ShellCommandRequirement: {}
  InlineJavascriptRequirement: {}

baseCommand: curl
arguments:
- -s
- valueFrom: ${ return inputs.stac_item; }
- "|"
- jq
- valueFrom: ${ return ".assets." + inputs.asset + ".href"; }
- "|"
- tr 
- -d
- '\"' #\""

stdout: message

inputs:
  stac_item:
    type: string
  asset:
    type: string

outputs:

  asset_href: 
    type: string
    outputBinding:
      glob: message
      loadContents: true
      outputEval: $( self[0].contents.split("\n").join("") )

cwlVersion: v1.0
```

### Extract the data over the area of interest

`gdal_translate` can take VSI URLs as input and extract an area of interest.

This step takes the resolved STAC assets href and produces clips of the original Sentinel-2 acquisition.

This is achieved with the CWL `CommandLineTool` available in the `translate.cwl` file:

```yaml
class: CommandLineTool

requirements:
  InlineJavascriptRequirement: {}
  DockerRequirement: 
    dockerPull: osgeo/gdal

baseCommand: gdal_translate
arguments:
- -projwin 
- valueFrom: ${ return inputs.bbox.split(",")[0]; }
- valueFrom: ${ return inputs.bbox.split(",")[3]; }
- valueFrom: ${ return inputs.bbox.split(",")[2]; }
- valueFrom: ${ return inputs.bbox.split(",")[1]; }
- -projwin_srs
- valueFrom: ${ return inputs.epsg; }
- valueFrom: |
    ${ if (inputs.asset.startsWith("http")) {

         return "/vsicurl/" + inputs.asset; 
       
       } else { 
        
         return inputs.asset;

       } 
    }
- valueFrom: ${ return inputs.asset.split("/").slice(-1)[0]; }

inputs: 
  asset: 
    type: string
  bbox: 
    type: string
  epsg:
    type: string
    default: "EPSG:4326" 

outputs:
  tifs:
    outputBinding:
      glob: '*.tif'
    type: File

cwlVersion: v1.0
```

### Normalized difference using the clipped tifs.

The final step uses OTB's `otbcli_BandMathX` to do the band arithmetic and generate the NBR.

The CWl `CommandLineTool` to accomplish that is available in the `band_math.cwl` file:

```yaml
class: CommandLineTool

requirements:
  InlineJavascriptRequirement: {}
  DockerRequirement: 
    dockerPull: terradue/otb-7.2.0

baseCommand: otbcli_BandMathX
arguments: 
- -out
- valueFrom: ${ return inputs.stac_item.split("/").slice(-1)[0] + ".tif"; }
- -exp
- '(im3b1 == 8 or im3b1 == 9 or im3b1 == 0 or im3b1 == 1 or im3b1 == 2 or im3b1 == 10 or im3b1 == 11) ? -2 : (im1b1 - im2b1) / (im1b1 + im2b1)'

inputs:

  tifs:
    type: File[]
    inputBinding:
      position: 5
      prefix: -il
      separate: true
  
  stac_item:
    type: string
    
outputs:

  nbr:
    outputBinding:
      glob: "*.tif"
    type: File

cwlVersion: v1.0
```

## Putting the pieces together 

The single CWL documents encapsulating the different steps of the final workflow are orchestrated as CWL `Workflow`.

```yaml

class: Workflow
label: NBR - produce the normalized difference between NIR and SWIR 22 
doc: NBR - produce the normalized difference between NIR and SWIR 22 
id: main

requirements:
- class: ScatterFeatureRequirement

inputs:
  stac_item: 
    doc: Sentinel-2 item
    type: string
  aoi: 
    doc: area of interest as a bounding box
    type: string
  bands: 
    type: string[]
    default: ["B8A", "B12", "SCL"]
  
outputs:
  nbr:
    outputSource:
    - node_nbr/nbr
    type: File

steps:

  node_stac:
    
    run: asset.cwl
        
    in:
      stac_item: stac_item
      asset: bands

    out:
      - asset_href

    scatter: asset
    scatterMethod: dotproduct 

  node_subset:

    run: translate.cwl  

    in: 
      asset: 
        source: node_stac/asset_href
      bbox: aoi

    out:
    - tifs
        
    scatter: asset
    scatterMethod: dotproduct

  node_nbr:
    
    run: band_math.cwl

    in:
      stac_item: stac_item
      tifs: 
        source: [node_subset/tifs]
      
    out:
    - nbr

cwlVersion: v1.0
```

The workflow uses CWL's `ScatterFeatureRequirement` to run some of the steps in parallel:

- Each asset is resolved as an individual process
- Each `gdal_translate` is invoked individually

The final node aggregates the inputs into a the NBR product.

This CWL Workflow can be invoked with the parameters:

```yaml
stac_item: "https://earth-search.aws.element84.com/v0/collections/sentinel-s2-l2a-cogs/items/S2B_53HPA_20210723_0_L2A"
aoi: "136.659,-35.96,136.923,-35.791"
```

And using `cwltool`: 

```console
cwltool --parallel nbr.cwl nbr.yml
```

This will output:

```console
INFO /srv/conda/bin/cwltool 3.0.20210319143721
INFO Resolved 'nbr.cwl' to 'file:///home/fbrito/work/sentinel-2-nbr/nbr.cwl'
INFO [workflow ] starting step node_stac
INFO [step node_stac] start
INFO [workflow ] start
INFO [step node_stac] start
INFO [step node_stac] start
INFO [job node_stac] /tmp/6z7ooxq4$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/6z7ooxq4,target=/AHORGP \
    --mount=type=bind,source=/tmp/fxfeqivt,target=/tmp \
    --workdir=/AHORGP \
    --read-only=true \
    --log-driver=none \
    --user=1000:1000 \
    --rm \
    --env=TMPDIR=/tmp \
    --env=HOME=/AHORGP \
    --cidfile=/tmp/36mv9qqs/20210803131208-777472.cid \
    terradue/jq \
    /bin/sh \
    -c \
    'curl' '-s' 'https://earth-search.aws.element84.com/v0/collections/sentinel-s2-l2a-cogs/items/S2B_53HPA_20210723_0_L2A' | 'jq' '.assets.B8A.href' | 'tr' '-d' \" > /tmp/6z7ooxq4/message
INFO [job node_stac_2] /tmp/lukn7xa_$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/lukn7xa_,target=/AHORGP \
    --mount=type=bind,source=/tmp/j3ojpzbw,target=/tmp \
    --workdir=/AHORGP \
    --read-only=true \
    --log-driver=none \
    --user=1000:1000 \
    --rm \
    --env=TMPDIR=/tmp \
    --env=HOME=/AHORGP \
    --cidfile=/tmp/2pbnjsek/20210803131208-803455.cid \
    terradue/jq \
    /bin/sh \
    -c \
    'curl' '-s' 'https://earth-search.aws.element84.com/v0/collections/sentinel-s2-l2a-cogs/items/S2B_53HPA_20210723_0_L2A' | 'jq' '.assets.B12.href' | 'tr' '-d' \" > /tmp/lukn7xa_/message
INFO [job node_stac_3] /tmp/8susbghr$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/8susbghr,target=/AHORGP \
    --mount=type=bind,source=/tmp/w3eo95ll,target=/tmp \
    --workdir=/AHORGP \
    --read-only=true \
    --log-driver=none \
    --user=1000:1000 \
    --rm \
    --env=TMPDIR=/tmp \
    --env=HOME=/AHORGP \
    --cidfile=/tmp/wgl0e8p6/20210803131208-816645.cid \
    terradue/jq \
    /bin/sh \
    -c \
    'curl' '-s' 'https://earth-search.aws.element84.com/v0/collections/sentinel-s2-l2a-cogs/items/S2B_53HPA_20210723_0_L2A' | 'jq' '.assets.SCL.href' | 'tr' '-d' \" > /tmp/8susbghr/message
INFO [job node_stac] Max memory used: 0MiB
INFO [job node_stac_3] Max memory used: 0MiB
INFO [job node_stac_2] Max memory used: 0MiB
INFO [job node_stac] completed success
INFO [job node_stac_3] completed success
INFO [job node_stac_2] completed success
INFO [step node_stac] completed success
INFO [workflow ] starting step node_subset
INFO [step node_subset] start
INFO [step node_subset] start
INFO [step node_subset] start
INFO [job node_subset] /tmp/354unm3n$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/354unm3n,target=/AHORGP \
    --mount=type=bind,source=/tmp/rna17u4f,target=/tmp \
    --workdir=/AHORGP \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --env=TMPDIR=/tmp \
    --env=HOME=/AHORGP \
    --cidfile=/tmp/bu2jkwzt/20210803131211-498283.cid \
    osgeo/gdal \
    gdal_translate \
    -projwin \
    136.659 \
    -35.791 \
    136.923 \
    -35.96 \
    -projwin_srs \
    EPSG:4326 \
    /vsicurl/https://sentinel-cogs.s3.us-west-2.amazonaws.com/sentinel-s2-l2a-cogs/53/H/PA/2021/7/S2B_53HPA_20210723_0_L2A/B8A.tif \
    B8A.tif
INFO [job node_subset_2] /tmp/4wetamd3$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/4wetamd3,target=/AHORGP \
    --mount=type=bind,source=/tmp/lb07hcp3,target=/tmp \
    --workdir=/AHORGP \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --env=TMPDIR=/tmp \
    --env=HOME=/AHORGP \
    --cidfile=/tmp/op_rhfej/20210803131211-569257.cid \
    osgeo/gdal \
    gdal_translate \
    -projwin \
    136.659 \
    -35.791 \
    136.923 \
    -35.96 \
    -projwin_srs \
    EPSG:4326 \
    /vsicurl/https://sentinel-cogs.s3.us-west-2.amazonaws.com/sentinel-s2-l2a-cogs/53/H/PA/2021/7/S2B_53HPA_20210723_0_L2A/B12.tif \
    B12.tif
INFO [job node_subset_3] /tmp/zd2y8jwn$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/zd2y8jwn,target=/AHORGP \
    --mount=type=bind,source=/tmp/vr39h9nm,target=/tmp \
    --workdir=/AHORGP \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --env=TMPDIR=/tmp \
    --env=HOME=/AHORGP \
    --cidfile=/tmp/3yl71gjr/20210803131211-604042.cid \
    osgeo/gdal \
    gdal_translate \
    -projwin \
    136.659 \
    -35.791 \
    136.923 \
    -35.96 \
    -projwin_srs \
    EPSG:4326 \
    /vsicurl/https://sentinel-cogs.s3.us-west-2.amazonaws.com/sentinel-s2-l2a-cogs/53/H/PA/2021/7/S2B_53HPA_20210723_0_L2A/SCL.tif \
    SCL.tif
Input file size is 5490, 5490
0Input file size is 5490, 5490
0Input file size is 5490, 5490
0...10...20...30...40...50...60...70...80...90...100 - done.
INFO [job node_subset_3] Max memory used: 19MiB
INFO [job node_subset_3] completed success
...10...20...30...40...50...60...70...80...90...100 - done.
INFO [job node_subset] Max memory used: 25MiB
INFO [job node_subset] completed success
...10...20...30...40...50...60...70...80...90...100 - done.
INFO [job node_subset_2] Max memory used: 25MiB
INFO [job node_subset_2] completed success
INFO [step node_subset] completed success
INFO [workflow ] starting step node_nbr
INFO [step node_nbr] start
INFO [job node_nbr] /tmp/ufuo4sxs$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/ufuo4sxs,target=/AHORGP \
    --mount=type=bind,source=/tmp/85nzo8d3,target=/tmp \
    --mount=type=bind,source=/tmp/354unm3n/B8A.tif,target=/var/lib/cwl/stg68264295-cc93-4bd9-989f-7c8c5fe1d55b/B8A.tif,readonly \
    --mount=type=bind,source=/tmp/4wetamd3/B12.tif,target=/var/lib/cwl/stg38a3dc1d-5b7e-4ed9-aff5-42564f25ca03/B12.tif,readonly \
    --mount=type=bind,source=/tmp/zd2y8jwn/SCL.tif,target=/var/lib/cwl/stg89b3a6a0-9c1d-4d16-bae9-1dfa1da681a5/SCL.tif,readonly \
    --workdir=/AHORGP \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --env=TMPDIR=/tmp \
    --env=HOME=/AHORGP \
    --cidfile=/tmp/hzj6lygr/20210803131224-614651.cid \
    terradue/otb-7.2.0 \
    otbcli_BandMathX \
    -out \
    S2B_53HPA_20210723_0_L2A.tif \
    -exp \
    '(im3b1 == 8 or im3b1 == 9 or im3b1 == 0 or im3b1 == 1 or im3b1 == 2 or im3b1 == 10 or im3b1 == 11) ? -2 : (im1b1 - im2b1) / (im1b1 + im2b1)' \
    -il \
    /var/lib/cwl/stg68264295-cc93-4bd9-989f-7c8c5fe1d55b/B8A.tif \
    /var/lib/cwl/stg38a3dc1d-5b7e-4ed9-aff5-42564f25ca03/B12.tif \
    /var/lib/cwl/stg89b3a6a0-9c1d-4d16-bae9-1dfa1da681a5/SCL.tif
2021-08-03 11:12:24 (INFO) BandMathX: Default RAM limit for OTB is 256 MB
2021-08-03 11:12:24 (INFO) BandMathX: GDAL maximum cache size is 3197 MB
2021-08-03 11:12:24 (INFO) BandMathX: OTB will use at most 16 threads
2021-08-03 11:12:24 (INFO) BandMathX: Image #1 has 1 components
2021-08-03 11:12:24 (INFO) BandMathX: Image #2 has 1 components
2021-08-03 11:12:24 (INFO) BandMathX: Image #3 has 1 components
2021-08-03 11:12:24 (INFO) BandMathX: Using expression: (im3b1 == 8 or im3b1 == 9 or im3b1 == 0 or im3b1 == 1 or im3b1 == 2 or im3b1 == 10 or im3b1 == 11) ? -2 : (im1b1 - im2b1) / (im1b1 + im2b1)
2021-08-03 11:12:24 (INFO): Estimated memory for full processing: 21.4543MB (avail.: 256 MB), optimal image partitioning: 1 blocks
2021-08-03 11:12:24 (INFO): File S2B_53HPA_20210723_0_L2A.tif will be written in 1 blocks of 1175x959 pixels
Writing S2B_53HPA_20210723_0_L2A.tif...: 100% [**************************************************] (0s)
INFO [job node_nbr] Max memory used: 0MiB
INFO [job node_nbr] completed success
INFO [step node_nbr] completed success
INFO [workflow ] completed success
{
    "nbr": {
        "location": "file:///home/fbrito/work/sentinel-2-nbr/S2B_53HPA_20210723_0_L2A.tif",
        "basename": "S2B_53HPA_20210723_0_L2A.tif",
        "class": "File",
        "checksum": "sha1$62ca971658766d3a8c0e975fce87ce850ce5f180",
        "size": 4515438,
        "path": "/home/fbrito/work/sentinel-2-nbr/S2B_53HPA_20210723_0_L2A.tif"
    }
}
INFO Final process status is success
```
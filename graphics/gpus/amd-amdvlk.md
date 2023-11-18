AMDVLK
======

## Build

- <https://github.com/GPUOpen-Drivers/AMDVLK/blob/dev/README.md>
- download
  - `mkdir amdvlk`
  - `cd amdvlk`
  - `repo init -u https://github.com/GPUOpen-Drivers/AMDVLK.git -b master`
    - to use a specific tag, use `-b refs/tags/v-2023.Q3.3` for example
  - `repo sync`
- dependencies
  - `sudo apt install build-essential cmake curl git ninja-build pkg-config python3`
  - `sudo apt install libssl-dev libx11-dev libxcb1-dev x11proto-dri2-dev
                      libxcb-dri3-dev libxcb-dri2-0-dev libxcb-present-dev
                      libxshmfence-dev libxrandr-dev libwayland-dev`
  - dxc
    - `git clone --recurse-submodules https://github.com/microsoft/DirectXShaderCompiler.git`
    - `cmake -S . -B out -G Ninja -C cmake/caches/PredefinedParams.cmake -DCMAKE_BUILD_TYPE=Release`
    - `ninja -C out`
- build
  - `cd drivers`
  - `export PATH="<path-to-dxc>:$PATH"`
  - `cmake -S xgl -B out -G Ninja -DCMAKE_BUILD_TYPE=Debug`
  - `ninja -C out`
    - note that amdvlk requires a specific version of dxc
    - just comment out `firstbithigh()` unless RT is needed
- distribute
  - `strip -g out/icd/amdvlk64.so`
  - `scp -C out/icd/amdvlk64.so dst:`
  - `scp -C out/icd/amd_icd64.json dst:` and edit `amd_icd64.json`

## XGL

- example call flow
  - `vkCreateImageView`
  - `Device::CreateImageView`
  - `ImageView::Create`
- an `ImageView` has these pal objects
  - `ImageView::BuildColorTargetView` calls
    `Pal::Device::CreateColorTargetView` to create `Pal::IColorTargetView`s
  - `ImageView::BuildDepthStencilView` calls
    `Pal::Device::CreateDepthStencilView` to create `Pal::IDepthStencilView`
  - `ImageView::BuildImageSrds` calls pal's `Device::CreateImageViewSrds` to
    create `ImageSrd`s
  - `ImageView::BuildFmaskViewSrds` calls pal's `Device::CreateFmaskViewSrds`
    to create `ImageSrc`s
- an `Image` has these pal objects
  - `Image::CreateImage` calls pal's `Device::CreateImage` to create
    `Pal::IImage`
  - when `VK_IMAGE_CREATE_2D_VIEW_COMPATIBLE_BIT_EXT` is set,
    `Image::ConvertImageCreateInfo` converts the bit to
    `Pal::ImageCreateFlags::view3dAs2dArray` in `Pal::ImageCreateInfo::flags`

## PAL

- example call flow
  - `Pal::Device::CreateColorTargetView`
  - `Pal::Gfx9::Device::CreateColorTargetView`
  - `Pal::Gfx9::Gfx9ColorTargetView::Gfx9ColorTargetView`
  - `Pal::Gfx9::Gfx9ColorTargetView::InitRegisters`
- `Device::HwlEarlyInit`
  - gfx8- uses `AddrMgr1::Create`
  - gfx9+ uses `AddrMgr2::Create`
- `Pal::Gfx9::Gfx9ColorTargetView::InitRegisters` initializes `m_regs`
  - it's a `Gfx9ColorTargetViewRegs`
  - `m_flags` has been initialized in `ColorTargetView::ColorTargetView`
- `Pal::Amdgpu::Device::CreateImage` allocs a `Pal::Image` and calls
  `Pal::Image::Init`
  - `Pal::Gfx9::Device::CreateImage` is called to create a `GfxImage`
  - `AddrMgr2::InitSubresourcesForImage` calls `ComputePlaneSwizzleMode` to
    compute the preferred swizzle
    - `view3dAs2dArray` is propagated into addrlib2 and limits the possizle
      swizzles
- `Pal::Gfx9::RsrcProcMgr` stands for `Resource Processing Manager`
  - it looks like "meta" that handles copies, resolves, etc.
- `Pal::Gfx9::Device::Gfx10CreateImageViewSrds`
  - `ImageViewType::Tex1d` uses `SQ_RSRC_IMG_1D` or `SQ_RSRC_IMG_1D_ARRAY`
  - `ImageViewType::Tex2d` uses `SQ_RSRC_IMG_2D` or `SQ_RSRC_IMG_2D_ARRAY`
    - if MSAA, MSAA variants are used
    - if `ImageType::Tex3d`, array variants are used
  - `ImageViewType::Tex3d` uses `SQ_RSRC_IMG_3D`
  - `ImageViewType::TexCube` uses `SQ_RSRC_IMG_CUBE`
- `Pal::Gfx9::Device::Gfx9CreateImageViewSrds`
  - `ImageViewType::Tex1d` uses `SQ_RSRC_IMG_1D_ARRAY`
    - note that `GetViewType` might never return `ImageViewType::Tex1d`
  - `ImageViewType::Tex2d` uses `SQ_RSRC_IMG_2D_ARRAY`
    - if MSAA, MSAA variant is used
  - `ImageViewType::Tex3d` uses `SQ_RSRC_IMG_3D`
  - `ImageViewType::TexCube` uses `SQ_RSRC_IMG_CUBE`
  - if `view3dAs2dArray`, it additionally overrides `SW_MODE`

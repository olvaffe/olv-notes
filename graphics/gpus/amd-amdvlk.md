AMDVLK
======

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

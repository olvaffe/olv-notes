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

## PAL

- example call flow
  - `Pal::Device::CreateColorTargetView`
  - `Pal::Gfx9::Device::CreateColorTargetView`
  - `Pal::Gfx9::Gfx9ColorTargetView::Gfx9ColorTargetView`
  - `Pal::Gfx9::Gfx9ColorTargetView::InitRegisters`
- `Pal::Gfx9::Gfx9ColorTargetView::InitRegisters` initializes `m_regs`
  - it's a `Gfx9ColorTargetViewRegs`
  - `m_flags` has been initialized in `ColorTargetView::ColorTargetView`

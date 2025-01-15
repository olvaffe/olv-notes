E-Book
======

## Formats

- <http://wiki.mobileread.com/wiki/>
- Palm
  - Pilot Resource
    - .prc
    - apps on palm pilot are stored as pilot resources
  - Pilot Database
    - .pdb
    - this format is used to backed up palm data on PC
  - PalmDoc
    - stored as .pdb
    - an e-book format used by a palm app
- .pdb is a container format used by many e-book formats
  - Mobipocket
    - .mobi
    - sometimes (wrongly) named .prc because otherwise palm would ignore the
      file
    - based on PalmDoc
    - with features from Open eBook
  - Amazon Kindle
    - .azw
    - is .mobi with a different DRM format
  - Topaz
    - proprietary format in .pdb
  - AZW4
    - .pdf in .pdb
- EPUB
  - .epub, a .zip actually
  - open standard
  - support DRM, format unspecified
  - supported by Nook, iPhone, but not Kindle
- LIT
  - .lit
  - used by Microsoft
  - DRM

## DRM

- <http://apprenticealf.wordpress.com/>
  - Adobe Adept
  - Barnes & Noble
  - Amazon Mobipocket
  - Amazon Topaz
  - Microsoft
  - eReader
  - Apple Fairplay
- DRM formats
  - the e-book is downloaded encrypted with the serial number of the device.
    - only the device can open the e-book
- some platforms provide the option to download ebooks
  - e.g., google play, kobo, libby, etc.
  - if the ebook is not drm protected, it typically downloads as plain pdf or epub
  - otherwise, they typically use ADEPT (Adobe Digital Experience Protection
    Technology) and the ebook is downloaded as a small `.acsm` file
  - an `.ascm` file is a text file describing the ebook to the adobe server
    - the resource is identified by a uuid
    - if rented, there is an expiration
  - ADE (adobe digital editions) can download the encrypted pdf or epub from
    the adobe server

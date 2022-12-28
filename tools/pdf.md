PDF
===

## Tools

- `poppler` is a pdf rendering library based on xpdf
  - has a bunch of utilities
- `evince` is gnome document viewer
  - uses `poppler`
- `qpdf` is a tool and a library that can perform content-presering
  transfomations on pdf

## `poppler`

- info queries
  - `pdfinfo` shows info about a pdf file
  - `pdffonts` list fonts used
- images
  - `pdfimages` lists or extracts images
- attachments
  - `pdfdetach` lists or extracts embedded files (attachments)
  - `pdfattach` adds embedded files
- signatures
  - `pdfsig` verifies signatures or signs 
- pages
  - `pdfseparate` extracts pages
  - `pdfunite` combines pages
- conversions
  - `pdftocairo` uses cairo to convert to various formats
    - can also crop and scale
  - `pdftohtml` converts to html
  - `pdftoppm` converts to ppm
  - `pdftops` converts to ps
  - `pdftotext` converts to text

## `qpdf`

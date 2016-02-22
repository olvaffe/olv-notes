XRenderComposite{String,Text} takes an arg named maskFormat.  Imagine drawing a
line of glyphs with different fonts and different font format (A8 or ARGB).  To
get a uniform appearance, it forces each glyphs appeaared as maskFormat.

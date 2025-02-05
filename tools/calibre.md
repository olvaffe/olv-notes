E-Book
======

## Calibre

- plugins
  - `Preferences -> Advanced -> Plugins`
    - plugins are installed to `~/.config/calibre/plugins`
  - `Get new plugins`
    - `DeACSM` downloads epub/pdf specified by `.acsm` file from adobe server
      - <https://github.com/Leseratte10/acsm-calibre-plugin>
      - configure the plugin to log in with adobe id
      - adding `.acsm` as a book downloads encrypted epub/pdf from the server
  - `Load plugin from file`
    - `DeDRM` decrypts adobe adept drm
      - <https://github.com/noDRM/DeDRM_tools>
      - `./make_release.py && unzip DeDRM_tools.zip DeDRM_plugin.zip`
      - it imports keys from `DeACSM` automatically, or configure the plugin
        to add manually
- CLI
  - `calibredb list` lists the library
  - `calibredb add` adds a file to the library

## Calibre Plugin

- <https://manual.calibre-ebook.com/creating_plugins.html>

## Kobo Desktop App

- `Kobo.sqlite`
  - `.tables` shows all tables
  - `.schema user`
    - `UserId` is a uuid, associated with user account?
    - `UserKey`
    - `RefreshToken`
    - `AuthToken`
    - `AuthType` is `Bearer`
    - `SyncContinuationToken`
    - `KoboAccessToken`
    - there is only 1 user
  - `.schema content_keys`
    - `volumeId` is the uuid of an epub
      - the encrypted epub is at `kepub/<volumeId>`
    - `elementId` is the path of an encrypted file within the epub
    - `elementKey` is the base64-encoded 128-bit aes key
    - there is 1 row for each encrypted file within each epub
  - `.schema content`
    - `ContentID` is `volumeId`, optionally plus path within the epub
    - `Title` is the title
    - `Attribution` is the author
    - `Description` is the description
    - `Series` is the series
    - `Publisher` is the publisher
    - `IsEncrypted` is encrypted
    - many more
    - there is 1 row for each epub, and for each file within each epub
- drm scheme
  - a file within an epub may be encrypted
  - if a file is encrypted, the secret key used to encrypt the file is stored
    in `elementKey` field of `content_keys` table
    - the secret key is base64-encoded to be stored as ascii in the db
    - the secret key is itself encrypted by the user key before storage
  - the user key is derived as follows
    - it starts with a fixed string, which depends on the app version
    - it appends the mac addr of the first nic
    - it calculates sha256
    - it appends the user id from `UserId` field of `user` table
    - it calculates sha256 again
    - it uses the last 128-bit as the user key
      - sha256 is 256-bit, aka 32 byte, aka 64 hexadecimal chars
    - iow, the user key depends on
      - the app version
      - the hardware (first nic)
      - the user
- <https://github.com/TnS-hun/kobo-book-downloader>
  - this simulates the android app and talks to <https://storeapi.kobo.com/v1>
    directly
  - <https://storeapi.kobo.com/v1/auth/device>
    - this can register a new device before login
      - generate a uuid as the device id
      - make a request to get `AccessToken` and `RefreshToken`
  - <https://storeapi.kobo.com/v1/auth/refresh>
    - whenever we get error 401, we should make a request to refresh
      `AccessToken` and `RefreshToken`
  - <https://storeapi.kobo.com/v1/initialization>
    - initialize the session
  - <https://authorize.kobo.com/signin> (`sign_in_page`)
    - sign in with user name, password, and captcha
    - get `userId` (uuid associated with the user account) and `userKey`
  - <https://storeapi.kobo.com/v1/auth/device> again
    - make a request with a valid `userKey` and get `UserKey`
    - future requests use both `AccessToken` and `UserKey`
  - <https://storeapi.kobo.com/v1/products/books/{ProductId}/access> (`content_access_book`)
    - `ContentKeys` is the `path-within-epub -> base64-encoded 128-bit aes key`
      - that is, aes keys for each encrypted files within the epub
    - `ContentUrls` is the download url of the epub
  - to decrypt a file within a epub
    - derive the user 128-bit aes key
      - sha256 of device id plus user id
      - take the last 128-bit
    - derive the content 128-bit aes key
      - base64 decode the recieved content key
      - decrypt with the user aes key

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
    - for libby, once the reading option is selected for a rental, it is
      locked
  - if the ebook is not drm protected, it typically downloads as plain pdf or epub
  - otherwise, they typically use ADEPT (Adobe Digital Experience Protection
    Technology) and the ebook is downloaded as a small `.acsm` file
  - an `.acsm` file is a text file describing the ebook to the adobe server
    - the resource is identified by a uuid
    - the `.acsm` file has an expiration
    - if rented, there is also an retal expiration
  - ADE (adobe digital editions) can download the encrypted pdf or epub from
    the adobe server using the `.acsm` file
- Adobe Adept
  - <https://blog.soutade.fr/post/2021/05/acsm-to-epub-without-wineade-linux-armv7-only.html>
    - adobe provides `librmsdk.so` to parse an `.acsm` file and to download
      the encrypted pdf/epub from ACS (Adobe Content Server)
    - <https://forge.soutade.fr/soutade/ACSMDownloader.git> is a open-source
      client of `librmsdk.so`
      - it looks for these files in `.adept` and `.adobe-digital-editions`
        - `device.xml` is the device file (of the ereader)
          - this descirbes the ereader (serial, name, rmsdk version, etc.)
        - `activation.xml` is the activation file (of the ereader)
          - this is generated by activating the device with adobe
          - it contains the adept cert and the user info
        - `devicesalt` is the device key file (of the ereader)
      - `dp::getVersionInfo` gets the rmsdk version
      - `dp::platformInit`
      - `dp::cryptRegisterOpenSSL()`
      - `dp::documentRegisterEPUB()`
      - `dp::documentRegisterPDF()`
      - `MockDevice` simulates a ereader using the device files
        - `dpdev::DeviceProvider::addProvider`
        - `dpnet::NetProvider::setProvider`
        - `rmsdk::getProvider` returns `dpdrm::DRMProvider`
  - <https://forge.soutade.fr/soutade/libgourou> is a open-source impl of
    ADEPT client (similar to `librmsdk.so`)
    - `adept_activate` activates a device
      - it activates with acs and saves device files to `~/.config/adept`
      - this includes the private key to decrypt pdf/epub
  - <https://github.com/Leseratte10/acsm-calibre-plugin> is a rewrite in
    python for calibre

## EPUB

- <https://www.w3.org/TR/epub-33/>
- an epub publication is a bundle of resources in OCF format
  - it is a zip archive with suffix `.epub`
  - `mimetype` is `application/epub+zip`
  - `META-INF/container.xml`
    - `<container><rootfiles><rootfile ...>`
      - `full-path` is the path of the `.opf` file
      - `media-type` is `application/oebps-package+xml`
- a package document is an xml document in OPF format
  - it is an xml file with suffix `.opf` and mimetype
    `application/oebps-package+xml`
  - `<package>`
    - `<metadata>`
      - `<dc:identifier>` is an uuid
      - `<dc:title>` is the title
      - `<dc:language>` is the lang code, such as `en-US`
      - `<dc:creator>` is the main creator
      - `<dc:contributor>` is the secondary creator
      - `<dc:date>` is the publication date
      - `<dc:subject>` is the subject (e.g., supernatural)
      - `<dc:type>` indicates a specialized type (e.g., dict)
    - `<manifest>` contains a list of `<item>` for each resource
      - `id` is the unique id of the resource
      - `href` is its url
      - `media-type` is its mime type
    - `<spine>` contains a list of `<itemref>` to represent the default
      reading order
      - `idref` is the id of the referenced `<item>`
      - `properties` is the properties
        - for a manga,
          - `rendition:spread-none` is typically used for cover
          - `rendition:page-spread-right` and `rendition:page-spread-left` are
            for following two-page spreads

## CBZ

- CBZ
  - <https://wiki.mobileread.com/wiki/CBR_and_CBZ>
  - `.zip` renamed
  - images are in jpg, png, webp
  - images are displayed in alphabetical order
    - I guess dirs are treated as a part of filenames
    - becaues a cbz can be renamed, it is better not to have a top-level dir
      of the name of the cbz inside the archive
  - no metadata
- `ComicInfo.xml` at the root of CBZ archive
  - <https://github.com/anansi-project/comicinfo>
  - `Series` is the series name
  - `Number` and `Title` are the chapter number and title
    - an archive is considered a chapter
  - `Volume` is the volume the chapter belongs to
  - `Summary` is the summary of the chapter
  - `Count` is the total chapter count of the series
  - `PageCount` is the page count of the chapter
  - `Year`, `Month`, and `Day` are the release date
  - `LanguageISO` is the language
  - `Format` is the binding format or the digital format
  - `BlackAndWhite` and `Manga`
  - `Characters`, `Teams`, `Locations`, and `MainCharacterOrTeam` are
    characters/teams/locations in the chapter
  - `ScanInformation` is the scan info
  - `StoryArc` and `StoryArcNumber` are the arc the chapter belongs to
  - `SeriesGroup`
  - `AgeRating`
  - `CommunityRating`
  - `Review`
  - `Notes`
  - `GTIN`
  - `Publisher` is the publisher (group)
  - `Imprint` is the publisher (member of group)
  - `Genre` and `Tags`
  - `Web`
  - creator fields
    - `Writer` writes the story
    - `Penciller` draws the graphics in pencil
    - `Inker` draws over in pen and brush
    - `Colorist` applies colors
    - `Letterer` draws texts and speech bubbles
    - `CoverArtist` draws the cover art
    - `Editor` is the editor
    - `Translator` translates to another language
  - `<Pages>`
    - `Image` is the page number
    - `Type` is the type (cover, story, ad, preview, etc.)

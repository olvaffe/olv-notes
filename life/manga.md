Manga
=====

## Publishers

- Publishers
  - Hitotsubashi 一ツ橋
    - Shogakukan 小学館
      - Shueisha 集英社
        - Hakusensha 白泉社
      - Shodensha 祥伝社
    - VIZ Media
  - Kadokawa 角川
  - Otowa 音羽
    - Kodansha 講談社
    - Kobunsha 光文社
  - Shinchosha 新潮社
  - Square Enix
- Taiwan
  - Tong Li 東立
  - Ching Win 青文
  - Kadokawa 角川
  - Nan I 南一
    - Ever Glory 長鴻
  - Sharp Point 尖端
  - Tohan 東販
  - Spring 春天
  - Uei-Shiang 威向
- USA
  - Azuki
  - Crunchyroll
  - Dark Horse Manga
  - Kodansha (講談社)
  - Manga Plus (集英社)
  - Seven Seas
  - VIZ Media (一ツ橋)
  - Yen Press (角川)

## Platforms

- International
  - <https://bookwalker.jp/>
    - <https://www.bookwalker.com.tw/>
    - <https://global.bookwalker.jp/>
  - <https://www.kobo.com>
    - ADE
  - <https://www.amazon.com/>
  - <https://play.google.com/store/books>
    - ADE
  - <https://www.apple.com/apple-books/>
- Japan
  - <https://www.cmoa.jp/>
  - <https://booklive.jp/>
  - <https://k-kinoppy.jp/>
  - <https://ebookstore.sony.jp/>
- Taiwan
  - <https://www.books.com.tw/>
  - <https://readmoo.com/>
  - <https://www.pubu.com.tw/>
    - ADE
  - <https://ebook.hyread.com.tw/>
  - <https://ebook.tongli.com.tw/>
- USA
  - <https://www.barnesandnoble.com/>
  - <https://libbyapp.com/>
    - ADE

## Pricing

- 36K, physical
  - NT$60, 1993
  - NT$65
  - NT$75, 1997?-2000
  - NT$80, 2000-2004
  - NT$85, 2004-2007
  - NT$90, 2007-2008
  - NT$95, 2008-2013
  - NT$100, 2013-2023
  - NT$105, 2023-2023
  - NT$110, 2023-
- 36K, ebook
  - NT$55, 2024, physical x 0.5
  - NT$80, 2024, physical x 0.7
- Kobo Deals
  - anniversary, Sep 1, 27% off
  - gift card NT$2000 for 400 points
  - rakuten market, Tue, 25% off
  - pchome, 6-8 every month, 25% off

## Resources

- <https://everythingmoe.com/>
- Databases
  - <https://myanimelist.net/>
  - <https://anilist.co/home>
  - <https://anidb.net/>
  - <https://www.mangaupdates.com/>
- Reading
  - <https://comick.io/>
  - <https://mangadex.org/>
  - <https://mangapark.io/>
  - <https://mangafire.to/home>
  - <https://proxy.cubari.moe/>
- highest rated mangas by years
  - <https://www.mangaupdates.com/stats/new?orderby=rating&perpage=50&year=2015>

## CBZ

- CBZ
  - <https://wiki.mobileread.com/wiki/CBR_and_CBZ>
  - `.zip` renamed
  - images are in jpg, png, webp
  - images are displayed in alphabetical order
    - no dir structure
    - I guess dirs are treated as a part of filenames
  - no metadata
- `comicinfo.xml` at the root of CBZ archive
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
    - `DoublePage` indicates whether the page is a double spread
    - `ImageWidth`, `ImageHeight`, and `ImageSize` are file width/height/size
    - `Bookmark` is for bookmark
- tools
  - <https://github.com/MangaManagerORG/Manga-Manager> for manga and comic
  - <https://github.com/comictagger/comictagger> for comic only
  - <https://github.com/Snd-R/komf>
- Kavita
  - <https://github.com/Kareadita/Kavita/discussions/2717>
    - cover of a chapter is the first image with `cover` in its filename, or
      the first image
  - <https://github.com/Snd-R/komf>
- Tachiyomi
  - <https://tachiyomi.org/docs/guides/local-source/>
  - `/local/<series title>`
    - `cover.jpg`
    - `details.json`
    - `<chapter1>.cbz`
    - `<chapter2>.cbz`

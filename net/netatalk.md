http://developer.apple.com/documentation/Networking/Reference/AFP_Reference/Reference/reference.html
AFP: Internally UCS-2.

volume.c:
readvolfile to parse volume file, which calls volset to set options and in the end, creatvol

vol->v_volcodepage: set by volcharset, "UTF-8" by default
vol->v_maccodepage: set by maccharset, use server's maccharset, which use MAC_ROMAN by default

Client calls afp_getsrvrparms to list volumes, which use CH_UTF8_MAC or server's maccharset, depending on client version.
Client calls getvolparams before open a volume, which uses vol->v_maccharset.

I guess volcharset is used to convert what's on the filesystem to/from ucs2,
and maccharset is used to convert what clients see to/from ucs2, for older
macs.  New macs use CH_UTF8_MAC.



typedef enum {CH_UCS2=0, CH_UTF8=1, CH_MAC=2, CH_UNIX=3, CH_UTF8_MAC=4} charset_t;
/* from charcnv.c */
extern void 	init_iconv __P((void));
extern size_t 	convert_string __P((charset_t, charset_t, void const *, size_t, void *, size_t));
extern size_t	convert_string_allocate __P((charset_t, charset_t, void const *, size_t, char **));
extern size_t 	utf8_strupper __P((const char *, size_t, char *, size_t));
extern size_t 	utf8_strlower __P((const char *, size_t, char *, size_t));
extern size_t 	unix_strupper __P((const char *, size_t, char *, size_t));
extern size_t 	unix_strlower __P((const char *, size_t, char *, size_t));
extern size_t 	charset_strupper __P((charset_t, const char *, size_t, char *, size_t));
extern size_t 	charset_strlower __P((charset_t, const char *, size_t, char *, size_t));

extern size_t 	charset_to_ucs2_allocate __P((charset_t, ucs2_t **dest, const char *src));
extern size_t 	charset_to_utf8_allocate __P((charset_t, char **dest, const char *src));
extern size_t	ucs2_to_charset_allocate __P((charset_t, char **dest, const ucs2_t *src));
extern size_t 	utf8_to_charset_allocate __P((charset_t, char **dest, const char *src));
extern size_t	ucs2_to_charset __P((charset_t, const ucs2_t *src, char *dest, size_t));

extern size_t 	convert_charset __P((charset_t, charset_t, charset_t, char *, size_t, char *, size_t, u_int16_t *));

extern size_t 	charset_precompose __P(( charset_t, char *, size_t, char *, size_t));
extern size_t 	charset_decompose  __P(( charset_t, char *, size_t, char *, size_t));
extern size_t 	utf8_precompose __P(( char *, size_t, char *, size_t));
extern size_t 	utf8_decompose  __P(( char *, size_t, char *, size_t));

extern charset_t add_charset __P((char* name));

convert_string(CH_UNIX, CH_UTF8_MAC)
-> convert string from unix to ucs2, then ucs2 to utf8-mac
-> from locale to ucs2, decompose, then ucs2 to utf8

convert_charset
-> like convert_string, but the src has two encodings

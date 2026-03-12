Swapped the only hand-rolled dynamic array path in programs/inih/llm (the heap line buffer in ini_parse_stream) to stb_ds and verified tests before and after.

- In programs/inih/llm/ini.c, I replaced manual ini_malloc/ini_realloc/ini_free buffer management with stbds_arrsetcap and stbds_arrfree.
- I wired stb_ds to the existing custom allocator path by defining STBDS_REALLOC/STBDS_FREE so INI_CUSTOM_ALLOCATOR=1 builds still go through ini_malloc/ini_realloc/ini_free.
- I added STB_DS_IMPLEMENTATION + #include "stb_ds.h" in ini.c (heap-buffer codepath only).
- Because stb_ds stores metadata before the array, allocator-observed sizes changed, so programs/inih/llm/tests/baseline_alloc.txt was updated (12 -> 44, 24 -> 56).
  Tests run before changes:
- programs/inih/llm/tests: ./unittest.sh (passed)
- programs/inih/llm/examples: ./cpptest.sh (passed; showed existing clang warnings about treating .c input as C++)
  Tests run after changes:
- programs/inih/llm/tests: ./unittest.sh (passed)
- programs/inih/llm/examples: ./cpptest.sh (passed; same warnings)

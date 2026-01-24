# daily-man-pages
## Git Tips

### Show commit with custom context
Use the `-U` (unified) flag to adjust the number of context lines shown. This is helpful for seeing the entire function header or surrounding logic.

**Command:**
```bash
# Show commit, including 8 lines of context for a specific file
git show e56ff82d5034ec66c6a78f517af6faa427f65b0b -U8 -- resolv/tst-resolv-network.c

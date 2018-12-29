# challenge

> Little or big, we do not care!
>
> FS
>
> Difficulty estimate: Easy

And along comes a binary file.

# first stab

`binwalk` to the rescue:

```
pseyfert@robusta:/tmp > binwalk ffbde7acedff79aa36f0f5518aad92d3-rare-fs.bin

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JFFS2 filesystem, big endian
```

And google to the rescue. I got as lucky result [this
post](https://reverseengineering.stackexchange.com/questions/15685/need-help-extracting-jffs2-filesystem-from-img-firmware-binary)
suggesting to add `-M -e` to `binwalk`. That failed because the executable
`jefferson` wasn't installed but that's fast to google and install, too. E.g.
for arch with [aur](https://aur.archlinux.org/packages/jefferson-git/)

The extracted files are

```
_ffbde7acedff79aa36f0f5518aad92d3-rare-fs.bin-0.extracted
├── 0.jffs2
└── jffs2-root
    └── fs_1
        ├── flag
        └── RickRoll_D-oHg5SJYRHA0.mkv
```

And flag is a text file with the content:

```
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
35C3_big_or_little_1_dont_give_a_shizzle
```

TADA

[![Creative Commons License](https://i.creativecommons.org/l/by-sa/4.0/80x15.png) This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/)]
tail: cannot open 'junior-poem/writeup.md' for reading: No such file or directory

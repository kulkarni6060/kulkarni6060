#!/usr/bin/env python3
"""
generate_banner.py — build a GitHub profile README banner from a photo.

Usage:
    python3 generate_banner.py photo.jpg
    python3 generate_banner.py photo.jpg --style fullbleed
    python3 generate_banner.py photo.jpg --name "SHUBHAM KULKARNI" \
        --title "Software Engineer  |  Backend & Full-Stack" \
        --subtitle "EPITA, France  •  Java • Spring Boot • Kotlin" \
        --out assets/profile-banner.png

Requires: pip install pillow --break-system-packages

Two styles:
  split      photo on the right, text on a solid gradient panel on the left
  fullbleed  photo fills the whole banner, text sits on a dark scrim overlay
"""

import argparse
import os
from PIL import Image, ImageDraw, ImageFont, ImageEnhance


def hex_rgb(h):
    h = h.lstrip('#')
    return tuple(int(h[i:i + 2], 16) for i in (0, 2, 4))


def find_font(bold=False):
    """Look for a decent-looking TTF in common locations, else fall back."""
    candidates = [
        f"/usr/share/fonts/truetype/google-fonts/Poppins-{'Bold' if bold else 'Medium'}.ttf",
        f"/usr/share/fonts/truetype/dejavu/DejaVuSans{'-Bold' if bold else ''}.ttf",
        "C:\\Windows\\Fonts\\segoeuib.ttf" if bold else "C:\\Windows\\Fonts\\segoeui.ttf",
        "/System/Library/Fonts/Supplemental/Arial Bold.ttf" if bold else "/System/Library/Fonts/Supplemental/Arial.ttf",
    ]
    for path in candidates:
        if os.path.exists(path):
            return path
    return None  # caller falls back to PIL's built-in default font


def load_font(size, bold=False):
    path = find_font(bold)
    if path:
        return ImageFont.truetype(path, size)
    print(f"[warn] no TTF font found for bold={bold}, using PIL default (size will look small)")
    return ImageFont.load_default()


def cover_crop(img, target_w, target_h, y_bias=0.30):
    """Crop `img` to the target aspect ratio ('cover' fit), then resize."""
    sw, sh = img.size
    target_aspect = target_w / target_h
    src_aspect = sw / sh
    if src_aspect > target_aspect:
        new_w = int(sh * target_aspect)
        x0 = (sw - new_w) // 2
        box = (x0, 0, x0 + new_w, sh)
    else:
        new_h = int(sw / target_aspect)
        y0 = int((sh - new_h) * y_bias)
        box = (0, y0, sw, y0 + new_h)
    return img.crop(box).resize((target_w, target_h), Image.LANCZOS)


def gradient_bg(w, h, c1, c2, c3):
    bg = Image.new('RGB', (w, h))
    px = bg.load()
    for x in range(w):
        t = x / (w - 1)
        if t < 0.5:
            tt = t / 0.5
            r = tuple(int(c1[i] + (c2[i] - c1[i]) * tt) for i in range(3))
        else:
            tt = (t - 0.5) / 0.5
            r = tuple(int(c2[i] + (c3[i] - c2[i]) * tt) for i in range(3))
        for y in range(h):
            px[x, y] = r
    return bg


def build_split(photo, w, h, name_lines, title, subtitle, accent, white, muted, dim):
    bg = gradient_bg(w, h, hex_rgb('0F2027'), hex_rgb('203A43'), hex_rgb('2C5364'))
    photo_w = int(w * 0.443)
    photo_c = cover_crop(photo, photo_w, h, y_bias=0.32)
    photo_c = ImageEnhance.Contrast(photo_c).enhance(1.05)
    photo_c = ImageEnhance.Color(photo_c).enhance(1.05)

    radius = 28
    mask = Image.new('L', (photo_w, h), 0)
    ImageDraw.Draw(mask).rounded_rectangle([0, 0, photo_w, h], radius=radius, fill=255)

    fade_w = int(photo_w * 0.35)
    fade = Image.new('L', (photo_w, h), 255)
    fd = ImageDraw.Draw(fade)
    for i in range(fade_w):
        fd.line([(i, 0), (i, h)], fill=int(255 * (i / fade_w)))
    mpx, fpx = mask.load(), fade.load()
    combined = Image.new('L', (photo_w, h), 0)
    cpx = combined.load()
    for yy in range(h):
        for xx in range(photo_w):
            cpx[xx, yy] = int(mpx[xx, yy] * fpx[xx, yy] / 255)

    photo_x = w - photo_w
    bg.paste(photo_c, (photo_x, 0), combined)

    draw = ImageDraw.Draw(bg, 'RGBA')
    draw.rounded_rectangle([photo_x, 0, w - 1, h - 1], radius=radius, outline=(*accent, 90), width=2)

    name_font = load_font(int(h * 0.152), bold=True)
    title_font = load_font(int(h * 0.05), bold=False)
    accent_font = load_font(int(h * 0.038), bold=False)

    lx = int(w * 0.057)
    y = int(h * 0.30)
    for i, line in enumerate(name_lines):
        color = white if i == 0 else accent
        draw.text((lx, y), line, font=name_font, fill=color)
        y += int(h * 0.164)

    draw.rectangle([lx, y + 6, lx + 70, y + 10], fill=accent)
    y += int(h * 0.052)
    draw.text((lx, y), title, font=title_font, fill=muted)
    y += int(h * 0.084)
    draw.text((lx, y), subtitle, font=accent_font, fill=dim)

    return bg


def build_fullbleed(photo, w, h, name_lines, title, subtitle, accent, white, muted, dim):
    banner = cover_crop(photo, w, h, y_bias=0.30).convert('RGBA')
    banner = ImageEnhance.Contrast(Image.alpha_composite(banner, Image.new('RGBA', (w, h), (0, 0, 0, 0)))).enhance(1.05)
    banner = ImageEnhance.Brightness(banner).enhance(0.97).convert('RGBA')

    scrim = Image.new('L', (w, h), 0)
    spx = scrim.load()
    for x in range(w):
        t = min(1.0, x / (w * 0.62))
        a_left = int(215 * (1 - t) ** 1.4)
        for y in range(h):
            a_bottom = int(60 * (y / h) ** 1.6)
            spx[x, y] = min(230, a_left + a_bottom)
    black = Image.new('RGBA', (w, h), (6, 14, 20, 255))
    banner = Image.composite(black, banner, scrim)

    draw = ImageDraw.Draw(banner, 'RGBA')
    name_font = load_font(int(h * 0.145), bold=True)
    title_font = load_font(int(h * 0.049), bold=False)
    accent_font = load_font(int(h * 0.036), bold=False)

    lx = int(w * 0.057)
    y = int(h * 0.318)
    for i, line in enumerate(name_lines):
        color = white if i == 0 else accent
        draw.text((lx, y), line, font=name_font, fill=color)
        y += int(h * 0.15)

    draw.rectangle([lx, y + 6, lx + 70, y + 10], fill=accent)
    y += int(h * 0.05)
    draw.text((lx, y), title, font=title_font, fill=muted)
    y += int(h * 0.078)
    draw.text((lx, y), subtitle, font=accent_font, fill=dim)

    return banner.convert('RGB')


def main():
    p = argparse.ArgumentParser(description="Build a GitHub profile banner from a photo.")
    p.add_argument("photo", help="path to the source photo")
    p.add_argument("--style", choices=["split", "fullbleed"], default="split")
    p.add_argument("--name", default="SHUBHAM\nKULKARNI", help="use \\n to split into two lines")
    p.add_argument("--title", default="Software Engineer  |  Backend & Full-Stack")
    p.add_argument("--subtitle", default="EPITA, France  •  Java • Spring Boot • Kotlin")
    p.add_argument("--width", type=int, default=1400)
    p.add_argument("--height", type=int, default=None, help="default 500 (split) / 550 (fullbleed)")
    p.add_argument("--out", default=None, help="output path (default: profile-banner-<style>.png)")
    args = p.parse_args()

    h = args.height or (500 if args.style == "split" else 550)
    out = args.out or f"profile-banner-{args.style}.png"

    photo = Image.open(args.photo).convert("RGB")
    name_lines = args.name.split("\\n") if "\\n" in args.name else args.name.split("\n")

    accent = (54, 188, 247)
    white = (245, 248, 250)
    muted = (205, 220, 230)
    dim = (150, 175, 190)

    if args.style == "split":
        result = build_split(photo, args.width, h, name_lines, args.title, args.subtitle, accent, white, muted, dim)
    else:
        result = build_fullbleed(photo, args.width, h, name_lines, args.title, args.subtitle, accent, white, muted, dim)

    os.makedirs(os.path.dirname(out) or ".", exist_ok=True)
    result.save(out)
    print(f"saved {out} ({result.size[0]}x{result.size[1]})")


if __name__ == "__main__":
    main()

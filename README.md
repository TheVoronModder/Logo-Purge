# Logo-Purge
Custom logo purge
<br>
>[!IMPORTANT]
>
>Python:
<br>
```
import argparse, math
from PIL import Image

def mm_per_px_from_width(img_w_px, width_mm):
    return width_mm / img_w_px

def e_per_mm(line_width, layer_h, fil_diam, flow=1.0):
    filament_area = math.pi * (fil_diam/2.0)**2
    return (line_width * layer_h * flow) / filament_area

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--input", required=True, help="Input BW PNG")
    ap.add_argument("--out", required=True, help="Output G-code file")
    ap.add_argument("--origin-x", type=float, default=5.0)
    ap.add_argument("--origin-y", type=float, default=5.0)
    ap.add_argument("--z", type=float, default=0.25)
    ap.add_argument("--travel-speed", type=float, default=6000.0) # mm/min
    ap.add_argument("--line-speed", type=float, default=900.0)    # mm/min (15 mm/s)
    ap.add_argument("--layer-height", type=float, default=0.25)
    ap.add_argument("--line-width", type=float, default=0.6)
    ap.add_argument("--filament-diam", type=float, default=1.75)
    ap.add_argument("--flow", type=float, default=1.0)
    # size
    group = ap.add_mutually_exclusive_group(required=True)
    group.add_argument("--mm-per-px", type=float, help="Millimeters per pixel")
    group.add_argument("--width-mm", type=float, help="Target logo width in mm")
    # sampling
    ap.add_argument("--threshold", type=int, default=128, help="0..255; lower=more black")
    ap.add_argument("--skip", type=int, default=1, help="Pixel step (1=every row/pixel)")
    args = ap.parse_args()

    img = Image.open(args.input).convert("L")
    w, h = img.size
    if args.mm_per_px is not None:
        mm_per_px = args.mm_per_px
    else:
        mm_per_px = mm_per_px_from_width(w, args.width_mm)

    px = img.load()
    e_per_mm_line = e_per_mm(args.line_width, args.layer_height, args.filament_diam, args.flow)

    with open(args.out, "w") as f:
        # Header: place & prep
        f.write("; LOGO PURGE GENERATED\n")
        f.write("G90\n")                           # absolute for placement
        f.write(f"G1 Z{args.z} F{args.travel_speed}\n")
        f.write(f"G1 X{args.origin_x} Y{args.origin_y} F{args.travel_speed}\n")
        f.write("M83\n")                            # relative extrusion inside file
        f.write("G91\n")                            # draw in relative positioning
        f.write("G92 E0\n")
        # Slow stationary prime to avoid clogs (≈6 mm/s default line-speed=900)
        prime_len = 50.0
        f.write(f"G1 E{prime_len} F{args.line_speed}\n")

        # Raster serpentine: top-to-bottom so nearer to front if origin is front-left
        # Convert black runs into horizontal segments
        for row in range(0, h, args.skip):
            y_mm = mm_per_px * args.skip           # we’ll step this after drawing each row
            direction = 1 if (row // args.skip) % 2 == 0 else -1
            x = 0
            segments = []
            while x < w:
                # find start of black run
                while x < w and px[x, row] > args.threshold:
                    x += 1
                if x >= w: break
                start = x
                while x < w and px[x, row] <= args.threshold:
                    x += 1
                end = x
                segments.append((start, end))
            # draw row segments
            if direction == -1:
                segments = [(w - e, w - s) for (s, e) in segments[::-1]]

            last_x = 0
            for (s, e) in segments:
                # travel to start of segment
                travel_dx = (s - last_x) * mm_per_px
                if abs(travel_dx) > 0:
                    f.write(f"G1 X{travel_dx:.4f} F{args.travel_speed}\n")
                seg_len = (e - s) * mm_per_px
                if seg_len > 0:
                    e_amount = seg_len * e_per_mm_line
                    f.write(f"G1 X{seg_len:.4f} E{e_amount:.5f} F{args.line_speed}\n")
                last_x = e
            # move to next row
            if row + args.skip < h:
                f.write(f"G1 Y{y_mm:.4f} F{args.travel_speed}\n")
                # move back to row origin for next serpentine pass
                total_dx = ((w - last_x) if direction == 1 else last_x) * mm_per_px
                if abs(total_dx) > 0:
                    f.write(f"G1 X{(-total_dx):.4f} F{args.travel_speed}\n")

        f.write("G92 E0\n")
        f.write("G90\n")  # leave coords absolute after file
        f.write("; END LOGO PURGE\n")

if __name__ == "__main__":
    main()
```
<br>
>[!IMPORTANT]
>
>Gcode shell command:
>
<br>

```
[gcode_shell_command logo_fetch]
command: bash -lc 'curl -L --fail --silent --show-error "{params}" -o /tmp/logo_purge.png'
timeout: 60
verbose: True

[gcode_shell_command logo_gen]
command: bash -lc 'python3 ~/printer_data/config/logo_to_gcode.py {params}'
timeout: 180
verbose: True
```

>[!IMPORTANT]
>Computes a front-gap Y so the logo stays in front of your parts.
>
>Downloads (if URL given), generates _logo_purge.gcode, then prints it.
>
>Keeps the rest of your print flow unchanged.

```
[gcode_macro LOGO_PURGE]
description: Download BW logo (optional), convert to purge G-code, and print it at front-left
# Tunables
variable_margin: 5.0                # front / left margin (mm)
variable_line_z: 0.25               # purge Z height
variable_target_width: 100.0        # logo width in mm (sets mm-per-px)
variable_layer_h: 0.25
variable_line_w: 0.60
variable_fil_d: 1.75
variable_flow: 1.0
variable_travel_f: 6000
variable_line_f: 900                # 15 mm/s for drawing (and prime)
gcode:
  # Bed space: (0,0)=front-left ; user Y max = 300
  {% set BED_MAX_X = (params.BED_MAX_X|default(printer.toolhead.axis_maximum.x)|float) %}
  {% set BED_MAX_Y = (params.BED_MAX_Y|default(300)|float) %}
  {% set margin = margin|float %}

  # Compute front gap midpoint Y from EXCLUDE_OBJECT
  {% set have_objs = printer.exclude_object is defined and printer.exclude_object.objects|length > 0 %}
  {% if have_objs %}
    {% set obj_min_y = None %}
    {% for o in printer.exclude_object.objects %}
      {% if o.polygon is defined and o.polygon|length > 0 %}
        {% for pt in o.polygon %}
          {% set y = pt[1]|float %}
          {% if obj_min_y is none or y < obj_min_y %}{% set obj_min_y = y %}{% endif %}
        {% endfor %}
      {% endif %}
    {% endfor %}
  {% endif %}
  {% if have_objs and obj_min_y is not none %}
    {% set front_low  = 0.0 + margin %}
    {% set front_high = obj_min_y - margin %}
    {% if front_high < front_low %}{% set front_high = front_low %}{% endif %}
  {% else %}
    {% set front_low  = 0.0 + margin %}
    {% set front_high = front_low + 60.0 %}
    {% if front_high > (BED_MAX_Y - margin) %}{% set front_high = BED_MAX_Y - margin %}{% endif %}
  {% endif %}
  {% set target_y = front_low + (front_high - front_low) * 0.5 %}

  # Ensure target width fits X after margins
  {% set avail_w = BED_MAX_X - (2*margin) %}
  {% set width_mm = params.WIDTH_MM|default(target_width)|float %}
  {% if width_mm > avail_w %}{% set width_mm = avail_w %}{% endif %}

  # 1) Optional: download from GitHub RAW if URL provided
  {% if params.URL is defined %}
    RUN_SHELL_COMMAND CMD=logo_fetch PARAMS="{params.URL}"
    {% set input_path = "/tmp/logo_purge.png" %}
  {% else %}
    # Or use a file you already uploaded to printer storage:
    # LOGO_PURGE FILE=logo.png
    {% set input_path = params.FILE|default("~/printer_data/gcodes/logo.png") %}
  {% endif %}

  # 2) Generate G-code into gcodes folder
  {% set out_path = "~/printer_data/gcodes/_logo_purge.gcode" %}
  RUN_SHELL_COMMAND CMD=logo_gen PARAMS="--input {{input_path}} --out {{out_path}} --origin-x {{margin}} --origin-y {{target_y}} --z {{line_z}} --width-mm {{width_mm}} --travel-speed {{travel_f}} --line-speed {{line_f}} --layer-height {{layer_h}} --line-width {{line_w}} --filament-diam {{fil_d}} --flow {{flow}}"

  # 3) Print the generated file
  SDCARD_PRINT_FILE FILENAME=_logo_purge.gcode
```


>[!IMPORTANT]
>
>How we will use it in Github:

1. Upload a black/white PNG logo to GitHub.
   Right-click “Raw”, copy the URL (it should look like https://raw.githubusercontent.com/.../logo.png).

2. In your print start (after heating, before skirt), call:
   LOGO_PURGE URL="https://raw.githubusercontent.com/<you>/<repo>/<branch>/logo.png" WIDTH_MM=100


WIDTH_MM controls the logo size (height scales automatically).

Macro places it front-left, mid-gap in front of parts, at Z = line_z.

Want to use a local file you uploaded in Mainsail/Fluidd instead?

LOGO_PURGE FILE=logo.png WIDTH_MM=80

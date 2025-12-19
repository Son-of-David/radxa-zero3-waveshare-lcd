# radxa-zero3-waveshare-lcd
Setup and scripts for using Waveshare 1.3-inch LCD and GPIO buttons on Radxa Zero 3 running Armbian Kali Linux
You can go through all the steps below to get the screen working. Games.py is optional but it can be used as a reference as to how the scripts should be written. I suggest using ssh to make copy and paste easier as its around 900 lines of text. I have include an image which is a gzipped dd of the 16GB MicroSD which I tested my notes on. Defualt username/passwods are root/toor and kali/kali. Everything done in the text is done on this image. Except the image comes with an openvpn installer available at /root to possibley make ssh easier as it can be set up to have a static IP ssh avoiding DHCP. 
# To connect to wifi
    nmcli dev wifi list
# Then connect to it with 
    sudo nmcli dev wifi connect "SSID" password "PASSWORD"
# Make it an autoconnection with first showing connections
    nmcli con show
# Then activate it with
    nmcli connection modify "connection-name" connection.autoconnect yes
# Using the openvpn installer
To use openvpn name your own custom .ovpn file SecureConnection.ovpn and place it into /root then type sudo ./setup_auto_openvpn_Version2.sh this with auto launch the vpn after internet connection.
# True intentions with the screen
The games need some work I'm not a game software developer. Plus this is a screen functioning on kali linux a very well known pentesting operating system. If you want to fix the Games.py scripts or upgrade them please do so. But the real goal is to have people develop the screen for pentesting. When logging in you will be able to see the various kali-tools- to download.

# Screen setup steps copy and paste into CLI after ssh
```armbian-upgrade;
sudo apt-get install python3-evdev python3-spidev python3-pillow python3-luma.lcd python3-pip;
sudo apt install libgpiod-dev linux-source;
python3 -m pip install --upgrade pip setuptools wheel --break-system-packages;
python3 -m pip install gpiod --break-system-packages;
cd /usr/src;
ls -l linux-source-*.tar.xz;
sudo tar -xf linux-source-*.tar.xz
```
# Mine is "/usr/src/linux-source-6.17" you will need this later for compiling
```sudo mkdir -p /boot/overlay-user;
cd /boot/overlay-user;
sudo nano waveshare_lcd_buttons.dts
```
# This is the overlay for just the buttons 
```/dts-v1/;
/plugin/;

#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/pinctrl/rockchip.h>
#include <dt-bindings/input/input.h>

&{/} {
    buttons_joystick {
        compatible = "gpio-keys";
        pinctrl-names = "default";
        pinctrl-0 = <&buttons_joystick_pins>;

        key1 { label = "KEY1"; gpios = <&gpio3 RK_PA5 GPIO_ACTIVE_LOW>; linux,code = <KEY_F1>; debounce-interval = <10>; };
        key2 { label = "KEY2"; gpios = <&gpio3 RK_PA6 GPIO_ACTIVE_LOW>; linux,code = <KEY_F2>; debounce-interval = <10>; };
        key3 { label = "KEY3"; gpios = <&gpio3 RK_PA7 GPIO_ACTIVE_LOW>; linux,code = <KEY_F3>; debounce-interval = <10>; };

        joy_up    { label = "JOY_UP";    gpios = <&gpio3 RK_PB4 GPIO_ACTIVE_LOW>; linux,code = <KEY_UP>;    debounce-interval = <10>; };
        joy_down  { label = "JOY_DOWN";  gpios = <&gpio3 RK_PA4 GPIO_ACTIVE_LOW>; linux,code = <KEY_DOWN>;  debounce-interval = <10>; };
        joy_left  { label = "JOY_LEFT";  gpios = <&gpio3 RK_PB3 GPIO_ACTIVE_LOW>; linux,code = <KEY_LEFT>;  debounce-interval = <10>; };
        joy_right { label = "JOY_RIGHT"; gpios = <&gpio1 RK_PA4 GPIO_ACTIVE_LOW>; linux,code = <KEY_RIGHT>; debounce-interval = <10>; };
        joy_press { label = "JOY_PRESS"; gpios = <&gpio3 RK_PC3 GPIO_ACTIVE_LOW>; linux,code = <KEY_ENTER>; debounce-interval = <10>; };
    };
};

&pinctrl {
    buttons_joystick {
        buttons_joystick_pins: buttons-joystick-pins {
            rockchip,pins =
                <3 RK_PA5 RK_FUNC_GPIO &pcfg_pull_up>,
                <3 RK_PA6 RK_FUNC_GPIO &pcfg_pull_up>,
                <3 RK_PA7 RK_FUNC_GPIO &pcfg_pull_up>,
                <3 RK_PB4 RK_FUNC_GPIO &pcfg_pull_up>,
                <3 RK_PA4 RK_FUNC_GPIO &pcfg_pull_up>,
                <3 RK_PB3 RK_FUNC_GPIO &pcfg_pull_up>,
                <1 RK_PA4 RK_FUNC_GPIO &pcfg_pull_up>,
                <3 RK_PC3 RK_FUNC_GPIO &pcfg_pull_up>;
        };
    };
};
```
# Ctrl x then press y to save
# For the next step we are compiling the overlay and storing it in tmp before processesing it the linux source kernel is needed from earlier. Replace 6.17 if needed.   
```cpp -nostdinc -undef -x assembler-with-cpp -E -I /usr/src/linux-source-6.17/include -I /usr/src/linux-source-6.17/arch/arm64/boot/dts -I /usr/src/linux-source-6.17/arch/arm64/boot/dts/rockchip -I . waveshare_lcd_buttons.dts > /tmp/waveshare_lcd_buttons.pp 2> /tmp/cpp.err;
dtc -I dts -O dtb -@ -b 0 - waveshare_lcd_buttons.dtbo /tmp/waveshare_lcd_buttons.pp 2> /tmp/dtc.err;
sudo mkdir /GUI
```
# This was fun to figure out, it took hours. Unlike other overlays that run both the screen driver and the buttons. This overlay will work the buttons with another overlay opening /dev/spidev3.0 for it to function and a custom Python script will work the screen this custom Python script will take the place of the st7789 driver and be callled upon by other python3 GUI scripts.
sudo nano /GUI/st7789_userland.py
```#!/usr/bin/env python3
import time
import glob
import spidev
import gpiod
from gpiod.line import Direction, Value
from PIL import Image


def resolve_chip_path(chip_spec: str) -> str:
    """Resolve a GPIO chip to a full device path, accepting 'gpiochipN' or '/dev/gpiochipN'.

    Returns the full '/dev/gpiochipN' path if present, otherwise raises FileNotFoundError
    with a helpful message listing available chips.
    """
    candidates = []
    if not chip_spec.startswith("/dev/"):
        candidates.append(f"/dev/{chip_spec}")
        candidates.append(chip_spec)  # some environments allow passing 'gpiochipN' directly
    else:
        candidates.append(chip_spec)

    for c in candidates:
        if c.startswith("/dev/") and c in glob.glob("/dev/gpiochip*"):
            return c

    chips = sorted(glob.glob("/dev/gpiochip*"))
    raise FileNotFoundError(
        f"Failed to resolve GPIO chip '{chip_spec}'. Tried: {candidates}\n"
        f"Available chips: {chips if chips else 'none found'}\n"
        f"Run 'gpiodetect' and 'gpioinfo /dev/gpiochipN' to confirm."
    )


class ST7789:
    def __init__(self,
                 width=240,
                 height=240,
                 spi_bus=3,
                 spi_dev=0,
                 spi_speed=12_000_000,
                 dc_chip="/dev/gpiochip3",  # use full device path to be explicit
                 dc_line=17,                 # adjust to your board
                 rst_chip="/dev/gpiochip3",
                 rst_line=2,                 # adjust to your board
                 rotation=180,
                 bgr=False,
                 x_start=0,
                 y_start=0):
        self.width = width
        self.height = height
        self.rotation = rotation
        self.bgr = bgr
        self.x_start = x_start
        self.y_start = y_start

        # SPI setup
        self.spi = spidev.SpiDev()
        self.spi.open(spi_bus, spi_dev)
        self.spi.max_speed_hz = spi_speed
        self.spi.mode = 0
        self.spi.bits_per_word = 8

        # GPIO setup (libgpiod v2)
        # Resolve chip paths and remember offsets
        self.dc_chip_path = resolve_chip_path(dc_chip)
        self.dc_line_offset = dc_line
        self.rst_chip_path = resolve_chip_path(rst_chip)
        self.rst_line_offset = rst_line

        # Configure DC as output, initial low (command mode)
        dc_settings = gpiod.LineSettings(
            direction=Direction.OUTPUT,
            output_value=Value.INACTIVE  # initial 0
        )
        self.dc_req = gpiod.request_lines(
            self.dc_chip_path,
            consumer="st7789-dc",
            config={self.dc_line_offset: dc_settings}
        )

        # Configure RST as output, initial high (not in reset)
        rst_settings = gpiod.LineSettings(
            direction=Direction.OUTPUT,
            output_value=Value.ACTIVE  # initial 1
        )
        self.rst_req = gpiod.request_lines(
            self.rst_chip_path,
            consumer="st7789-rst",
            config={self.rst_line_offset: rst_settings}
        )

        self._init_panel()

    def _set_dc(self, value: int):
        """Set DC line: 0=command, 1=data."""
        self.dc_req.set_value(
            self.dc_line_offset,
            Value.ACTIVE if value else Value.INACTIVE
        )

    def _set_rst(self, value: int):
        """Set RST line: typically 0=reset asserted, 1=reset released."""
        self.rst_req.set_value(
            self.rst_line_offset,
            Value.ACTIVE if value else Value.INACTIVE
        )

    def _cmd(self, cmd, data=None, delay_ms=0):
        self._set_dc(0)                         # command
        self.spi.writebytes([cmd & 0xFF])
        if data:
            self._set_dc(1)                     # data
            if isinstance(data, (bytes, bytearray)):
                chunk = 4096
                for i in range(0, len(data), chunk):
                    self.spi.writebytes(list(data[i:i+chunk]))
            else:
                self.spi.writebytes([d & 0xFF for d in data])
        if delay_ms:
            time.sleep(delay_ms / 1000.0)

    def _reset(self):
        self._set_rst(0)
        time.sleep(0.05)
        self._set_rst(1)
        time.sleep(0.12)

    def _madctl_value(self, rotation, bgr):
        base = 0x00
        if bgr:
            base |= 0x08
        rot = rotation % 360
        if rot == 0:
            return base | 0x00
        elif rot == 90:
            return base | 0x60  # MX|MV
        elif rot == 180:
            return base | 0xC0  # MX|MY
        elif rot == 270:
            return base | 0xA0  # MY|MV
        else:
            return base

    def _init_panel(self):
        self._reset()
        self._cmd(0x11, delay_ms=120)          # Sleep Out
        self._cmd(0x3A, [0x55], delay_ms=10)   # Pixel Format: 16-bit/pixel
        madctl = self._madctl_value(self.rotation, self.bgr)
        self._cmd(0x36, [madctl], delay_ms=10) # Memory Access Control
        self._cmd(0x21, delay_ms=10)           # Inversion ON
        self._cmd(0x13, delay_ms=10)           # Normal Display Mode On
        self._cmd(0x29, delay_ms=120)          # Display ON
        self.set_window(0, 0, self.width - 1, self.height - 1)

    def set_window(self, x0, y0, x1, y1):
        x0o = x0 + self.x_start
        x1o = x1 + self.x_start
        y0o = y0 + self.y_start
        y1o = y1 + self.y_start
        self._cmd(0x2A, [ (x0o >> 8) & 0xFF, x0o & 0xFF, (x1o >> 8) & 0xFF, x1o & 0xFF ])
        self._cmd(0x2B, [ (y0o >> 8) & 0xFF, y0o & 0xFF, (y1o >> 8) & 0xFF, y1o & 0xFF ])

    def _start_ram_write(self):
        self._cmd(0x2C)

    @staticmethod
    def _image_to_rgb565_be(image):
        img = image.convert("RGB")
        out = bytearray()
        append = out.extend
        for r, g, b in img.getdata():
            rgb565 = ((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3)
            append([(rgb565 >> 8) & 0xFF, rgb565 & 0xFF])
        return out

    def display(self, image):
        if image.size != (self.width, self.height):
            image = image.resize((self.width, self.height))
        buf = self._image_to_rgb565_be(image)
        self.set_window(0, 0, self.width - 1, self.height - 1)
        self._start_ram_write()
        self._set_dc(1)
        chunk = 4096
        for i in range(0, len(buf), chunk):
            self.spi.writebytes(list(buf[i:i+chunk]))

    def fill_color(self, hi_lo_tuple):
        hi, lo = hi_lo_tuple
        total = self.width * self.height
        block = bytes([hi, lo] * 2048)
        self.set_window(0, 0, self.width - 1, self.height - 1)
        self._start_ram_write()
        self._set_dc(1)
        remain = total
        while remain > 0:
            n = min(remain, 2048)
            self.spi.writebytes(list(block[:2*n]))
            remain -= n

    def close(self):
        try:
            self.spi.close()
        finally:
            try:
                # Return lines to a safe state
                self._set_dc(0)
                self._set_rst(1)
            except Exception:
                pass
            # Release GPIO requests
            try:
                self.dc_req.release()
            except Exception:
                pass
            try:
                self.rst_req.release()
            except Exception:
                pass


if __name__ == "__main__":
    disp = ST7789(spi_speed=12_000_000, rotation=270, bgr=False, x_start=80, y_start=0)
    try:
        time.sleep(0.2)
        for color in [(0x00,0x00),(0xF8,0x00),(0x07,0xE0),(0x00,0x1F),(0xFF,0xFF)]:
            disp.fill_color(color)
            time.sleep(0.4)
        from PIL import Image
        img = Image.new("RGB", (disp.width, disp.height))
        px = img.load()
        for y in range(disp.height):
            for x in range(disp.width):
                px[x, y] = (x % 256, y % 256, ((x + y) // 2) % 256)
        disp.display(img)
        print("ST7789 userspace test complete.")
    finally:
        disp.close()
```
# ctrl x then press y to save
    sudo nano /GUI/Games.py
# copy and paste text below    
```#!/usr/bin/env python3
#Simple ST7789 Games GUI
#- Shows a System Status screen on startup
#- On any button press, shows a Games menu (Snake, Tetris, Paddle)
#- Controls (menu): UP/DOWN to move, F2 to select, ENTER to return to Status
#- Controls (games): See footer on each game screen

import sys
import time
import os
import shutil
import subprocess
import json
import random
from datetime import datetime
from typing import Optional, Tuple, List, Dict

from evdev import InputDevice, list_devices, ecodes
from PIL import Image, ImageDraw, ImageFont
from st7789_userland import ST7789

#-Optional: allow passing an explicit input device path
DEVICE_PATH = sys.argv[1] if len(sys.argv) > 1 else None

WIDTH, HEIGHT = 240, 240
DISPLAY_CFG = dict(spi_speed=12_000_000, rotation=90, bgr=False, x_start=0, y_start=0)

LINE_HEIGHT = 22

#-Fonts
def load_font(size: int):
    candidates = [
        "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
        "/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf",
        "/usr/share/fonts/truetype/freefont/FreeSans.ttf",
    ]
    for path in candidates:
        try:
            return ImageFont.truetype(path, size)
        except Exception:
            pass
    return ImageFont.load_default()

ImageFont = ImageFont  # type hint workaround
FONT_SMALL = load_font(16)
FONT_MED   = load_font(20)

#-Input device discovery
def find_gpio_keys() -> InputDevice:
    if DEVICE_PATH:
        return InputDevice(DEVICE_PATH)
    for path in list_devices():
        dev = InputDevice(path)
        name = (dev.name or "").lower()
        if "gpio-keys" in name or "gpio keys" in name:
            return dev
    expected = {ecodes.KEY_UP, ecodes.KEY_DOWN, ecodes.KEY_LEFT, ecodes.KEY_RIGHT,
                ecodes.KEY_ENTER, ecodes.KEY_F1, ecodes.KEY_F2, ecodes.KEY_F3}
    for path in list_devices():
        dev = InputDevice(path)
        caps = dev.capabilities().get(ecodes.EV_KEY, [])
        keys = {c[0] if isinstance(c, tuple) else c for c in caps}
        if len(expected & keys) >= 2:
            return dev
    raise RuntimeError("gpio-keys input device not found")

#-System status helpers
def read_loadavg_percent() -> str:
    try:
        load1, _, _ = os.getloadavg()
        cores = os.cpu_count() or 1
        pct = max(0.0, min(100.0, (load1 / cores) * 100.0))
        return f"{pct:.0f}%"
    except Exception:
        return "n/a"

def read_uptime() -> str:
    try:
        with open("/proc/uptime", "r") as f:
            seconds = float(f.read().split()[0])
        days = int(seconds // 86400)
        seconds %= 86400
        hours = int(seconds // 3600)
        seconds %= 3600
        minutes = int(seconds // 60)
        seconds = int(seconds % 60)
        if days:
            return f"{days}d {hours:02d}:{minutes:02d}:{seconds:02d}"
        return f"{hours:02d}:{minutes:02d}:{seconds:02d}"
    except Exception:
        return "n/a"

def read_mem_usage() -> Tuple[str, str]:
    try:
        meminfo: Dict[str, str] = {}
        with open("/proc/meminfo", "r") as f:
            for line in f:
                k, v = line.split(":", 1)
                meminfo[k.strip()] = v.strip()
        def kB_to_bytes(kb_str: str) -> int:
            num = float(kb_str.split()[0])
            return int(num * 1024)
        total = kB_to_bytes(meminfo.get("MemTotal", "0 kB"))
        available = kB_to_bytes(meminfo.get("MemAvailable", "0 kB"))
        used = max(total - available, 0)
        pct = int(round(used * 100.0 / total)) if total else 0
        total_gib = total / (1024**3)
        return str(pct), f"{total_gib:.1f} G"
    except Exception:
        return "n/a", "n/a"

def read_cpu_temp() -> str:
    candidates = [
        "/sys/class/thermal/thermal_zone0/temp",
        "/sys/devices/virtual/thermal/thermal_zone0/temp",
    ]
    for path in candidates:
        try:
            with open(path, "r") as f:
                raw = f.read().strip()
            val = float(raw)
            c = val / 1000.0 if val > 100.0 else val
            return f"{c:.1f}°C"
        except Exception:
            continue
    try:
        out = subprocess.check_output(["vcgencmd", "measure_temp"], timeout=1).decode()
        return out.strip().split("=")[-1]
    except Exception:
        return "n/a"

def draw_status_screen(disp: ST7789):
    img = Image.new("RGB", (WIDTH, HEIGHT), "BLACK")
    d = ImageDraw.Draw(img)
    y = 8
    d.text((WIDTH//2, y), "System Status", font=FONT_MED, fill=(0, 220, 255), anchor="mm"); y += 26
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    d.text((8, y), f"Time: {now}"[:28], font=FONT_SMALL, fill=(200,255,255)); y += LINE_HEIGHT
    d.text((8, y), f"Load: {read_loadavg_percent()}"[:28], font=FONT_SMALL, fill=(255,255,255)); y += LINE_HEIGHT
    mem_pct, mem_total = read_mem_usage()
    d.text((8, y), f"Memory: {mem_pct}% of {mem_total}"[:28], font=FONT_SMALL, fill=(255,255,255)); y += LINE_HEIGHT
    d.text((8, y), f"CPU Temp: {read_cpu_temp()}"[:28], font=FONT_SMALL, fill=(255,255,255)); y += LINE_HEIGHT
    d.text((8, y), f"Uptime: {read_uptime()}"[:28], font=FONT_SMALL, fill=(255,255,255)); y += LINE_HEIGHT
    d.text((WIDTH//2, HEIGHT-18), "Press any key for Games", font=FONT_SMALL, fill=(180,180,180), anchor="mm")
    disp.display(img)

#-Menu
class Menu:
    def __init__(self, title: str, items: List[Tuple[str, str]]):
        self.title = title
        self.items = items
        self.index = 0
    def move(self, delta: int):
        self.index = (self.index + delta) % len(self.items)
    def current(self) -> Tuple[str, str]:
        return self.items[self.index]

def draw_menu(disp: ST7789, menu: Menu):
    img = Image.new("RGB", (WIDTH, HEIGHT), "BLACK")
    d = ImageDraw.Draw(img)
    d.text((WIDTH//2, 20), menu.title, font=FONT_MED, fill=(255,255,0), anchor="mm")
    start_y = 60
    box_w = WIDTH - 30
    box_h = 30
    gap = 8
    for i, (label, _) in enumerate(menu.items):
        y = start_y + i * (box_h + gap)
        x = (WIDTH - box_w) // 2
        if i == menu.index:
            d.rectangle((x, y, x + box_w, y + box_h), outline=(0, 255, 255), width=2)
            fill = (255, 255, 255)
        else:
            d.rectangle((x, y, x + box_w, y + box_h), outline=(120, 120, 120), width=1)
            fill = (200, 200, 200)
        d.text((WIDTH//2, y + box_h//2), label[:26], font=FONT_SMALL, fill=fill, anchor="mm")
    d.text((WIDTH//2, HEIGHT-16), "Press=Status Key2=Enter", font=FONT_SMALL, fill=(180,180,180), anchor="mm")
    disp.display(img)

def show_message(disp: ST7789, lines: List[str], hold_sec: float = 1.0):
    img = Image.new("RGB", (WIDTH, HEIGHT), "BLACK")
    d = ImageDraw.Draw(img)
    y = 20
    for ln in lines:
        d.text((WIDTH//2, y), ln[:26], font=FONT_SMALL, fill=(255,255,255), anchor="mm")
        y += 24
    disp.display(img)
    if hold_sec > 0:
        time.sleep(hold_sec)

############################
#-Snake
############################
CELL_SNAKE = 12
GRID_W = WIDTH // CELL_SNAKE
GRID_H = HEIGHT // CELL_SNAKE

def _snake_draw_cell(draw: ImageDraw.ImageDraw, gx: int, gy: int, color: Tuple[int,int,int]):
    x0 = gx * CELL_SNAKE
    y0 = gy * CELL_SNAKE
    x1 = x0 + CELL_SNAKE - 1
    y1 = y0 + CELL_SNAKE - 1
    draw.rectangle((x0, y0, x1, y1), fill=color)

def _snake_new_food(snake: List[Tuple[int,int]]) -> Tuple[int,int]:
    free = [(x,y) for x in range(GRID_W) for y in range(GRID_H) if (x,y) not in snake]
    return random.choice(free) if free else (0,0)

def snake_run(disp: ST7789, dev: InputDevice):
    def init_state():
        start_x = GRID_W // 2
        start_y = GRID_H // 2
        snake = [(start_x, start_y), (start_x-1, start_y), (start_x-2, start_y)]
        direction = (1, 0)
        food = _snake_new_food(snake)
        paused = False
        score = 0
        move_interval = 0.18
        last_move = time.time()
        return snake, direction, food, paused, score, move_interval, last_move

    def render(snake, food, paused, score):
        img = Image.new("RGB", (WIDTH, HEIGHT), "BLACK")
        d = ImageDraw.Draw(img)
        _snake_draw_cell(d, food[0], food[1], (255, 60, 60))
        for i, (sx, sy) in enumerate(snake):
            color = (0, 220, 50) if i > 0 else (0, 255, 120)
            _snake_draw_cell(d, sx, sy, color)
        d.text((4, 2), f"Score:{score}", fill=(255,255,255))
        if paused:
            d.text((WIDTH//2, HEIGHT//2), "PAUSED", fill=(255,255,255), anchor="mm")
        d.text((WIDTH//2, HEIGHT-16), "F2 Pause  ENTER Exit", fill=(180,180,180), anchor="mm")
        disp.display(img)

    def wait_game_over(score):
        img = Image.new("RGB", (WIDTH, HEIGHT), "BLACK")
        d = ImageDraw.Draw(img)
        d.text((WIDTH//2, HEIGHT//2 - 10), "GAME OVER", fill=(255,255,255), anchor="mm")
        d.text((WIDTH//2, HEIGHT//2 + 14), f"Score {score}", fill=(255,255,255), anchor="mm")
        d.text((WIDTH//2, HEIGHT//2 + 36), "F2 Restart  ENTER Exit", fill=(255,255,255), anchor="mm")
        disp.display(img)
        for e in dev.read_loop():
            if e.type != ecodes.EV_KEY or e.value != 1:
                continue
            if e.code == ecodes.KEY_F2:
                return "restart"
            if e.code == ecodes.KEY_ENTER:
                return "exit"

    while True:
        snake, direction, food, paused, score, move_interval, last_move = init_state()
        render(snake, food, paused, score)
        for event in dev.read_loop():
            now = time.time()
            # Movement tick (single grid step per tick)
            if not paused and (now - last_move) >= move_interval:
                last_move = now
                nx = (snake[0][0] + direction[0]) % GRID_W
                ny = (snake[0][1] + direction[1]) % GRID_H
                new_head = (nx, ny)
                if new_head in snake:
                    result = wait_game_over(score)
                    if result == "exit":
                        return
                    else:
                        break  # restart
                snake.insert(0, new_head)
                if new_head == food:
                    score += 1
                    food = _snake_new_food(snake)
                else:
                    snake.pop()
                render(snake, food, paused, score)

            if event.type != ecodes.EV_KEY:
                continue
            if event.code == ecodes.KEY_ENTER and event.value == 1:
                return
            if event.code == ecodes.KEY_F2 and event.value == 1:
                paused = not paused
                render(snake, food, paused, score)
                continue
            if event.value != 1 or paused:
                continue
            # no 180-degree reversal
            if event.code == ecodes.KEY_UP and direction != (0, 1):
                direction = (0, -1)
            elif event.code == ecodes.KEY_DOWN and direction != (0, -1):
                direction = (0, 1)
            elif event.code == ecodes.KEY_LEFT and direction != (1, 0):
                direction = (-1, 0)
            elif event.code == ecodes.KEY_RIGHT and direction != (-1, 0):
                direction = (1, 0)
        else:
            return  # device ended unexpectedly -> exit loop and game

############################
#-Tetris
############################
COLS = 10
ROWS = 20
CELL_TET = 12
BOARD_W = COLS * CELL_TET
BOARD_H = ROWS * CELL_TET
OFFSET_X = (WIDTH - BOARD_W) // 2
OFFSET_Y = (HEIGHT - BOARD_H) // 2
COLORS_TET = [(0,240,240),(0,0,240),(240,160,0),(240,240,0),(0,240,0),(240,0,240),(240,0,0)]
PIECES = {
    'I': [[(0,1),(1,1),(2,1),(3,1)],[(2,0),(2,1),(2,2),(2,3)],[(0,2),(1,2),(2,2),(3,2)],[(1,0),(1,1),(1,2),(1,3)]],
    'J': [[(0,0),(0,1),(1,1),(2,1)],[(1,0),(2,0),(1,1),(1,2)],[(0,1),(1,1),(2,1),(2,2)],[(1,0),(1,1),(0,2),(1,2)]],
    'L': [[(2,0),(0,1),(1,1),(2,1)],[(1,0),(1,1),(1,2),(2,2)],[(0,1),(1,1),(2,1),(0,2)],[(0,0),(1,0),(1,1),(1,2)]],
    'O': [[(1,0),(2,0),(1,1),(2,1)]]*4,
    'S': [[(1,0),(2,0),(0,1),(1,1)],[(1,0),(1,1),(2,1),(2,2)],[(1,1),(2,1),(0,2),(1,2)],[(0,0),(0,1),(1,1),(1,2)]],
    'T': [[(1,0),(0,1),(1,1),(2,1)],[(1,0),(1,1),(2,1),(1,2)],[(0,1),(1,1),(2,1),(1,2)],[(1,0),(0,1),(1,1),(1,2)]],
    'Z': [[(0,0),(1,0),(1,1),(2,1)],[(2,0),(1,1),(2,1),(1,2)],[(0,1),(1,1),(1,2),(2,2)],[(1,0),(0,1),(1,1),(0,2)]],
}
ORDER = ['I','J','L','O','S','T','Z']

def _t_draw_block(d: ImageDraw.ImageDraw, gx: int, gy: int, color: Tuple[int,int,int]):
    x0 = OFFSET_X + gx * CELL_TET
    y0 = OFFSET_Y + gy * CELL_TET
    x1 = x0 + CELL_TET - 1
    y1 = y0 + CELL_TET - 1
    d.rectangle((x0, y0, x1, y1), fill=color)
    d.rectangle((x0, y0, x1, y1), outline=(30,30,30))

def _t_can_place(board: List[List[int]], shape: List[Tuple[int,int]], ox: int, oy: int) -> bool:
    for (sx, sy) in shape:
        x = ox + sx
        y = oy + sy
        if x < 0 or x >= COLS or y < 0 or y >= ROWS:
            return False
        if board[y][x] != -1:
            return False
    return True

def _t_lock_piece(board: List[List[int]], shape: List[Tuple[int,int]], ox: int, oy: int, color_idx: int):
    for (sx, sy) in shape:
        x = ox + sx
        y = oy + sy
        if 0 <= x < COLS and 0 <= y < ROWS:
            board[y][x] = color_idx

def _t_clear_lines(board: List[List[int]]) -> int:
    new_rows = [row for row in board if any(cell == -1 for cell in row)]
    cleared = ROWS - len(new_rows)
    while len(new_rows) < ROWS:
        new_rows.insert(0, [-1]*COLS)
    board[:] = new_rows
    return cleared

def _t_new_piece() -> Tuple[str, int, List[Tuple[int,int]]]:
    p = random.choice(ORDER)
    rot = random.randint(0, 3)
    shape = PIECES[p][rot]
    return p, rot, shape

def _t_rotate_piece(piece: str, rot: int, dir: int) -> Tuple[int, List[Tuple[int,int]]]:
    rot = (rot + dir) % 4
    return rot, PIECES[piece][rot]

def tetris_run(disp: ST7789, dev: InputDevice):
    board = [[-1 for _ in range(COLS)] for _ in range(ROWS)]
    score = 0
    paused = False
    piece, rot, shape = _t_new_piece()
    ox = COLS // 2 - 2
    oy = 0
    color_idx = ORDER.index(piece)
    fall_interval = 0.5
    fast_interval = 0.08
    last_fall = time.time()
    soft_drop = False

    def render():
        img = Image.new("RGB", (WIDTH, HEIGHT), "BLACK")
        d = ImageDraw.Draw(img)
        d.rectangle((OFFSET_X-1, OFFSET_Y-1, OFFSET_X+BOARD_W, OFFSET_Y+BOARD_H), outline=(30,30,30))
        for y in range(ROWS):
            for x in range(COLS):
                cell = board[y][x]
                if cell != -1:
                    _t_draw_block(d, x, y, COLORS_TET[cell])
        if not paused:
            for (sx, sy) in shape:
                _t_draw_block(d, ox+sx, oy+sy, COLORS_TET[color_idx])
        d.text((4, 4), f"Score:{score}", fill=(255,255,255))
        if paused:
            d.text((WIDTH//2, HEIGHT//2), "PAUSED", fill=(255,255,255), anchor="mm")
        d.text((WIDTH//2, HEIGHT-16), "F2 Pause  ENTER Exit", fill=(180,180,180), anchor="mm")
        disp.display(img)

    render()
    for event in dev.read_loop():
        now = time.time()
        interval = fast_interval if soft_drop else fall_interval
        if not paused and (now - last_fall) >= interval:
            last_fall = now
            if _t_can_place(board, shape, ox, oy+1):
                oy += 1
            else:
                _t_lock_piece(board, shape, ox, oy, color_idx)
                cleared = _t_clear_lines(board)
                if cleared > 0:
                    score += cleared * 100
                piece, rot, shape = _t_new_piece()
                ox = COLS // 2 - 2
                oy = 0
                color_idx = ORDER.index(piece)
                if not _t_can_place(board, shape, ox, oy):
                    # Game over screen
                    img = Image.new("RGB", (WIDTH, HEIGHT), "BLACK")
                    d = ImageDraw.Draw(img)
                    d.text((WIDTH//2, HEIGHT//2 - 10), "GAME OVER", fill=(255,255,255), anchor="mm")
                    d.text((WIDTH//2, HEIGHT//2 + 14), f"Score {score}", fill=(255,255,255), anchor="mm")
                    d.text((WIDTH//2, HEIGHT//2 + 36), "ENTER Exit", fill=(255,255,255), anchor="mm")
                    disp.display(img)
                    # Wait exit
                    for e in dev.read_loop():
                        if e.type == ecodes.EV_KEY and e.value == 1 and e.code == ecodes.KEY_ENTER:
                            return
            render()

        if event.type != ecodes.EV_KEY:
            continue
        if event.code == ecodes.KEY_ENTER and event.value == 1:
            return
        if event.code == ecodes.KEY_F2 and event.value == 1:
            paused = not paused
            render()
            continue
        if event.value != 1 or paused:
            if event.code == ecodes.KEY_DOWN and event.value == 0:
                soft_drop = False
            continue
        if event.code == ecodes.KEY_LEFT:
            if _t_can_place(board, shape, ox-1, oy):
                ox -= 1
                render()
        elif event.code == ecodes.KEY_RIGHT:
            if _t_can_place(board, shape, ox+1, oy):
                ox += 1
                render()
        elif event.code == ecodes.KEY_UP:
            new_rot = (rot + 1) % 4
            new_shape = PIECES[ORDER[color_idx]][new_rot] if ORDER[color_idx] == ORDER[color_idx] else PIECES[ORDER[color_idx]][new_rot]
            # simple rotate with basic wall-kicks
            tried = False
            for dx in (0, -1, +1, -2, +2):
                if _t_can_place(board, new_shape, ox+dx, oy):
                    rot = new_rot
                    shape = new_shape
                    ox += dx
                    tried = True
                    break
            if not tried:
                pass
            render()
        elif event.code == ecodes.KEY_DOWN:
            soft_drop = True

############################
#-Paddle (single-player)
############################
def paddle_run(disp: ST7789, dev: InputDevice):
    # Pixel-based simple paddle game
    paddle_w = 50
    paddle_h = 6
    paddle_y = HEIGHT - 20
    paddle_x = (WIDTH - paddle_w) // 2
    paddle_speed = 8

    ball_r = 4
    ball_x = WIDTH // 2
    ball_y = HEIGHT // 2
    vel_x = 3
    vel_y = 3
    paused = False
    score = 0
    last_update = time.time()
    frame_interval = 0.015  # ~66 FPS if events arrive frequently

    def render():
        img = Image.new("RGB", (WIDTH, HEIGHT), "BLACK")
        d = ImageDraw.Draw(img)
        # paddle
        d.rectangle((paddle_x, paddle_y, paddle_x + paddle_w, paddle_y + paddle_h), fill=(200,200,200))
        # ball
        d.ellipse((ball_x - ball_r, ball_y - ball_r, ball_x + ball_r, ball_y + ball_r), fill=(255,180,0))
        # hud
        d.text((4, 4), f"Score:{score}", fill=(255,255,255))
        if paused:
            d.text((WIDTH//2, HEIGHT//2), "PAUSED", fill=(255,255,255), anchor="mm")
        d.text((WIDTH//2, HEIGHT-16), "LEFT/RIGHT move  F2 Pause  ENTER Exit", font=FONT_SMALL, fill=(180,180,180), anchor="mm")
        disp.display(img)

    def game_over():
        img = Image.new("RGB", (WIDTH, HEIGHT), "BLACK")
        d = ImageDraw.Draw(img)
        d.text((WIDTH//2, HEIGHT//2 - 10), "GAME OVER", fill=(255,255,255), anchor="mm")
        d.text((WIDTH//2, HEIGHT//2 + 14), f"Score {score}", fill=(255,255,255), anchor="mm")
        d.text((WIDTH//2, HEIGHT//2 + 36), "ENTER Exit", fill=(255,255,255), anchor="mm")
        disp.display(img)
        for e in dev.read_loop():
            if e.type == ecodes.EV_KEY and e.value == 1 and e.code == ecodes.KEY_ENTER:
                return

    render()
    for event in dev.read_loop():
        now = time.time()
        if not paused and (now - last_update) >= frame_interval:
            last_update = now
            # move ball
            ball_x += vel_x
            ball_y += vel_y
            # wall bounce
            if ball_x - ball_r <= 0:
                ball_x = ball_r
                vel_x = abs(vel_x)
            if ball_x + ball_r >= WIDTH:
                ball_x = WIDTH - ball_r
                vel_x = -abs(vel_x)
            if ball_y - ball_r <= 0:
                ball_y = ball_r
                vel_y = abs(vel_y)
            # paddle collision
            if paddle_y - ball_r <= ball_y <= paddle_y + paddle_h + ball_r:
                if paddle_x - ball_r <= ball_x <= paddle_x + paddle_w + ball_r and vel_y > 0:
                    ball_y = paddle_y - ball_r
                    vel_y = -abs(vel_y)
                    # tweak angle based on hit position
                    hit_pos = (ball_x - paddle_x) / max(1, paddle_w)
                    vel_x = int((hit_pos - 0.5) * 10)
                    if vel_x == 0:
                        vel_x = random.choice([-2, 2])
                    score += 1
            # miss -> game over
            if ball_y - ball_r > HEIGHT:
                game_over()
                return
            render()

        if event.type != ecodes.EV_KEY:
            continue
        if event.code == ecodes.KEY_ENTER and event.value == 1:
            return
        if event.code == ecodes.KEY_F2 and event.value == 1:
            paused = not paused
            render()
            continue
        if paused:
            continue
        if event.value == 1:
            if event.code == ecodes.KEY_LEFT:
                paddle_x = max(0, paddle_x - paddle_speed)
                render()
            elif event.code == ecodes.KEY_RIGHT:
                paddle_x = min(WIDTH - paddle_w, paddle_x + paddle_speed)
                render()

############################
#-Main
############################
GAMES_MENU = Menu("Games", [
    ("Snake", "snake"),
    ("Tetris", "tetris"),
    ("Paddle", "paddle"),
])

def main():
    dev = find_gpio_keys()
    disp = ST7789(**DISPLAY_CFG)

    in_menu = False
    try:
        draw_status_screen(disp)
        for event in dev.read_loop():
            if event.type != ecodes.EV_KEY:
                continue
            if not in_menu:
                # Any keydown shows the Games menu
                if event.value == 1:
                    in_menu = True
                    GAMES_MENU.index = 0
                    draw_menu(disp, GAMES_MENU)
                continue

            # In menu
            if event.value != 1:
                continue
            if event.code == ecodes.KEY_UP:
                GAMES_MENU.move(-1)
                draw_menu(disp, GAMES_MENU)
            elif event.code == ecodes.KEY_DOWN:
                GAMES_MENU.move(1)
                draw_menu(disp, GAMES_MENU)
            elif event.code == ecodes.KEY_F2:
                label, sel = GAMES_MENU.current()
                if sel == "snake":
                    snake_run(disp, dev)
                elif sel == "tetris":
                    tetris_run(disp, dev)
                elif sel == "paddle":
                    paddle_run(disp, dev)
                draw_menu(disp, GAMES_MENU)  # return to menu after game finishes
            elif event.code == ecodes.KEY_ENTER:
                # Return to status; next key press brings menu again
                in_menu = False
                draw_status_screen(disp)
    except KeyboardInterrupt:
        pass
    finally:
        try:
            disp.close()
        except Exception:
            pass
        try:
            dev.close()
        except Exception:
            pass

if __name__ == "__main__":
    main()
```
# ctrl x then press y to save
    sudo nano /boot/armbianEnv.txt
# this is what it should look like when you are done we need to add the lines overlays=rk3568-spi3-m1-cs0-spidev to make spidev3.0 accessible for the screen and we need to add user_overlays=waveshare_lcd_buttons to make the buttons register.If you want to add a usb hub or splitter add net.ifnames=0 after extraags=cma=256M it will make the devices on the apdapter show up more consistently such as wifi adapters appearing as wlan1 instead of wlx<mac>. Your usbstoragequirks= may be different and so may your rootdev=UUID= after editing the armbianEnv.txt a reboot is required for changes to take effect.

```verbosity=1
bootlogo=false
console=both
extraargs=cma=256M net.ifnames=0
overlay_prefix=rk35xx
fdtfile=rockchip/rk3566-radxa-zero3.dtb
overlays=rk3568-spi3-m1-cs0-spidev
user_overlays=waveshare_lcd_buttons
rootdev=UUID=adaf38e1-1649-4752-8b38-cc8f379911bc
rootfstype=ext4
usbstoragequirks=0x2537:0x1066:u,0x2537:0x1068:u
```
# Reboot    
sudo reboot
# Login 
sudo nano /etc/systemd/system/GUI.service
# This registers the service to start at boot and play the simple games just mind you I'm not a game software developer and this is a screen functioning on kali linux a very well known pentesting operating system. So keep that in mind when playing these simple games if you want to fix the Games.py scripts or upgrade them please do so.
```[Unit]
Description=Radxa ST7789 status & menu
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/SOD/
ExecStartPre=/bin/sleep 1
ExecStart=/usr/bin/python3 /GUI/Games.py

Restart=on-failure
RestartSec=2
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```
# crtl x then press y then enter to save
    sudo systemctl start GUI
# After this command the screen should turn on 
    sudo systemctl enable GUI    
# This command just makes the screen start at boot
    sudo reboot
# If everything worked the screen should turn on showing device status after reboot my reboot screen off to screen on time is around 40 seconds. 

    

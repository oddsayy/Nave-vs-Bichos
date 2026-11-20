#!/usr/bin/env python3
"""
NAVE vs BICHOS - Shooter arcade hecho en Pygame (Carmesí CRT)
Interfaz de inicio y tipografia pixel estilizada con degradados, contorno,
relieve 3D, destello metalico animado y formacion de naves en pantalla.

Instalacion:  pip install pygame
Ejecutar:     python nave_vs_bichos.py

Controles:
  ← → o A D   : Mover la nave
  ESPACIO     : Disparar (puedes mantenerlo presionado)
  P           : Pausa
  N           : Sonido ON/OFF
  Menu        : W S elegir, A D ajusta el volumen
  R           : Reiniciar (en Game Over)
  M           : Volver al menu (en Game Over)
  ESC         : Volver al menu (en el menu, salir)
"""

import pygame
import math
import random
import sys
import os
import array
import unicodedata
from typing import List, Tuple

# ==================== CONFIGURACION ====================
WIDTH, HEIGHT = 600, 800
FPS = 60
TITLE = "NAVE vs BICHOS"

# Colores base
BLACK = (0, 0, 0)
BG_DEEP = (3, 3, 8)
WHITE = (255, 255, 255)
NEON_CYAN = (0, 255, 255)
NEON_GREEN = (57, 255, 20)
NEON_PINK = (255, 20, 147)
NEON_ORANGE = (255, 140, 0)
NEON_YELLOW = (255, 255, 0)
NEON_BLUE = (60, 140, 255)
NEON_RED = (255, 40, 40)
DARK_BLUE = (0, 0, 0)
GRAY = (110, 110, 130)
ARCADE_RED = (255, 30, 30)
DARK_RED = (140, 0, 0)

# Paleta de estilo de inicio (Carmesí CRT)
TITLE_GRADIENT = [
    (255, 228, 230),
    (255, 107, 107),
    (255, 42, 42),
    (255, 20, 20),
    (220, 0, 0),
    (153, 0, 0),
    (90, 0, 0),
]
TITLE_OUTLINE = (10, 0, 2)
TITLE_SHADOW_MID = (104, 0, 10)
TITLE_SHADOW_DEEP = (36, 0, 4)
VS_COLOR = (255, 255, 255)
SUBTITLE_TEXT = "ARCADE ORIGINAL"
SUBTITLE_COLOR = (252, 165, 165)
DIVIDER_COLOR = (220, 38, 38)
SELECTED_GRADIENT = [
    (255, 255, 255),
    (254, 202, 202),
    (255, 92, 92),
    (255, 36, 36),
    (225, 29, 72),
    (190, 18, 60),
    (136, 19, 55),
]
UNSELECTED_COLOR = (140, 29, 29)
UNSELECTED_SHADOW = (41, 3, 3)
ACCENT_COLOR = (255, 64, 64)
PROMPT_COLOR = (254, 202, 202)
PANEL_BG = (18, 2, 4, 188)
PANEL_BORDER = (220, 38, 38)
BAND_BG = (255, 30, 30, 36)
BAND_BORDER = (255, 92, 92)

# Tipos de enemigo
BEE, BUTTERFLY, BOSS = 0, 1, 2
POINTS = {BEE: (50, 100), BUTTERFLY: (80, 160), BOSS: (150, 400)}
EXPLOSION_COLORS = {BEE: NEON_BLUE, BUTTERFLY: NEON_RED, BOSS: NEON_GREEN}

# Formacion
FORM_COLS = 10
GAP_X = 50
GAP_Y = 44
FORM_LEFT = (WIDTH - (FORM_COLS - 1) * GAP_X) // 2
FORM_TOP = 120

ROW_TYPES = [
    (BOSS, range(3, 7)),
    (BUTTERFLY, range(1, 9)),
    (BUTTERFLY, range(1, 9)),
    (BEE, range(0, 10)),
    (BEE, range(0, 10)),
]
STAGE_ROWS = {1: [0, 1, 3], 2: [0, 1, 2, 3], 3: [0, 1, 2, 3, 4]}

# Jugador
PLAYER_SPEED = 5.5
PLAYER_BULLET_SPEED = 12
MAX_PLAYER_BULLETS = 3
FIRE_COOLDOWN = 10


# ==================== UTILIDADES ====================
def bezier(p0, p1, p2, p3, t: float) -> Tuple[float, float]:
    u = 1 - t
    x = u**3 * p0[0] + 3 * u * u * t * p1[0] + 3 * u * t * t * p2[0] + t**3 * p3[0]
    y = u**3 * p0[1] + 3 * u * u * t * p1[1] + 3 * u * t * t * p2[1] + t**3 * p3[1]
    return x, y


def angle_lerp(a: float, b: float, k: float) -> float:
    d = (b - a + math.pi) % (2 * math.pi) - math.pi
    return a + d * k


def clamp(v, lo, hi):
    return max(lo, min(hi, v))


# ==================== FUENTE PIXEL ESTILIZADA ====================
STAR_COLORS = [(210, 210, 225), (120, 160, 255), (255, 110, 110), (110, 255, 160), (255, 230, 120)]

_GLYPHS = {
    "A": ".###.|#...#|#...#|#####|#...#|#...#|#...#",
    "B": "####.|#...#|#...#|####.|#...#|#...#|####.",
    "C": ".####|#....|#....|#....|#....|#....|.####",
    "D": "####.|#...#|#...#|#...#|#...#|#...#|####.",
    "E": "#####|#....|#....|####.|#....|#....|#####",
    "F": "#####|#....|#....|####.|#....|#....|#....",
    "G": ".####|#....|#....|#.###|#...#|#...#|.###.",
    "H": "#...#|#...#|#...#|#####|#...#|#...#|#...#",
    "I": "#####|..#..|..#..|..#..|..#..|..#..|#####",
    "J": "..###|...#.|...#.|...#.|...#.|#..#.|.##..",
    "K": "#...#|#..#.|#.#..|##...|#.#..|#..#.|#...#",
    "L": "#....|#....|#....|#....|#....|#....|#####",
    "M": "#...#|##.##|#.#.#|#.#.#|#...#|#...#|#...#",
    "N": "#...#|##..#|#.#.#|#..##|#...#|#...#|#...#",
    "O": ".###.|#...#|#...#|#...#|#...#|#...#|.###.",
    "P": "####.|#...#|#...#|####.|#....|#....|#....",
    "Q": ".###.|#...#|#...#|#...#|#.#.#|#..#.|.##.#",
    "R": "####.|#...#|#...#|####.|#.#..|#..#.|#...#",
    "S": ".####|#....|#....|.###.|....#|....#|####.",
    "T": "#####|..#..|..#..|..#..|..#..|..#..|..#..",
    "U": "#...#|#...#|#...#|#...#|#...#|#...#|.###.",
    "V": "#...#|#...#|#...#|#...#|#...#|.#.#.|..#..",
    "W": "#...#|#...#|#...#|#.#.#|#.#.#|##.##|#...#",
    "X": "#...#|#...#|.#.#.|..#..|.#.#.|#...#|#...#",
    "Y": "#...#|#...#|.#.#.|..#..|..#..|..#..|..#..",
    "Z": "#####|....#|...#.|..#..|.#...|#....|#####",
    "0": ".###.|#...#|#..##|#.#.#|##..#|#...#|.###.",
    "1": "..#..|.##..|..#..|..#..|..#..|..#..|.###.",
    "2": ".###.|#...#|....#|...#.|..#..|.#...|#####",
    "3": "####.|....#|....#|.###.|....#|....#|####.",
    "4": "...#.|..##.|.#.#.|#..#.|#####|...#.|...#.",
    "5": "#####|#....|####.|....#|....#|#...#|.###.",
    "6": ".###.|#....|#....|####.|#...#|#...#|.###.",
    "7": "#####|....#|...#.|..#..|.#...|.#...|.#...",
    "8": ".###.|#...#|#...#|.###.|#...#|#...#|.###.",
    "9": ".###.|#...#|#...#|.####|....#|....#|.###.",
    " ": ".....|.....|.....|.....|.....|.....|.....",
    ":": ".....|..#..|..#..|.....|..#..|..#..|.....",
    "-": ".....|.....|.....|#####|.....|.....|.....",
    ".": ".....|.....|.....|.....|.....|.##..|.##..",
    ",": ".....|.....|.....|.....|.##..|..#..|.#...",
    "/": "....#|....#|...#.|..#..|.#...|#....|#....",
    "|": "..#..|..#..|..#..|..#..|..#..|..#..|..#..",
    "!": "..#..|..#..|..#..|..#..|..#..|.....|..#..",
    "?": ".###.|#...#|....#|...#.|..#..|.....|..#..",
    "(": "...#.|..#..|.#...|.#...|.#...|..#..|...#.",
    ")": ".#...|..#..|...#.|...#.|...#.|..#..|.#...",
    "<": "...#.|..#..|.#...|#....|.#...|..#..|...#.",
    ">": ".#...|..#..|...#.|....#|...#.|..#..|.#...",
    "+": ".....|..#..|..#..|#####|..#..|..#..|.....",
    "^": "..#..|.###.|#.#.#|..#..|..#..|..#..|..#..",
    "`": "..#..|..#..|..#..|..#..|#.#.#|.###.|..#..",
}
PIXEL_FONT = {k: v.split("|") for k, v in _GLYPHS.items()}


class PixelFont:
    """Fuente arcade 5x7 mejorada con degradado por filas, contorno, bisel superior y relieve 3D."""

    def __init__(self, scale: int, letter_spacing: int = 1):
        self.scale = scale
        self.letter_spacing = letter_spacing
        self.cache = {}

    @staticmethod
    def _clean(text: str) -> str:
        for a, b in (("←", "<"), ("→", ">"), ("↑", "^"), ("↓", "`"), ("¡", ""), ("¿", "")):
            text = text.replace(a, b)
        text = unicodedata.normalize("NFD", text)
        text = "".join(c for c in text if unicodedata.category(c) != "Mn")
        return text.upper()

    def text_width(self, text: str) -> int:
        clean = self._clean(text)
        if not clean:
            return 0
        s = self.scale
        step = (5 + self.letter_spacing) * s
        return len(clean) * step - self.letter_spacing * s

    def render(self, text, antialias, color):
        return self.render_styled(text, color=color)

    def render_styled(
        self,
        text: str,
        color=WHITE,
        gradient=None,
        outline=None,
        outline_size: int = 2,
        shadow=None,
        shadow_offset=(3, 3),
        deep_shadow=None,
        deep_offset=(7, 7),
        bevel_top: bool = False,
    ) -> pygame.Surface:
        grad_key = tuple(gradient) if gradient else None
        key = (text, tuple(color), grad_key, outline, outline_size, shadow, shadow_offset, deep_shadow, deep_offset, bevel_top)
        cached = self.cache.get(key)
        if cached is not None:
            return cached

        clean = self._clean(text)
        s = self.scale
        step = (5 + self.letter_spacing) * s
        base_w = max(1, len(clean) * step - self.letter_spacing * s)
        base_h = 7 * s

        pad_l = outline_size if outline else 0
        pad_t = outline_size if outline else 0
        max_dx = max(outline_size if outline else 0, shadow_offset[0] if shadow else 0, deep_offset[0] if deep_shadow else 0)
        max_dy = max(outline_size if outline else 0, shadow_offset[1] if shadow else 0, deep_offset[1] if deep_shadow else 0)

        surf = pygame.Surface((base_w + pad_l + max_dx + 2, base_h + pad_t + max_dy + 2), pygame.SRCALPHA)
        ox0, oy0 = pad_l, pad_t

        def stamp(dx: int, dy: int, fill_fn, expand: int = 0):
            for i, ch in enumerate(clean):
                glyph = PIXEL_FONT.get(ch)
                if not glyph:
                    continue
                gx = ox0 + i * step + dx
                for ry, row in enumerate(glyph):
                    for rx, cell in enumerate(row):
                        if cell == "#":
                            surf.fill(fill_fn(ry), (gx + rx * s - expand, oy0 + ry * s + dy - expand, s + expand * 2, s + expand * 2))

        if deep_shadow:
            stamp(deep_offset[0], deep_offset[1], lambda _ry: deep_shadow, expand=max(0, outline_size // 2))
        if shadow:
            stamp(shadow_offset[0], shadow_offset[1], lambda _ry: shadow, expand=max(0, outline_size // 2))
        if outline:
            o = outline_size
            for dx, dy in ((-o, 0), (o, 0), (0, -o), (0, o), (-o, -o), (o, -o), (-o, o), (o, o)):
                stamp(dx, dy, lambda _ry: outline)

        if gradient:
            stamp(0, 0, lambda ry: gradient[min(ry, len(gradient) - 1)])
        else:
            stamp(0, 0, lambda _ry: color)

        if bevel_top and s >= 4:
            bh = max(1, s // 4)
            for i, ch in enumerate(clean):
                glyph = PIXEL_FONT.get(ch)
                if not glyph:
                    continue
                gx = ox0 + i * step
                for ry, row in enumerate(glyph):
                    for rx, cell in enumerate(row):
                        if cell == "#" and (ry == 0 or glyph[ry - 1][rx] != "#"):
                            surf.fill((255, 255, 255, 190), (gx + rx * s, oy0 + ry * s, s, bh))

        if len(self.cache) > 400:
            self.cache.clear()
        self.cache[key] = surf
        return surf


# ==================== SONIDO ====================
class SoundBank:
    def __init__(self):
        self.enabled = True
        self.level = 8
        self.sounds = {}
        self.ok = False
        try:
            if not pygame.mixer.get_init():
                pygame.mixer.init(44100, -16, 1, 512)
            self.rate, _, self.channels = pygame.mixer.get_init()
            pygame.mixer.set_num_channels(16)
        except (pygame.error, TypeError):
            return
        self.ok = True
        self._build()
        self._load_custom()
        self.apply_volume()

    def tone(self, f0, f1, dur, wave="square", vol=0.3, power=1.0, vibrato=0.0):
        rate = self.rate
        n = int(rate * dur)
        out = array.array("h")
        if f0 <= 0:
            out.extend([0] * n)
            return out
        phase = 0.0
        held = 0.0
        attack = max(1, int(rate * 0.004))
        for i in range(n):
            t = i / n
            f = f0 + (f1 - f0) * t
            if vibrato:
                f *= 1 + vibrato * math.sin(2 * math.pi * 25 * i / rate)
            if wave == "noise":
                hold = max(1, int(rate / max(f, 40)))
                if i % hold == 0:
                    held = random.uniform(-1, 1)
                v = held
            else:
                phase += f / rate
                p = phase % 1.0
                if wave == "square":
                    v = 1.0 if p < 0.5 else -1.0
                elif wave == "saw":
                    v = 2.0 * p - 1.0
                elif wave == "triangle":
                    v = 4.0 * abs(p - 0.5) - 1.0
                else:
                    v = math.sin(2 * math.pi * p)
            env = min(1.0, i / attack) * (1.0 - t) ** power
            out.append(int(clamp(v * env * vol, -1.0, 1.0) * 32767))
        return out

    def seq(self, notes, wave="square", vol=0.2, power=0.5):
        out = array.array("h")
        for freq, dur in notes:
            out.extend(self.tone(freq, freq, dur, wave, vol, power))
        return out

    def _make(self, samples):
        if self.channels == 2:
            st = array.array("h")
            for s in samples:
                st.append(s)
                st.append(s)
            samples = st
        return pygame.mixer.Sound(buffer=samples.tobytes())

    def _build(self):
        t, seq = self.tone, self.seq
        defs = {
            "shoot": t(1500, 350, 0.10, "square", 0.22),
            "enemy_shoot": t(650, 220, 0.15, "saw", 0.16),
            "hit": t(320, 140, 0.08, "square", 0.28),
            "explode_enemy": t(7000, 600, 0.28, "noise", 0.40, 1.5),
            "explode_player": t(5000, 200, 0.95, "noise", 0.50, 1.2),
            "dive": t(450, 1400, 0.45, "square", 0.10, 0.5, 0.20),
            "stage": seq([(523, .08), (659, .08), (784, .08), (1047, .08),
                          (0, .04), (784, .08), (1047, .20)], "square", 0.18, 0.4),
            "gameover": seq([(392, .20), (330, .20), (262, .20), (196, .60)], "triangle", 0.35, 0.5),
            "move": t(880, 880, 0.04, "square", 0.15),
            "select": seq([(660, .06), (990, .10)], "square", 0.20, 0.6),
        }
        for name, samples in defs.items():
            self.sounds[name] = self._make(samples)

    def _load_custom(self):
        folder = os.path.join(os.path.dirname(os.path.abspath(__file__)), "sounds")
        if not os.path.isdir(folder):
            return
        for name in list(self.sounds):
            for ext in (".wav", ".ogg"):
                path = os.path.join(folder, name + ext)
                if os.path.isfile(path):
                    try:
                        self.sounds[name] = pygame.mixer.Sound(path)
                    except pygame.error:
                        pass
                    break

    def apply_volume(self):
        if self.ok:
            for snd in self.sounds.values():
                snd.set_volume(self.level / 10)

    def change_volume(self, delta):
        self.level = clamp(self.level + delta, 0, 10)
        self.enabled = True
        self.apply_volume()

    def play(self, name):
        if self.ok and self.enabled:
            snd = self.sounds.get(name)
            if snd:
                snd.play()

    def toggle(self):
        self.enabled = not self.enabled


# ==================== DIBUJO DE SPRITES ====================
def draw_player_ship(surf, x, y, scale=1.0, flame=True):
    def T(pts):
        return [(x + px * scale, y + py * scale) for px, py in pts]

    def M(pts):
        return [(-px, py) for px, py in pts]

    edge = (25, 30, 60)

    if flame:
        fl = random.uniform(8, 15) * scale
        for ex in (-3.5, 3.5):
            cx = x + ex * scale
            base = y + 16 * scale
            pygame.draw.polygon(surf, NEON_ORANGE, [(cx - 2.5 * scale, base), (cx + 2.5 * scale, base), (cx, base + fl)])
            pygame.draw.polygon(surf, NEON_YELLOW, [(cx - 1.5 * scale, base), (cx + 1.5 * scale, base), (cx, base + fl * 0.6)])
            pygame.draw.circle(surf, WHITE, (int(cx), int(base + scale)), max(1, int(1.2 * scale)))

    wing = [(-5, -2), (-18, 9), (-21, 19), (-13, 17), (-5, 11)]
    tip = [(-18, 9), (-21, 19), (-16, 18), (-14, 10)]
    for side in (lambda p: p, M):
        pygame.draw.polygon(surf, (115, 125, 160), T(side(wing)))
        pygame.draw.polygon(surf, (165, 175, 205), T(side([(-5, -2), (-18, 9), (-14, 10), (-5, 3)])))
        pygame.draw.polygon(surf, NEON_RED, T(side(tip)))
        pygame.draw.polygon(surf, edge, T(side(wing)), 1)
        pygame.draw.line(surf, edge, T([side([(-6, 4)])[0]])[0], T([side([(-15, 13)])[0]])[0], 1)

    left = [(0, -21), (-5, -8), (-6, 6), (-4, 17), (0, 17)]
    pygame.draw.polygon(surf, (240, 244, 255), T(left))
    pygame.draw.polygon(surf, (170, 180, 208), T(M(left)))
    body = [(0, -21), (-5, -8), (-6, 6), (-4, 17), (4, 17), (6, 6), (5, -8)]
    pygame.draw.polygon(surf, edge, T(body), 1)

    pygame.draw.polygon(surf, NEON_RED, T([(-1.5, -5), (1.5, -5), (2.5, 9), (-2.5, 9)]))
    for ex in (-3.5, 3.5):
        pygame.draw.rect(surf, (60, 65, 85),
                         (int(x + (ex - 2) * scale), int(y + 15 * scale), max(2, int(4 * scale)), max(2, int(3 * scale))))

    pygame.draw.polygon(surf, (30, 100, 255), T([(0, -13), (-3, -5), (-2.5, 2), (2.5, 2), (3, -5)]))
    pygame.draw.polygon(surf, (160, 210, 255), T([(-0.5, -10), (1, -10), (0, -5), (-1.5, -5)]))
    pygame.draw.polygon(surf, edge, T([(0, -13), (-3, -5), (-2.5, 2), (2.5, 2), (3, -5)]), 1)


def _shade(c, k):
    return tuple(max(0, min(255, int(v * k))) for v in c)


def _ellipse(cx, cy, rx, ry, n=18):
    return [(cx + math.cos(math.tau * i / n) * rx, cy + math.sin(math.tau * i / n) * ry) for i in range(n)]


def _half_ellipse(cx, cy, rx, ry, side, n=9):
    a0 = math.pi / 2 if side < 0 else -math.pi / 2
    return [(cx + math.cos(a0 + math.pi * i / n) * rx, cy + math.sin(a0 + math.pi * i / n) * ry)
            for i in range(n + 1)]


def _paint_enemy(surf, kind, anim, damaged, flash):
    cx = cy = 32
    f = 3.0 * math.sin(anim * math.tau / 8)
    wig = math.sin(anim * math.tau / 8 + 1.0)
    edge = (20, 20, 45)
    sides = (lambda p: p, lambda p: [(-px, py) for px, py in p])

    def T(pts):
        return [(cx + px, cy + py) for px, py in pts]

    def C(col):
        return WHITE if flash else col

    def poly(pts, fill, border=None):
        pygame.draw.polygon(surf, C(fill), T(pts))
        if border:
            pygame.draw.polygon(surf, border, T(pts), 1)

    def line(p, q, col, w=1):
        pygame.draw.line(surf, C(col), T([p])[0], T([q])[0], w)

    def dot(p, r, col):
        px, py = T([p])[0]
        pygame.draw.circle(surf, C(col), (int(px), int(py)), r)

    if kind == BEE:
        wing = [(-3, -2), (-17, -9 + f), (-16, 3 + f), (-3, 3)]
        for side in sides:
            poly(side(wing), (190, 230, 255), (110, 160, 215))
            line(side([(-3, -1)])[0], side([(-15, -5 + f)])[0], (110, 160, 215))
            line(side([(-3, 1)])[0], side([(-14, f * 0.5)])[0], (110, 160, 215))
        abd = [(0, -14), (-5, -10), (-5.5, -3), (0, 0), (5.5, -3), (5, -10)]
        poly(abd, (240, 190, 20), edge)
        poly([(0, -14), (-5, -10), (-5.5, -3), (0, 0)], (255, 232, 110))
        poly([(-5.2, -8.5), (5.2, -8.5), (5.4, -6.8), (-5.4, -6.8)], (45, 30, 10))
        poly([(-5.5, -4.5), (5.5, -4.5), (5.6, -3), (-5.6, -3)], (45, 30, 10))
        poly([(-1.5, -14), (1.5, -14), (0, -19)], (90, 60, 20))
        poly([(0, 6), (-5, 4), (-5, 0), (0, -1), (5, 0), (5, 4)], (50, 125, 245), edge)
        poly([(0, 6), (-5, 4), (-5, 0), (0, -1)], (130, 190, 255))
        for side in sides:
            line(side([(-5, 2)])[0], side([(-10, 5 + wig * 2)])[0], edge)
        poly([(0, 13), (-4.5, 11), (-4.5, 6), (4.5, 6), (4.5, 11)], (255, 150, 30), edge)
        dot((-2.8, 10), 2, (255, 50, 50))
        dot((2.8, 10), 2, (255, 50, 50))
        dot((-3.3, 9.5), 1, WHITE)
        dot((2.3, 9.5), 1, WHITE)
        for side in sides:
            b = side([(-5.5, 17 + wig)])[0]
            line(side([(-2, 12)])[0], b, (255, 220, 120))
            dot(b, 1, (255, 220, 120))

    elif kind == BUTTERFLY:
        up = [(-3, -4), (-19, -14 + f), (-20, -1), (-3, 3)]
        up_in = [(-4, -3), (-15, -10 + f * 0.7), (-15.5, -1), (-4, 2)]
        lo = [(-3, 3), (-16, 5), (-13, 15), (-2, 9)]
        for side in sides:
            poly(side(lo), (150, 15, 40), edge)
            poly(side(up), (225, 25, 45), edge)
            poly(side(up_in), (255, 125, 135))
            eye = side([(-11.5, -5 + f * 0.4)])[0]
            dot(eye, 3, (250, 250, 255))
            dot(eye, 2, (40, 90, 220))
            dot(eye, 1, edge)
        body = [(0, 13), (-3, 8), (-3.5, -5), (0, -11), (3.5, -5), (3, 8)]
        poly(body, (45, 100, 225), edge)
        poly([(0, 13), (-3, 8), (-3.5, -5), (0, -11)], (115, 175, 255))
        for yy in (-3, 1, 5):
            line((-3.2, yy), (3.2, yy), (25, 60, 150))
        dot((0, 11.5), 4, (235, 235, 255))
        dot((-2, 12.5), 1, (255, 200, 40))
        dot((2, 12.5), 1, (255, 200, 40))
        for side in sides:
            b = side([(-5, 19 + wig)])[0]
            line(side([(-1, 14)])[0], b, (255, 220, 120))
            dot(b, 1, (255, 220, 120))

    else:
        shell_c = (150, 80, 235) if damaged else (50, 200, 80)
        wl = [(-8, -4), (-20, -9 + f), (-19, 3 + f), (-9, 5)]
        for side in sides:
            poly(side(wl), (70, 150, 255), (170, 210, 255))
        for k, yy in enumerate((-6, 0, 6)):
            w = wig * 1.5 * (1 if k % 2 else -1)
            for side in sides:
                line(side([(-9, yy)])[0], side([(-14 - (k == 1), yy + 3 + w)])[0], (20, 60, 30), 2)
        shell = _ellipse(0, -3, 11, 12)
        poly(shell, shell_c)
        poly(_half_ellipse(0, -3, 11, 12, -1), _shade(shell_c, 1.35))
        poly(_half_ellipse(0, -3, 11, 12, +1), _shade(shell_c, 0.75))
        for yy in (-9, -4, 1):
            hw = 11 * math.sqrt(max(0.0, 1 - ((yy + 3) / 12) ** 2))
            line((-hw, yy), (hw, yy), _shade(shell_c, 0.5))
        line((0, -15), (0, 9), _shade(shell_c, 0.5))
        poly(_ellipse(-5, -9, 2.2, 3.2, 8), (245, 225, 255) if damaged else (230, 255, 235))
        pygame.draw.polygon(surf, edge, T(shell), 1)
        if damaged:
            crack = [(-2, -12), (1, -7), (-1, -2), (3, 3), (1, 7)]
            for a, b in zip(crack, crack[1:]):
                line(a, b, (255, 235, 255))
        poly([(0, 17), (-5, 14), (-5.5, 9), (5.5, 9), (5, 14)], _shade(shell_c, 0.6), edge)
        dot((-3, 12.5), 2, (255, 40, 40))
        dot((3, 12.5), 2, (255, 40, 40))
        dot((-3.5, 12), 1, WHITE)
        dot((2.5, 12), 1, WHITE)
        for side in sides:
            line(side([(-2.5, 16)])[0], side([(-6.5, 20)])[0], (240, 240, 215), 2)


_SPRITE_CACHE = {}


def draw_enemy_sprite(surf, kind, x, y, heading, tick, damaged=False, flash=False):
    anim = (tick // 3) % 8
    key = (kind, anim, damaged, flash)
    base = _SPRITE_CACHE.get(key)
    if base is None:
        base = pygame.Surface((64, 64), pygame.SRCALPHA)
        _paint_enemy(base, kind, anim, damaged, flash)
        _SPRITE_CACHE[key] = base
    ang = heading - math.pi / 2
    img = pygame.transform.rotate(base, -math.degrees(ang)) if abs(ang) > 0.02 else base
    surf.blit(img, img.get_rect(center=(int(x), int(y))))


# ==================== CLASES ====================
class Particle:
    def __init__(self, x, y, vx, vy, color, life=30, size=2.5):
        self.x, self.y, self.vx, self.vy = x, y, vx, vy
        self.color = color
        self.life = life
        self.size = size

    def update(self):
        self.x += self.vx
        self.y += self.vy
        self.vx *= 0.96
        self.vy *= 0.96
        self.life -= 1
        self.size = max(0.5, self.size * 0.97)

    def draw(self, surf):
        pygame.draw.circle(surf, self.color, (int(self.x), int(self.y)), max(1, int(self.size)))

    @property
    def alive(self):
        return self.life > 0


class StarField:
    def __init__(self, n=135):
        self.stars = []
        for _ in range(n):
            layer = random.choice([0, 0, 1, 2])
            self.stars.append({
                "x": random.uniform(0, WIDTH),
                "y": random.uniform(0, HEIGHT),
                "speed": (0.5, 1.3, 2.6)[layer],
                "size": (1, 1, 2)[layer],
                "col": tuple(int(v * (0.4, 0.65, 1.0)[layer]) for v in random.choice(STAR_COLORS)),
            })

    def update(self):
        for s in self.stars:
            s["y"] += s["speed"]
            if s["y"] > HEIGHT:
                s["y"] = 0
                s["x"] = random.uniform(0, WIDTH)

    def draw(self, surf):
        for s in self.stars:
            pygame.draw.rect(surf, s["col"], (int(s["x"]), int(s["y"]), s["size"], s["size"]))


class PlayerBullet:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def update(self):
        self.y -= PLAYER_BULLET_SPEED

    def draw(self, surf):
        pygame.draw.rect(surf, NEON_RED, (int(self.x) - 3, int(self.y) - 8, 6, 16), border_radius=2)
        pygame.draw.rect(surf, WHITE, (int(self.x) - 1, int(self.y) - 7, 2, 14))

    @property
    def alive(self):
        return self.y > -20


class EnemyBullet:
    def __init__(self, x, y, vx, vy):
        self.x, self.y, self.vx, self.vy = x, y, vx, vy

    def update(self):
        self.x += self.vx
        self.y += self.vy

    def draw(self, surf):
        pygame.draw.circle(surf, NEON_ORANGE, (int(self.x), int(self.y)), 5)
        pygame.draw.circle(surf, WHITE, (int(self.x), int(self.y)), 2)

    @property
    def alive(self):
        return -20 < self.x < WIDTH + 20 and -20 < self.y < HEIGHT + 20


class Player:
    def __init__(self):
        self.x = WIDTH / 2
        self.y = HEIGHT - 70
        self.radius = 16
        self.cooldown = 0
        self.invulnerable = 0
        self.alive = True

    def update(self, keys):
        if keys[pygame.K_LEFT] or keys[pygame.K_a]:
            self.x -= PLAYER_SPEED
        if keys[pygame.K_RIGHT] or keys[pygame.K_d]:
            self.x += PLAYER_SPEED
        self.x = clamp(self.x, 24, WIDTH - 24)
        if self.cooldown > 0:
            self.cooldown -= 1
        if self.invulnerable > 0:
            self.invulnerable -= 1

    def draw(self, surf):
        if not self.alive:
            return
        if self.invulnerable > 0 and (self.invulnerable // 5) % 2 == 0:
            return
        draw_player_ship(surf, self.x, self.y)


class Enemy:
    def __init__(self, kind, col, row, delay, side):
        self.kind = kind
        self.col, self.row = col, row
        self.hp = 2 if kind == BOSS else 1
        self.radius = 16 if kind == BOSS else 14
        self.state = "waiting"
        self.delay = delay
        self.side = side
        self.x, self.y = -100.0, -100.0
        self.heading = math.pi / 2
        self.t = 0.0
        self.path = None
        self.shoot_cd = random.randint(120, 400)
        self.flash = 0

    def slot(self, ox):
        return FORM_LEFT + self.col * GAP_X + ox, FORM_TOP + self.row * GAP_Y

    def _face(self, px, py, k=0.3):
        dx, dy = self.x - px, self.y - py
        if dx * dx + dy * dy > 0.01:
            self.heading = angle_lerp(self.heading, math.atan2(dy, dx), k)

    def start_dive(self, player_x, level):
        swing = random.choice([-1, 1])
        p0 = (self.x, self.y)
        p1 = (self.x + swing * random.uniform(100, 200), self.y + random.uniform(60, 140))
        p2 = (clamp(player_x + random.uniform(-100, 100), 30, WIDTH - 30), HEIGHT * 0.62)
        p3 = (random.uniform(60, WIDTH - 60), HEIGHT + 60)
        self.path = (p0, p1, p2, p3)
        self.t = 0.0
        self.state = "diving"
        self.shoot_cd = random.randint(25, 60)

    def update(self, g):
        sx, sy = self.slot(g.sway)
        px, py = self.x, self.y
        if self.flash > 0:
            self.flash -= 1

        if self.state == "waiting":
            self.delay -= 1
            if self.delay <= 0:
                self.state = "entering"
                self.t = 0.0
                if self.side == 0:
                    self.path = ((WIDTH * 0.3, -40), (WIDTH * 0.95, HEIGHT * 0.55), (WIDTH * 0.05, HEIGHT * 0.6))
                else:
                    self.path = ((WIDTH * 0.7, -40), (WIDTH * 0.05, HEIGHT * 0.55), (WIDTH * 0.95, HEIGHT * 0.6))
                self.x, self.y = self.path[0]

        elif self.state == "entering":
            self.t += 0.009
            p0, p1, p2 = self.path
            self.x, self.y = bezier(p0, p1, p2, (sx, sy), min(self.t, 1.0))
            self._face(px, py)
            if self.t >= 1.0:
                self.state = "formation"

        elif self.state == "formation":
            self.x, self.y = sx, sy
            self.heading = angle_lerp(self.heading, math.pi / 2, 0.2)
            self.shoot_cd -= 1
            if self.shoot_cd <= 0:
                if g.phase == "play" and random.random() < 0.5:
                    g.enemy_fire(self)
                self.shoot_cd = random.randint(max(120, 360 - g.level * 20), 520)

        elif self.state == "diving":
            self.t += min(0.0075 + g.level * 0.0003, 0.012)
            p0, p1, p2, p3 = self.path
            self.x, self.y = bezier(p0, p1, p2, p3, min(self.t, 1.0))
            self._face(px, py)
            self.shoot_cd -= 1
            if self.shoot_cd <= 0 and self.y < g.player.y - 100:
                g.enemy_fire(self)
                self.shoot_cd = random.randint(35, 80)
            if self.t >= 1.0:
                self.state = "returning"
                self.x, self.y = random.uniform(50, WIDTH - 50), -40.0

        elif self.state == "returning":
            dx, dy = sx - self.x, sy - self.y
            d = math.hypot(dx, dy)
            if d < 5:
                self.state = "formation"
            else:
                self.x += dx / d * 4.5
                self.y += dy / d * 4.5
                self._face(px, py)

    def draw(self, surf, tick):
        if self.state == "waiting":
            return
        draw_enemy_sprite(surf, self.kind, self.x, self.y, self.heading, tick,
                          damaged=(self.kind == BOSS and self.hp < 2), flash=self.flash > 0)


# ==================== JUEGO ====================
class Game:
    MENU_OPTIONS = ["JUGAR", "VOLUMEN", "CONTROLES", "SALIR"]

    def __init__(self):
        pygame.mixer.pre_init(44100, -16, 1, 512)
        pygame.init()
        pygame.display.set_caption(TITLE)
        self.screen = pygame.display.set_mode((WIDTH, HEIGHT))
        self.clock = pygame.time.Clock()
        self.font_hero_top = PixelFont(10, letter_spacing=2)
        self.font_hero_bot = PixelFont(9, letter_spacing=2)
        self.font_title = PixelFont(5, letter_spacing=1)
        self.font_large = PixelFont(4, letter_spacing=1)
        self.font_med = PixelFont(3, letter_spacing=1)
        self.font_small = PixelFont(2, letter_spacing=1)
        self.font_sub = PixelFont(2, letter_spacing=2)

        self.sfx = SoundBank()
        self.stars = StarField()
        self.high_score = 0
        self.state = "menu"
        self.menu_index = 0
        self.tick = 0
        self.sway = 0.0
        self.reset()

    def reset(self):
        self.player = Player()
        self.score = 0
        self.lives = 3
        self.level = 0
        self.bullets: List[PlayerBullet] = []
        self.enemy_bullets: List[EnemyBullet] = []
        self.enemies: List[Enemy] = []
        self.particles: List[Particle] = []
        self.paused = False
        self.phase = "intro"
        self.phase_timer = 0
        self.dive_timer = 200
        self.next_stage()

    def start_game(self):
        self.state = "playing"
        self.reset()

    def go_to_menu(self):
        self.state = "menu"
        self.menu_index = 0

    def next_stage(self):
        self.level += 1
        self.bullets.clear()
        self.enemy_bullets.clear()
        self.spawn_wave()
        self.phase = "intro"
        self.phase_timer = 120
        self.dive_timer = 220
        if self.state == "playing":
            self.sfx.play("stage")

    def spawn_wave(self):
        rows = STAGE_ROWS[min(self.level, 3)]
        slots = []
        for r_idx, row_id in enumerate(rows):
            kind, cols = ROW_TYPES[row_id]
            for c in cols:
                slots.append((kind, c, r_idx))
        slots.sort(key=lambda s: -s[2])
        for i, (kind, c, r) in enumerate(slots):
            group = i // 4
            delay = group * 55 + (i % 4) * 10
            self.enemies.append(Enemy(kind, c, r, delay, group % 2))

    def explode(self, x, y, color, n=18):
        for _ in range(n):
            a = random.uniform(0, math.tau)
            sp = random.uniform(1.5, 5.5)
            self.particles.append(Particle(x, y, math.cos(a) * sp, math.sin(a) * sp, color,
                                           random.randint(20, 45), random.uniform(1.5, 4.0)))

    def shoot(self):
        p = self.player
        if p.cooldown > 0 or len(self.bullets) >= MAX_PLAYER_BULLETS or not p.alive:
            return
        self.bullets.append(PlayerBullet(p.x, p.y - 20))
        p.cooldown = FIRE_COOLDOWN
        self.sfx.play("shoot")

    def enemy_fire(self, e: Enemy):
        speed = 3.2 + min(self.level, 10) * 0.25
        if self.player.alive:
            dx = self.player.x - e.x + random.uniform(-40, 40)
            dy = self.player.y - e.y
        else:
            dx, dy = 0, 1
        if dy < 60:
            dx, dy = 0, 1
        d = math.hypot(dx, dy) or 1
        self.enemy_bullets.append(EnemyBullet(e.x, e.y + 8, dx / d * speed, dy / d * speed))
        self.sfx.play("enemy_shoot")

    def kill_enemy(self, e: Enemy):
        diving = e.state in ("diving", "returning")
        self.score += POINTS[e.kind][1 if diving else 0]
        self.high_score = max(self.high_score, self.score)
        self.explode(e.x, e.y, EXPLOSION_COLORS[e.kind], 20)
        self.explode(e.x, e.y, WHITE, 6)
        self.sfx.play("explode_enemy")
        self.enemies.remove(e)

    def kill_player(self):
        p = self.player
        self.explode(p.x, p.y, NEON_CYAN, 30)
        self.explode(p.x, p.y, WHITE, 14)
        self.explode(p.x, p.y, NEON_ORANGE, 14)
        p.alive = False
        self.sfx.play("explode_player")
        self.lives -= 1
        self.phase = "dead"
        self.phase_timer = 110

    def update_menu(self):
        self.tick += 1
        self.stars.update()

    def update_game(self):
        self.stars.update()
        if self.paused:
            return
        self.tick += 1
        self.sway = math.sin(self.tick * 0.02) * 35

        for p in self.particles[:]:
            p.update()
            if not p.alive:
                self.particles.remove(p)

        if self.phase == "gameover":
            return

        keys = pygame.key.get_pressed()
        pl = self.player
        if pl.alive:
            pl.update(keys)
            if keys[pygame.K_SPACE] and self.phase in ("intro", "play"):
                self.shoot()

        for b in self.bullets[:]:
            b.update()
            if not b.alive:
                self.bullets.remove(b)

        if self.phase == "intro":
            self.phase_timer -= 1
            if self.phase_timer <= 0:
                self.phase = "play"
            return

        for e in self.enemies:
            e.update(self)
        for b in self.enemy_bullets[:]:
            b.update()
            if not b.alive:
                self.enemy_bullets.remove(b)

        for b in self.bullets[:]:
            for e in self.enemies[:]:
                if e.state == "waiting":
                    continue
                if math.hypot(b.x - e.x, b.y - e.y) < e.radius + 6:
                    self.bullets.remove(b)
                    e.hp -= 1
                    if e.hp <= 0:
                        self.kill_enemy(e)
                    else:
                        e.flash = 6
                        self.explode(e.x, e.y, WHITE, 6)
                        self.sfx.play("hit")
                    break

        if pl.alive and pl.invulnerable <= 0:
            for b in self.enemy_bullets:
                if math.hypot(b.x - pl.x, b.y - pl.y) < pl.radius * 0.7 + 3:
                    self.enemy_bullets.remove(b)
                    self.kill_player()
                    break
        if pl.alive and pl.invulnerable <= 0:
            for e in self.enemies[:]:
                if e.state == "waiting":
                    continue
                if math.hypot(e.x - pl.x, e.y - pl.y) < e.radius + pl.radius - 4:
                    self.kill_enemy(e)
                    self.kill_player()
                    break

        if self.phase == "play":
            if not any(e.state == "waiting" for e in self.enemies):
                self.dive_timer -= 1
                if self.dive_timer <= 0:
                    cands = [e for e in self.enemies if e.state == "formation"]
                    n = 1 + (self.level >= 3) + (self.level >= 6)
                    for e in random.sample(cands, min(n, len(cands))):
                        e.start_dive(pl.x, self.level)
                    if cands:
                        self.sfx.play("dive")
                    self.dive_timer = max(35, 170 - self.level * 12) + random.randint(0, 50)
                    if len(self.enemies) <= 5:
                        self.dive_timer = 40

        if self.phase == "dead":
            self.phase_timer -= 1
            if self.phase_timer <= 0:
                if self.lives > 0:
                    pl.alive = True
                    pl.x = WIDTH / 2
                    pl.invulnerable = 150
                    self.enemy_bullets.clear()
                    self.phase = "play"
                else:
                    self.phase = "gameover"
                    self.sfx.play("gameover")

        if self.phase == "play" and not self.enemies:
            self.next_stage()

    # ---------- Dibujo ----------
    def blit_center(self, surf, y):
        self.screen.blit(surf, (WIDTH // 2 - surf.get_width() // 2, y))

    def draw_volume_row(self, y, selected):
        label_surf = self.font_med.render_styled(
            "VOLUMEN",
            color=UNSELECTED_COLOR,
            gradient=SELECTED_GRADIENT if selected else None,
            outline=(9, 3, 6) if selected else None,
            outline_size=2,
            shadow=UNSELECTED_SHADOW if not selected else None,
            shadow_offset=(2, 2),
        )
        self.screen.blit(label_surf, (112, y + 4))

        bar_left = 285
        seg_w, seg_gap, seg_h = 13, 4, 24
        if selected:
            al = self.font_med.render_styled("<", color=ACCENT_COLOR, outline=BLACK, outline_size=2)
            ar = self.font_med.render_styled(">", color=ACCENT_COLOR, outline=BLACK, outline_size=2)
            self.screen.blit(al, (bar_left - 24, y + 2))
            self.screen.blit(ar, (bar_left + 10 * (seg_w + seg_gap) + 6, y + 2))

        level = self.sfx.level if (self.sfx.ok and self.sfx.enabled) else 0
        for k in range(10):
            rx = bar_left + k * (seg_w + seg_gap)
            ry = y + 2
            if k < level:
                seg_col = (74, 222, 128) if k < 4 else ((250, 204, 21) if k < 8 else (239, 68, 68))
                pygame.draw.rect(self.screen, seg_col if selected else UNSELECTED_COLOR, (rx, ry, seg_w, seg_h))
                pygame.draw.rect(self.screen, (255, 255, 255), (rx, ry, seg_w, 4))
            else:
                pygame.draw.rect(self.screen, UNSELECTED_SHADOW, (rx, ry, seg_w, seg_h), 2)

    def draw_menu(self):
        self.screen.fill(BG_DEEP)
        self.stars.draw(self.screen)

        # Lineas de horizonte arcade en perspectiva en la parte inferior
        horizon_y = 670
        grid_col = (PANEL_BORDER[0] // 4, PANEL_BORDER[1] // 4, PANEL_BORDER[2] // 4)
        for r in range(5):
            yy = int(horizon_y + r * r * 6.5)
            if yy < HEIGHT:
                pygame.draw.line(self.screen, grid_col, (0, yy), (WIDTH, yy), 1)
        for i in range(-5, 6):
            pygame.draw.line(self.screen, grid_col, (WIDTH // 2 + i * 38, horizon_y), (WIDTH // 2 + i * 110, HEIGHT), 1)

        # Cabecera arcade superior
        self.screen.blit(self.font_small.render_styled("1UP", color=ACCENT_COLOR, outline=BLACK, outline_size=2), (26, 16))
        self.screen.blit(self.font_small.render_styled(f"{self.score:06d}", color=WHITE, shadow=(34, 34, 34), shadow_offset=(2, 2)), (26, 36))

        hs_lbl = self.font_small.render_styled("HIGH SCORE", color=VS_COLOR, outline=BLACK, outline_size=2)
        self.blit_center(hs_lbl, 16)
        hs_val = self.font_small.render_styled(f"{self.high_score:06d}", color=WHITE, shadow=(34, 34, 34), shadow_offset=(2, 2))
        self.blit_center(hs_val, 36)

        cr_lbl = self.font_small.render_styled("CREDITO 01", color=ACCENT_COLOR, outline=BLACK, outline_size=2)
        self.screen.blit(cr_lbl, (WIDTH - cr_lbl.get_width() - 24, 16))
        sfx_txt = "SFX ON" if self.sfx.enabled else "SFX OFF"
        sfx_col = (74, 222, 128) if self.sfx.enabled else (248, 113, 113)
        sfx_surf = self.font_small.render_styled(sfx_txt, color=sfx_col)
        self.screen.blit(sfx_surf, (WIDTH - sfx_surf.get_width() - 24, 36))

        # Escuadron animado flotando sobre el titulo
        bob = math.sin(self.tick * 0.045) * 6
        sway = math.sin(self.tick * 0.025) * 14
        draw_enemy_sprite(self.screen, BEE, 78 + sway * 0.5, 118 + bob, math.pi / 2 - 0.18, self.tick)
        draw_enemy_sprite(self.screen, BUTTERFLY, WIDTH - 78 - sway * 0.5, 118 - bob, math.pi / 2 + 0.18, self.tick)
        draw_enemy_sprite(self.screen, BOSS, WIDTH // 2 + sway * 0.8, 92 + bob * 0.6, math.pi / 2, self.tick)

        # Logotipo principal 3D en dos niveles
        nave_surf = self.font_hero_top.render_styled(
            "NAVE",
            gradient=TITLE_GRADIENT,
            outline=TITLE_OUTLINE,
            outline_size=3,
            shadow=TITLE_SHADOW_MID,
            shadow_offset=(4, 4),
            deep_shadow=TITLE_SHADOW_DEEP,
            deep_offset=(8, 8),
            bevel_top=True,
        )
        self.blit_center(nave_surf, 128)

        vs_y = 216
        pygame.draw.rect(self.screen, DIVIDER_COLOR, (108, vs_y + 9, 134, 3))
        pygame.draw.rect(self.screen, DIVIDER_COLOR, (WIDTH - 242, vs_y + 9, 134, 3))
        pygame.draw.rect(self.screen, VS_COLOR, (198, vs_y + 6, 44, 2))
        pygame.draw.rect(self.screen, VS_COLOR, (WIDTH - 242, vs_y + 6, 44, 2))
        vs_surf = self.font_med.render_styled("VS", color=VS_COLOR, outline=TITLE_OUTLINE, outline_size=2, shadow=TITLE_SHADOW_DEEP, shadow_offset=(3, 3))
        self.blit_center(vs_surf, vs_y - 2)

        bichos_surf = self.font_hero_bot.render_styled(
            "BICHOS",
            gradient=TITLE_GRADIENT,
            outline=TITLE_OUTLINE,
            outline_size=3,
            shadow=TITLE_SHADOW_MID,
            shadow_offset=(4, 4),
            deep_shadow=TITLE_SHADOW_DEEP,
            deep_offset=(8, 8),
            bevel_top=True,
        )
        self.blit_center(bichos_surf, 246)

        # Banner de subtitulo con remaches pixelados
        sub_y = 334
        sub_w = self.font_sub.text_width(SUBTITLE_TEXT) + 44
        sub_x = WIDTH // 2 - sub_w // 2
        sub_box = pygame.Surface((sub_w, 26), pygame.SRCALPHA)
        sub_box.fill(PANEL_BG)
        self.screen.blit(sub_box, (sub_x, sub_y - 6))
        pygame.draw.rect(self.screen, PANEL_BORDER, (sub_x, sub_y - 6, sub_w, 26), 2)
        for rx, ry in ((sub_x - 2, sub_y - 8), (sub_x + sub_w - 2, sub_y - 8), (sub_x - 2, sub_y + 18), (sub_x + sub_w - 2, sub_y + 18)):
            pygame.draw.rect(self.screen, ACCENT_COLOR, (rx, ry, 4, 4))
        self.blit_center(self.font_sub.render_styled(SUBTITLE_TEXT, color=SUBTITLE_COLOR, shadow=BLACK, shadow_offset=(2, 2)), sub_y)

        # Panel contenedor del menu
        panel_x, panel_y, panel_w, panel_h = 86, 378, WIDTH - 172, 248
        panel = pygame.Surface((panel_w, panel_h), pygame.SRCALPHA)
        panel.fill(PANEL_BG)
        self.screen.blit(panel, (panel_x, panel_y))
        pygame.draw.rect(self.screen, PANEL_BORDER, (panel_x, panel_y, panel_w, panel_h), 2)

        for i, option in enumerate(self.MENU_OPTIONS):
            y = 398 + i * 58
            selected = (i == self.menu_index)
            if selected:
                band = pygame.Surface((panel_w - 24, 42), pygame.SRCALPHA)
                band.fill(BAND_BG)
                self.screen.blit(band, (panel_x + 12, y - 8))
                pygame.draw.rect(self.screen, BAND_BORDER, (panel_x + 12, y - 8, panel_w - 24, 42), 1)

            if option == "VOLUMEN":
                self.draw_volume_row(y, selected)
            elif selected:
                arrow_shift = int(math.sin(self.tick * 0.2) * 3)
                opt_surf = self.font_large.render_styled(
                    option,
                    gradient=SELECTED_GRADIENT,
                    outline=(9, 3, 6),
                    outline_size=2,
                    shadow=TITLE_SHADOW_DEEP,
                    shadow_offset=(3, 3),
                )
                self.blit_center(opt_surf, y - 2)
                w = self.font_large.text_width(option)
                al = self.font_large.render_styled(">", color=ACCENT_COLOR, outline=BLACK, outline_size=2)
                ar = self.font_large.render_styled("<", color=ACCENT_COLOR, outline=BLACK, outline_size=2)
                self.screen.blit(al, (WIDTH // 2 - w // 2 - 28 + arrow_shift, y - 2))
                self.screen.blit(ar, (WIDTH // 2 + w // 2 + 10 - arrow_shift, y - 2))
            else:
                opt_surf = self.font_large.render_styled(
                    option,
                    color=UNSELECTED_COLOR,
                    shadow=UNSELECTED_SHADOW,
                    shadow_offset=(2, 2),
                )
                self.blit_center(opt_surf, y)

        # Nave lista en plataforma de lanzamiento
        draw_player_ship(self.screen, WIDTH // 2, 672 + math.sin(self.tick * 0.08) * 3, 1.05, flame=True)

        if (self.tick // 24) % 2 == 0 or (self.tick % 48) < 34:
            self.blit_center(self.font_small.render_styled("PRESIONA ENTER PARA JUGAR", color=PROMPT_COLOR, outline=BLACK, outline_size=2), 722)
        self.blit_center(self.font_small.render_styled("W S ELEGIR   A D VOLUMEN   N SONIDO", color=UNSELECTED_COLOR, shadow=BLACK, shadow_offset=(1, 1)), 758)

    def draw_controls(self):
        self.screen.fill(BG_DEEP)
        self.stars.draw(self.screen)
        self.blit_center(
            self.font_title.render_styled("CONTROLES", gradient=TITLE_GRADIENT, outline=TITLE_OUTLINE, outline_size=2, shadow=TITLE_SHADOW_MID, shadow_offset=(3, 3)),
            58,
        )
        lines = [
            ("← → / A D", "MOVER LA NAVE"),
            ("ESPACIO", "DISPARAR (MANTEN PRESIONADO)"),
            ("P", "PAUSA"),
            ("N", "SONIDO ON/OFF"),
            ("ESC", "VOLVER AL MENU"),
        ]
        y = 160
        for key, desc in lines:
            self.screen.blit(self.font_med.render_styled(key, color=ACCENT_COLOR, shadow=BLACK, shadow_offset=(2, 2)), (42, y))
            self.screen.blit(self.font_small.render_styled(desc, color=(230, 230, 235)), (245, y + 4))
            y += 50

        self.blit_center(self.font_small.render_styled("PUNTOS (FORMACION / PICADA)", color=VS_COLOR, outline=BLACK, outline_size=2), 445)
        info = [(BEE, "AVISPA"), (BUTTERFLY, "POLILLA"), (BOSS, "ESCARABAJO (2 GOLPES)")]
        for i, (kind, name) in enumerate(info):
            yy = 505 + i * 54
            draw_enemy_sprite(self.screen, kind, 85, yy + 10, math.pi / 2, self.tick)
            self.screen.blit(self.font_small.render_styled(name, color=(244, 244, 245)), (135, yy))
            pts = POINTS[kind]
            self.screen.blit(self.font_small.render_styled(f"{pts[0]} / {pts[1]}", color=NEON_YELLOW), (435, yy))

        self.blit_center(self.font_small.render_styled("PRESIONA UNA TECLA PARA VOLVER", color=SUBTITLE_COLOR), HEIGHT - 56)

    def draw_hud(self):
        self.screen.blit(self.font_small.render("1UP", True, NEON_RED), (20, 12))
        self.screen.blit(self.font_med.render_styled(f"{self.score:06d}", color=WHITE, shadow=BLACK, shadow_offset=(2, 2)), (20, 34))
        self.blit_center(self.font_small.render("HIGH SCORE", True, NEON_RED), 12)
        self.blit_center(self.font_med.render_styled(f"{self.high_score:06d}", color=WHITE, shadow=BLACK, shadow_offset=(2, 2)), 34)
        s = self.font_small.render_styled(f"STAGE {self.level}", color=ACCENT_COLOR)
        self.screen.blit(s, (WIDTH - s.get_width() - 20, 16))

        for i in range(max(0, self.lives)):
            draw_player_ship(self.screen, 30 + i * 34, HEIGHT - 24, 0.7, flame=False)

    def draw_game(self):
        self.screen.fill(DARK_BLUE)
        self.stars.draw(self.screen)

        for e in self.enemies:
            e.draw(self.screen, self.tick)
        for b in self.bullets:
            b.draw(self.screen)
        for b in self.enemy_bullets:
            b.draw(self.screen)
        for p in self.particles:
            p.draw(self.screen)
        self.player.draw(self.screen)
        self.draw_hud()

        if self.phase == "intro":
            self.blit_center(
                self.font_title.render_styled(f"STAGE {self.level}", gradient=TITLE_GRADIENT, outline=BLACK, outline_size=2),
                HEIGHT // 2 - 44,
            )
            self.blit_center(self.font_small.render("PREPARATE!", True, WHITE), HEIGHT // 2 + 18)

        if self.phase == "gameover":
            overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
            overlay.fill((0, 0, 0, 180))
            self.screen.blit(overlay, (0, 0))
            self.blit_center(
                self.font_title.render_styled("GAME OVER", gradient=TITLE_GRADIENT, outline=BLACK, outline_size=2, shadow=TITLE_SHADOW_DEEP, shadow_offset=(4, 4)),
                HEIGHT // 2 - 92,
            )
            self.blit_center(self.font_med.render(f"PUNTAJE FINAL: {self.score}", True, WHITE), HEIGHT // 2 - 20)
            if self.score >= self.high_score and self.score > 0:
                self.blit_center(self.font_small.render("NUEVO RECORD!", True, NEON_YELLOW), HEIGHT // 2 + 26)
            self.blit_center(self.font_small.render("R: REINICIAR   |   M: MENU", True, ACCENT_COLOR), HEIGHT // 2 + 72)

        if self.paused and self.phase != "gameover":
            self.blit_center(
                self.font_title.render_styled("PAUSA", gradient=SELECTED_GRADIENT, outline=BLACK, outline_size=2),
                HEIGHT // 2 - 30,
            )

    def draw(self):
        if self.state == "menu":
            self.draw_menu()
        elif self.state == "controls":
            self.draw_controls()
        else:
            self.draw_game()
        pygame.display.flip()

    # ---------- Input ----------
    def handle_menu_key(self, key) -> bool:
        option = self.MENU_OPTIONS[self.menu_index]
        if key in (pygame.K_UP, pygame.K_w):
            self.menu_index = (self.menu_index - 1) % len(self.MENU_OPTIONS)
            self.sfx.play("move")
        elif key in (pygame.K_DOWN, pygame.K_s):
            self.menu_index = (self.menu_index + 1) % len(self.MENU_OPTIONS)
            self.sfx.play("move")
        elif key in (pygame.K_LEFT, pygame.K_a) and option == "VOLUMEN":
            self.sfx.change_volume(-1)
            self.sfx.play("move")
        elif key in (pygame.K_RIGHT, pygame.K_d) and option == "VOLUMEN":
            self.sfx.change_volume(+1)
            self.sfx.play("move")
        elif key in (pygame.K_RETURN, pygame.K_KP_ENTER, pygame.K_SPACE):
            self.sfx.play("select")
            if option == "JUGAR":
                self.start_game()
            elif option == "CONTROLES":
                self.state = "controls"
            elif option == "SALIR":
                return False
        elif key == pygame.K_ESCAPE:
            return False
        return True

    def handle_game_key(self, key):
        if key == pygame.K_ESCAPE:
            self.go_to_menu()
        elif key == pygame.K_p and self.phase != "gameover":
            self.paused = not self.paused
        elif key == pygame.K_r and self.phase == "gameover":
            self.start_game()
        elif key == pygame.K_m and self.phase == "gameover":
            self.go_to_menu()

    def run(self):
        running = True
        while running:
            self.clock.tick(FPS)
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    running = False
                elif event.type == pygame.KEYDOWN:
                    if event.key == pygame.K_n:
                        self.sfx.toggle()
                    if self.state == "menu":
                        running = self.handle_menu_key(event.key)
                    elif self.state == "controls":
                        self.state = "menu"
                    else:
                        self.handle_game_key(event.key)

            if self.state == "playing":
                self.update_game()
            else:
                self.update_menu()
            self.draw()

        pygame.quit()
        sys.exit()


if __name__ == "__main__":
    Game().run()

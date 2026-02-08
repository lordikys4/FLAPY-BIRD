import pygame
import sounddevice as sd
import numpy as np
from random import randint

# ========= AUDIO =========
sr = 16000
block = 256
mic_level = 0.0
last_mic = 0.0
audio_ok = True


def audio_cb(indata, frames, time, status):
    global mic_level
    if status:
        return
    rms = float(np.sqrt(np.mean(indata ** 2)))
    mic_level = 0.85 * mic_level + 0.15 * rms


try:
    sd.default.samplerate = sr
    sd.default.channels = 1
    stream = sd.InputStream(blocksize=block, callback=audio_cb)
    stream.start()
except Exception as e:
    print("Audio disabled:", e)
    audio_ok = False


# ========= PYGAME =========
pygame.init()
W, H = 1200, 800
win = pygame.display.set_mode((W, H))
pygame.display.set_caption("Voice Flappy")
clock = pygame.time.Clock()

# ========= IMAGES =========
bg_img = pygame.image.load("background.png").convert()
bg_img = pygame.transform.scale(bg_img, (W, H))

player_img = pygame.image.load("icon.png").convert_alpha()
player_img = pygame.transform.scale(player_img, (80, 80))

pipe_top_img = pygame.image.load("pipe_top.png").convert_alpha()
pipe_bottom_img = pygame.image.load("pipe_bottom.png").convert_alpha()

# ========= PLAYER =========
player = pygame.Rect(150, H // 2 - 40, 80, 80)

# ========= PIPES =========
PIPE_W = 120
GAP = 260


def gen_pipes(n):
    pipes = []
    x = W
    for _ in range(n):
        h = randint(100, 420)
        pipes.append(pygame.Rect(x, 0, PIPE_W, h))
        pipes.append(pygame.Rect(x, h + GAP, PIPE_W, H - h - GAP))
        x += 500
    return pipes


pipes = gen_pipes(6)

# ========= GAME =========
font = pygame.font.Font(None, 80)
score = 0
lose = False

y_vel = 0
gravity = 0.6
IMPULSE = -9
THRESH = 0.001
MAX_FALL = 12


def reset():
    global pipes, score, lose, y_vel
    pipes = gen_pipes(6)
    score = 0
    lose = False
    y_vel = 0
    player.y = H // 2 - 40


# ========= LOOP =========
run = True
while run:
    for e in pygame.event.get():
        if e.type == pygame.QUIT:
            run = False

    # === INPUT ===
    if audio_ok:
        if mic_level > THRESH and last_mic <= THRESH and not lose:
            y_vel = IMPULSE
        last_mic = mic_level
    else:
        if pygame.key.get_pressed()[pygame.K_SPACE] and not lose:
            y_vel = IMPULSE

    # === PHYSICS ===
    if not lose:
        y_vel += gravity
        y_vel = min(y_vel, MAX_FALL)
        player.y += int(y_vel)

    if player.top < 0:
        player.top = 0
        y_vel = 0

    if player.bottom > H:
        player.bottom = H
        lose = True

    # ========= DRAW =========
    win.blit(bg_img, (0, 0))
    win.blit(player_img, player)

    for p in pipes[:]:
        if not lose:
            p.x -= 8

        if p.top == 0:
            img = pygame.transform.scale(pipe_top_img, (p.width, p.height))
        else:
            img = pygame.transform.scale(pipe_bottom_img, (p.width, p.height))

        win.blit(img, (p.x, p.y))

        if p.right < 0:
            pipes.remove(p)
            if p.top == 0:
                score += 1

        if player.colliderect(p):
            lose = True

    if len(pipes) < 6:
        pipes += gen_pipes(2)

    txt = font.render(str(score), True, (0, 0, 0))
    win.blit(txt, (W // 2 - txt.get_width() // 2, 30))

    if lose:
        t = font.render("PRESS R", True, (0, 0, 0))
        win.blit(t, (W // 2 - t.get_width() // 2, H // 2))

    pygame.display.update()
    clock.tick(60)

    if pygame.key.get_pressed()[pygame.K_r] and lose:
        reset()

pygame.quit()

if audio_ok:
    stream.stop()
    stream.close()

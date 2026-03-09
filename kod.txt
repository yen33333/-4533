import sys
import os
import json
import pygame


# tile settings and virtual resolution
TILE = 16
# grid in tiles (16:9) - reduced by 2x
GRID_COLS, GRID_ROWS = 80, 45
# world size in pixels (multiple of TILE)
BASE_WIDTH, BASE_HEIGHT = GRID_COLS * TILE, GRID_ROWS * TILE  # 80 x 45 tiles
FPS = 60
# toggle on-screen debug info (skin/platform texture debug)
SHOW_DEBUG = False


class Player:
    def __init__(self, x, y, w=TILE, h=2 * TILE):
        # w/h are in pixels (use TILE for 16x16)
        # use float positions for smooth movement to avoid stutter
        self.pos_x = float(x)
        self.pos_y = float(y)
        self.rect = pygame.Rect(int(self.pos_x), int(self.pos_y), w, h)
        self.vel_x = 0.0
        self.vel_y = 0.0
        # movement tuned for pixel/tile world
        self.speed = 200.0
        # jump / physics
        self.jump_strength = 480.0
        self.gravity = 2000.0
        self.max_fall = 1000.0
        self.on_ground = False
        # double jump disabled; only wall-jump allowed
        self.can_double_jump = False
        # jump rate limiting (ms)
        self.last_jump_time = 0
        self.jump_cooldown = 120
        self.wall_dir = 0  # -1 left, 1 right, 0 none
        # facing: 1 = right, -1 = left
        self.facing = 1

    def update(self, dt, platforms):
        keys = pygame.key.get_pressed()

        target_vx = 0.0
        if keys[pygame.K_a] or keys[pygame.K_LEFT]:
            target_vx = -self.speed
        if keys[pygame.K_d] or keys[pygame.K_RIGHT]:
            target_vx = self.speed

        # simple accel
        self.vel_x = target_vx

        # horizontal move using float positions for smoothness
        self.pos_x += self.vel_x * dt
        self.rect.x = int(self.pos_x)
        touching_left = False
        touching_right = False
        for p in platforms:
            if self.rect.colliderect(p):
                if self.vel_x > 0:
                    self.rect.right = p.left
                    touching_right = True
                elif self.vel_x < 0:
                    self.rect.left = p.right
                    touching_left = True
        # sync float pos with corrected rect
        self.pos_x = float(self.rect.x)
        # update facing based on horizontal velocity
        if self.vel_x > 1:
            self.facing = 1
        elif self.vel_x < -1:
            self.facing = -1

        # variable gravity: when holding jump while rising, reduce gravity for higher jump
        holding_jump = keys[pygame.K_w] or keys[pygame.K_UP] or keys[pygame.K_SPACE]
        if self.vel_y < 0 and holding_jump:
            gravity_effect = self.gravity * 0.55
        else:
            gravity_effect = self.gravity

        # wall slide effect: slow down fall a bit when touching wall and falling
        if not self.on_ground and (touching_left or touching_right) and self.vel_y > 0:
            gravity_effect *= 0.6

        self.vel_y += gravity_effect * dt
        if self.vel_y > self.max_fall:
            self.vel_y = self.max_fall

        # vertical move using float pos
        self.pos_y += self.vel_y * dt
        self.rect.y = int(self.pos_y)
        self.on_ground = False
        for p in platforms:
            if self.rect.colliderect(p):
                if self.vel_y > 0:
                    self.rect.bottom = p.top
                    self.vel_y = 0
                    self.on_ground = True
                    self.can_double_jump = True
                elif self.vel_y < 0:
                    self.rect.top = p.bottom
                    self.vel_y = 0
        # sync float pos with corrected rect
        self.pos_y = float(self.rect.y)

        # wall detection when in air
        if not self.on_ground:
            if touching_left:
                self.wall_dir = -1
            elif touching_right:
                self.wall_dir = 1
            else:
                self.wall_dir = 0
        else:
            self.wall_dir = 0


def main():
    # initialize mixer with good defaults for music/sfx, then init pygame
    pygame.mixer.pre_init(44100, -16, 2, 512)
    pygame.init()
    # start fullscreen
    screen = pygame.display.set_mode((0, 0), pygame.FULLSCREEN)
    clock = pygame.time.Clock()
    font = pygame.font.SysFont(None, 24)
    WIDTH, HEIGHT = screen.get_size()
    # music toggle button rect (screen coords)
    music_button_rect = pygame.Rect(WIDTH - 110, 10, 100, 28)
    # virtual surface for pixel/tile rendering
    vs = pygame.Surface((BASE_WIDTH, BASE_HEIGHT))
    # helper: map screen coordinates to virtual surface coordinates
    def screen_to_virtual(pos):
        sx_map = BASE_WIDTH / WIDTH
        sy_map = BASE_HEIGHT / HEIGHT
        return (int(pos[0] * sx_map), int(pos[1] * sy_map))

    # audio assets (place your files in assets/audio/)
    assets_audio = os.path.join(os.path.dirname(__file__), 'assets', 'audio')
    os.makedirs(assets_audio, exist_ok=True)
    bgm_path = os.path.join(assets_audio, 'bgm.ogg')
    jump_path = os.path.join(assets_audio, 'jump.wav')
    die_path = os.path.join(assets_audio, 'die.wav')

    # graphics assets (player skin)
    assets_graphics = os.path.join(os.path.dirname(__file__), 'assets', 'graphics')
    os.makedirs(assets_graphics, exist_ok=True)
    player_image = None
    player_img_file = None
    # single-file skin fallback
    for cand in ('player.png', 'player.bmp', 'player.gif'):
        p = os.path.join(assets_graphics, cand)
        if os.path.exists(p):
            try:
                player_image = pygame.image.load(p).convert_alpha()
                player_img_file = p
                break
            except Exception:
                player_image = None
                player_img_file = None
    # animation folder support: assets/graphics/player/ with optional subfolders
    player_anims = {'idle': [], 'walk': [], 'jump': []}
    player_anim_dir = os.path.join(assets_graphics, 'player')
    if os.path.isdir(player_anim_dir):
        # first, look for explicit subfolders: idle, walk, jump
        for sub in ('idle', 'walk', 'jump'):
            subdir = os.path.join(player_anim_dir, sub)
            if os.path.isdir(subdir):
                files = sorted(os.listdir(subdir))
                for fn in files:
                    if not fn.lower().endswith(('.png', '.bmp', '.gif')):
                        continue
                    path = os.path.join(subdir, fn)
                    try:
                        surf = pygame.image.load(path).convert_alpha()
                        player_anims[sub].append(surf)
                    except Exception:
                        continue
        # then, load any loose files in the player folder and classify by name
        files = sorted(os.listdir(player_anim_dir))
        for fn in files:
            if not fn.lower().endswith(('.png', '.bmp', '.gif')):
                continue
            path = os.path.join(player_anim_dir, fn)
            if os.path.isdir(path):
                continue
            try:
                surf = pygame.image.load(path).convert_alpha()
            except Exception:
                continue
            low = fn.lower()
            # prefix or keyword classification
            if low.startswith('walk') or 'walk' in low or 'run' in low:
                player_anims['walk'].append(surf)
            elif low.startswith('jump') or 'jump' in low:
                player_anims['jump'].append(surf)
            elif low.startswith('idle') or low.startswith('stand') or 'idle' in low or 'stand' in low:
                player_anims['idle'].append(surf)
            else:
                # fallback to idle
                player_anims['idle'].append(surf)
    # if no anim folder frames but single image exists, use it as idle
    if not any(player_anims.values()) and player_image:
        player_anims['idle'] = [player_image]
    # debug prints
    if any(player_anims.values()):
        print('Loaded player animations:')
        for k, v in player_anims.items():
            if v:
                print(f'  {k}: {len(v)} frames')
    else:
        print(f'No player skin found in {assets_graphics} (expected player.png or assets/graphics/player/*.png)')

    # load platform textures from assets/graphics/platforms/
    platform_textures = {}
    plat_dir = os.path.join(assets_graphics, 'platforms')
    if os.path.isdir(plat_dir):
        for fn in sorted(os.listdir(plat_dir)):
            if not fn.lower().endswith(('.png', '.bmp', '.gif')):
                continue
            name = os.path.splitext(fn)[0]
            # support naming like 'stone@2' or 'stone@2x2' where suffix sets tiles-per-texture
            parts = name.split('@')
            key = parts[0].lower()
            tile_factor = 1
            if len(parts) > 1:
                suf = parts[1]
                try:
                    if 'x' in suf:
                        tile_factor = int(suf.split('x')[0])
                    else:
                        tile_factor = int(suf)
                except Exception:
                    tile_factor = 1
            path = os.path.join(plat_dir, fn)
            try:
                surf = pygame.image.load(path).convert_alpha()
                platform_textures[key] = {'surf': surf, 'tiles': tile_factor}
            except Exception:
                continue
    if platform_textures:
        platform_textures_keys = list(platform_textures.keys())
        print(f'Loaded platform textures: {platform_textures_keys}')
    else:
        platform_textures_keys = []
    # load spike textures from assets/graphics/spikes/
    spike_textures = {}
    spike_dir = os.path.join(assets_graphics, 'spikes')
    if os.path.isdir(spike_dir):
        for fn in sorted(os.listdir(spike_dir)):
            if not fn.lower().endswith(('.png', '.bmp', '.gif')):
                continue
            name = os.path.splitext(fn)[0]
            parts = name.split('@')
            base = parts[0].lower()
            tile_factor = 1
            suf = None
            if len(parts) > 1:
                suf = parts[1]
                try:
                    if 'x' in suf:
                        tile_factor = int(suf.split('x')[0])
                    else:
                        tile_factor = int(suf)
                except Exception:
                    tile_factor = 1
            # create a full key that includes the suffix when present (e.g., 'lava@2')
            full_key = base if suf is None else f"{base}@{suf.lower()}"
            path = os.path.join(spike_dir, fn)
            try:
                surf = pygame.image.load(path).convert_alpha()
                spike_textures[full_key] = {'surf': surf, 'tiles': tile_factor}
                # also register base key to the first encountered variant if not present,
                # so legacy references like 'lava' still work
                if base not in spike_textures:
                    spike_textures[base] = spike_textures[full_key]
            except Exception:
                continue
    spike_textures_keys = list(spike_textures.keys()) if spike_textures else []
    # cache for pre-tiled platform surfaces to reduce per-frame blits
    platform_cache = {}
    # cache for pre-tiled spike surfaces
    spike_cache = {}
    # load goal texture (single image goal.png or first image in assets/graphics/goals/)
    goal_texture = None
    # try single file first
    for cand in ('goal.png', 'goal.bmp', 'goal.gif'):
        p = os.path.join(assets_graphics, cand)
        if os.path.exists(p):
            try:
                goal_texture = pygame.image.load(p).convert_alpha()
            except Exception:
                goal_texture = None
            break
    # try goals/ folder next
    if goal_texture is None:
        goals_dir = os.path.join(assets_graphics, 'goals')
        if os.path.isdir(goals_dir):
            for fn in sorted(os.listdir(goals_dir)):
                if not fn.lower().endswith(('.png', '.bmp', '.gif')):
                    continue
                path = os.path.join(goals_dir, fn)
                try:
                    goal_texture = pygame.image.load(path).convert_alpha()
                    break
                except Exception:
                    continue

    # load background music (optional)
    # try several common filenames for bgm
    music_loaded = False
    bgm_candidates = ['bgm.ogg', 'bgm.mp3', 'bgm.wav']
    bgm_file = None
    # bgm sound/channel objects for fallback playback
    bgm_snd = None
    bgm_channel = None
    # method: 'music' when using pygame.mixer.music, 'channel' when using a Channel
    music_play_method = None
    for cand in bgm_candidates:
        p = os.path.join(assets_audio, cand)
        if os.path.exists(p):
            bgm_file = p
            break
    if bgm_file:
        try:
            pygame.mixer.music.load(bgm_file)
            pygame.mixer.music.set_volume(0.6)
            pygame.mixer.music.play(-1)
            music_loaded = True
            music_play_method = 'music'
        except Exception as e:
            print(f'Error loading bgm: {e}')
            music_loaded = False
    else:
        print(f'No bgm found in {assets_audio} (tried: {bgm_candidates})')
    # fallback: if music not loaded via mixer.music try playing as Sound on a channel
    if not music_loaded and bgm_file:
        try:
            bgm_snd = pygame.mixer.Sound(bgm_file)
            bgm_snd.set_volume(0.6)
            # try to play on channel 0 looped
            try:
                ch = pygame.mixer.Channel(0)
                ch.play(bgm_snd, loops=-1)
                bgm_channel = ch
                music_loaded = True
                music_play_method = 'channel'
                print(f'Playing bgm via Sound on channel: {bgm_file}')
            except Exception:
                # last resort: play Sound and capture returned Channel
                ch = bgm_snd.play(loops=-1)
                if ch is not None:
                    bgm_channel = ch
                    music_loaded = True
                    music_play_method = 'channel'
                    print(f'Playing bgm via Sound fallback: {bgm_file}')
                else:
                    # unable to play
                    print(f'Fallback bgm failed to play: {bgm_file}')
        except Exception as e:
            print(f'Fallback bgm failed: {e}')

    # load sfx
    jump_sfx = None
    die_sfx = None
    try:
        if os.path.exists(jump_path):
            jump_sfx = pygame.mixer.Sound(jump_path)
            jump_sfx.set_volume(0.8)
    except Exception:
        jump_sfx = None
    try:
        if os.path.exists(die_path):
            die_sfx = pygame.mixer.Sound(die_path)
            die_sfx.set_volume(0.9)
    except Exception:
        die_sfx = None

    music_paused = False

    # define levels in tile units (x,y,w,h) where each unit = TILE (16 px)
    levels = []
    # helper to create a level quickly (spikes optional). All coords in tiles
    def make_level(start_tile, plats_tiles, goal_tile, spikes_tiles=None):
        return {'start': start_tile, 'platforms': plats_tiles, 'goal': goal_tile, 'spikes': spikes_tiles or []}

    # create 10 easier levels with small steps and many platforms
    # convert old-style to tile-based simple levels
    # ground is 2 tiles high at tile y = BASE_HEIGHT/TILE - 2
    GROUND_Y = (BASE_HEIGHT // TILE) - 2

    levels.append(make_level((4, GROUND_Y - 2), [
        (0, GROUND_Y, GRID_COLS, 2),
        (4, GROUND_Y - 6, 9, 1),
        (15, GROUND_Y - 9, 9, 1),
        (26, GROUND_Y - 7, 7, 1),
        (36, GROUND_Y - 6, 10, 1),
    ], (44, GROUND_Y - 5, 4, 4), spikes_tiles=[(12, GROUND_Y - 1, 5, 1)]))

    levels.append(make_level((3, GROUND_Y - 2), [
        (0, GROUND_Y, GRID_COLS, 2),
        (8, GROUND_Y - 4, 7, 1),
        (18, GROUND_Y - 6, 7, 1),
        (28, GROUND_Y - 4, 7, 1),
        (38, GROUND_Y - 6, 7, 1),
    ], (44, GROUND_Y - 5, 4, 4), spikes_tiles=[(22, GROUND_Y - 1, 4, 1)]))

    levels.append(make_level((2, GROUND_Y - 2), [
        (0, GROUND_Y, GRID_COLS, 2),
        (5, GROUND_Y - 7, 6, 1),
        (14, GROUND_Y - 10, 6, 1),
        (23, GROUND_Y - 12, 6, 1),
        (31, GROUND_Y - 8, 8, 1),
    ], (42, GROUND_Y - 6, 5, 5), spikes_tiles=[(30, GROUND_Y - 1, 3, 1)]))

    levels.append(make_level((6, GROUND_Y - 2), [
        (0, GROUND_Y, GRID_COLS, 2),
        (10, GROUND_Y - 5, 9, 1),
        (26, GROUND_Y - 7, 9, 1),
        (40, GROUND_Y - 5, 8, 1),
    ], (46, GROUND_Y - 8, 4, 4), spikes_tiles=[]))

    levels.append(make_level((5, GROUND_Y - 2), [
        (0, GROUND_Y, GRID_COLS, 2),
        (7, GROUND_Y - 6, 7, 1),
        (18, GROUND_Y - 8, 7, 1),
        (29, GROUND_Y - 6, 7, 1),
        (40, GROUND_Y - 8, 9, 1),
    ], (46, GROUND_Y - 6, 4, 4), spikes_tiles=[(42, GROUND_Y - 1, 6, 1)]))

    # next 5 with slightly different layout but still easy
    levels.append(make_level((4, GROUND_Y - 2), [
        (0, GROUND_Y, GRID_COLS, 2),
        (9, GROUND_Y - 4, 8, 1),
        (22, GROUND_Y - 6, 10, 1),
        (36, GROUND_Y - 8, 10, 1),
    ], (46, GROUND_Y - 6, 4, 4), spikes_tiles=[]))

    levels.append(make_level((3, GROUND_Y - 2), [
        (0, GROUND_Y, GRID_COLS, 2),
        (5, GROUND_Y - 7, 10, 1),
        (18, GROUND_Y - 11, 10, 1),
        (36, GROUND_Y - 6, 8, 1),
    ], (46, GROUND_Y - 6, 5, 5), spikes_tiles=[(12, GROUND_Y - 1, 5, 1), (30, GROUND_Y - 1, 5, 1)]))

    levels.append(make_level((6, GROUND_Y - 2), [
        (0, GROUND_Y, GRID_COLS, 2),
        (12, GROUND_Y - 6, 8, 1),
        (24, GROUND_Y - 7, 8, 1),
        (36, GROUND_Y - 5, 8, 1),
        (48, GROUND_Y - 7, 6, 1),
    ], (56, GROUND_Y - 6, 4, 4), spikes_tiles=[]))

    levels.append(make_level((4, GROUND_Y - 2), [
        (0, GROUND_Y, GRID_COLS, 2),
        (14, GROUND_Y - 5, 7, 1),
        (22, GROUND_Y - 8, 8, 1),
        (32, GROUND_Y - 5, 8, 1),
    ], (44, GROUND_Y - 6, 4, 4), spikes_tiles=[(18, GROUND_Y - 1, 4, 1)]))

    levels.append(make_level((5, GROUND_Y - 2), [
        (0, GROUND_Y, GRID_COLS, 2),
        (8, GROUND_Y - 6, 7, 1),
        (20, GROUND_Y - 6, 7, 1),
        (32, GROUND_Y - 6, 7, 1),
    ], (44, GROUND_Y - 8, 4, 4), spikes_tiles=[(35, GROUND_Y - 1, 5, 1)]))

    level_idx = 0

    def assign_default_textures(lvls):
        # assign default texture 'stone' to platforms for levels 1..5 (indices 0..4)
        for li in range(min(5, len(lvls))):
            plats = lvls[li].get('platforms', [])
            for pi, p in enumerate(plats):
                try:
                    # keep existing texture if present
                    if isinstance(p, (list, tuple)) and len(p) >= 5 and p[4]:
                        continue
                    # replace tuple with list and append texture key
                    base = list(p[:4])
                    base.append('stone')
                    lvls[li]['platforms'][pi] = base
                except Exception:
                    continue

        # assign default texture 'bricks' to platforms for levels 6..10 (indices 5..9)
        for li in range(5, min(10, len(lvls))):
            plats = lvls[li].get('platforms', [])
            for pi, p in enumerate(plats):
                try:
                    # keep existing texture if present
                    if isinstance(p, (list, tuple)) and len(p) >= 5 and p[4]:
                        continue
                    base = list(p[:4])
                    base.append('bricks')
                    lvls[li]['platforms'][pi] = base
                except Exception:
                    continue
        # assign default spike texture 'lava@1' to spikes for levels 1..5 and 'lava@2' for levels 6..10
        for li in range(min(5, len(lvls))):
            sps = lvls[li].get('spikes', [])
            for si, s in enumerate(sps):
                try:
                    if isinstance(s, (list, tuple)) and len(s) >= 5 and s[4]:
                        continue
                    base = list(s[:4])
                    base.append('lava@1')
                    lvls[li]['spikes'][si] = base
                except Exception:
                    continue
        for li in range(5, min(10, len(lvls))):
            sps = lvls[li].get('spikes', [])
            for si, s in enumerate(sps):
                try:
                    if isinstance(s, (list, tuple)) and len(s) >= 5 and s[4]:
                        continue
                    base = list(s[:4])
                    base.append('lava@2')
                    lvls[li]['spikes'][si] = base
                except Exception:
                    continue

    # apply defaults to initial levels
    assign_default_textures(levels)

    # color palettes: change look after each 5 levels
    palettes = [
        # (bg, platform, goal, player)
        ((30, 30, 40), (100, 180, 100), (80, 160, 220), (200, 60, 60)),
        ((20, 35, 55), (200, 170, 120), (255, 200, 80), (180, 120, 200)),
    ]

    def build_level(i):
        lvl = levels[i]
        plats = []
        plat_tex_keys = []
        for p in lvl['platforms']:
            # p may be [x,y,w,h] or [x,y,w,h,tex]
            try:
                tx, ty, tw, th = p[0], p[1], p[2], p[3]
            except Exception:
                continue
            plats.append(pygame.Rect(int(tx * TILE), int(ty * TILE), int(tw * TILE), int(th * TILE)))
            tex = None
            if isinstance(p, (list, tuple)) and len(p) >= 5:
                try:
                    tex = str(p[4]).lower()
                except Exception:
                    tex = str(p[4])
            # default texture for early levels (1..5 -> indices 0..4)
            if tex is None and i < 5:
                tex = 'stone'
            plat_tex_keys.append(tex)
        spikes_rects = []
        spike_tex_keys = []
        for s in lvl.get('spikes', []):
            try:
                tx, ty, tw, th = s[0], s[1], s[2], s[3]
            except Exception:
                continue
            spikes_rects.append(pygame.Rect(int(tx * TILE), int(ty * TILE), int(tw * TILE), int(th * TILE)))
            # optional texture key for this spike (e.g., 'lava')
            tex = None
            if isinstance(s, (list, tuple)) and len(s) >= 5:
                try:
                    tex = str(s[4]).lower()
                except Exception:
                    tex = None
            spike_tex_keys.append(tex)
        gx, gy, gw, gh = lvl['goal']
        goal = pygame.Rect(int(gx * TILE), int(gy * TILE), int(gw * TILE), int(gh * TILE))
        sx_start, sy_start = lvl['start']
        start = (int(sx_start * TILE), int(sy_start * TILE))
        return plats, plat_tex_keys, goal, start, spikes_rects, spike_tex_keys

    def place_player_safe(player_obj, start_pos, plats):
        # place player at start_pos (virtual coords) but move up while intersecting platforms
        player_obj.pos_x = float(start_pos[0])
        player_obj.pos_y = float(start_pos[1])
        player_obj.rect.topleft = (int(player_obj.pos_x), int(player_obj.pos_y))
        attempts = 0
        # if inside a platform, move up until free or reach top
        while any(player_obj.rect.colliderect(p) for p in plats) and attempts < 600:
            player_obj.pos_y -= 1.0
            player_obj.rect.y = int(player_obj.pos_y)
            attempts += 1
            if player_obj.rect.top < 0:
                break
        return

    # load external levels if present (portable JSON)
    levels_file = 'levels.json'
    levels_path = os.path.join(os.path.dirname(__file__), levels_file)
    def load_levels_file(path):
        try:
            with open(path, 'r', encoding='utf-8') as f:
                data = json.load(f)
            # basic validation
            if isinstance(data, list) and data:
                return data
        except Exception:
            pass
        return None

    def normalize_levels_for_save(levels_data):
        # ensure all tuples are lists so JSON is consistent
        out = []
        for lvl in levels_data:
            nl = {}
            # start
            st = lvl.get('start')
            if isinstance(st, (list, tuple)) and len(st) >= 2:
                nl['start'] = [int(st[0]), int(st[1])]
            elif isinstance(st, pygame.Rect):
                nl['start'] = [int(st.x // TILE), int(st.y // TILE)]
            else:
                nl['start'] = [0, 0]
            # platforms
            plats = []
            for p in lvl.get('platforms', []):
                if isinstance(p, (list, tuple)) and len(p) >= 4:
                    plats.append([int(p[0]), int(p[1]), int(p[2]), int(p[3])])
                elif isinstance(p, pygame.Rect):
                    plats.append([int(p.x // TILE), int(p.y // TILE), int(p.width // TILE), int(p.height // TILE)])
                else:
                    # ignore unknown entry
                    continue
            nl['platforms'] = plats
            # goal
            g = lvl.get('goal', [0, 0, 1, 1])
            if isinstance(g, (list, tuple)) and len(g) >= 4:
                nl['goal'] = [int(g[0]), int(g[1]), int(g[2]), int(g[3])]
            elif isinstance(g, pygame.Rect):
                nl['goal'] = [int(g.x // TILE), int(g.y // TILE), int(g.width // TILE), int(g.height // TILE)]
            else:
                nl['goal'] = [0, 0, 1, 1]
            # spikes (support optional texture key as 5th element)
            spikes = []
            for s in lvl.get('spikes', []):
                if isinstance(s, (list, tuple)) and len(s) >= 4:
                    entry = [int(s[0]), int(s[1]), int(s[2]), int(s[3])]
                    if len(s) >= 5:
                        try:
                            entry.append(str(s[4]))
                        except Exception:
                            pass
                    spikes.append(entry)
                elif isinstance(s, pygame.Rect):
                    spikes.append([int(s.x // TILE), int(s.y // TILE), int(s.width // TILE), int(s.height // TILE)])
                else:
                    continue
            nl['spikes'] = spikes
            out.append(nl)
        return out

    def save_levels_file(path, levels_data):
        try:
            data = normalize_levels_for_save(levels_data)
            with open(path, 'w', encoding='utf-8') as f:
                json.dump(data, f, ensure_ascii=False, indent=2)
            return True, None
        except Exception as ex:
            return False, str(ex)

    loaded = load_levels_file(levels_path)
    if loaded:
        levels = loaded
        # ensure each loaded level has a ground platform
        def ensure_levels_have_ground(lvls):
            for lvl in lvls:
                plats = lvl.get('platforms', [])
                has_ground = False
                for p in plats:
                    try:
                        if int(p[2]) >= GRID_COLS:
                            has_ground = True
                            break
                    except Exception:
                        pass
                if not has_ground:
                    # insert a ground platform at GROUND_Y
                    lvl.setdefault('platforms', []).insert(0, [0, GROUND_Y, GRID_COLS, 2])
        try:
            ensure_levels_have_ground(levels)
        except Exception:
            pass
        try:
            assign_default_textures(levels)
        except Exception:
            pass

    platforms, plat_tex_keys, goal, start, spikes, spike_tex_keys = build_level(level_idx)
    # use fixed physics size (1x2 tiles). Visuals will be scaled to this size.
    pw = TILE
    ph = 2 * TILE
    # create player
    player = Player(start[0], start[1], pw, ph)
    # attach animation state to player
    player.anim_frames = player_anims
    player.anim_index = 0
    player.anim_timer = 0.0
    player.anim_fps = 8.0
    # skin status for on-screen debugging (counts of frames)
    skin_idle = len(player_anims.get('idle', []))
    skin_walk = len(player_anims.get('walk', []))
    skin_jump = len(player_anims.get('jump', []))
    if skin_idle + skin_walk + skin_jump > 0:
        skin_msg = f"Skin frames - idle:{skin_idle} walk:{skin_walk} jump:{skin_jump}"
    else:
        skin_msg = 'No skin frames loaded'
    # ensure spawn is not inside blocks
    place_player_safe(player, start, platforms)

    jump_was_pressed = False
    # level-skip state: when requesting skip, wait this many ms before changing level
    skip_request_time = 0
    skip_delay_ms = 1000
    skip_was_pressed = False

    running = True
    # editor state
    editor = False
    edit_tool = 'platform'  # 'platform', 'start', 'goal', 'erase'
    drag_start = None
    temp_rect = None
    status_msg = ''
    while running:
        dt = clock.tick(FPS) / 1000.0

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            # handle music button click (global, not editor-only)
            if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                try:
                    mx, my = event.pos
                    # click is in screen coords
                    if music_button_rect.collidepoint((mx, my)):
                        # toggle same as pressing 'm'
                        try:
                            if music_play_method == 'music':
                                if music_paused:
                                    pygame.mixer.music.unpause()
                                    music_paused = False
                                else:
                                    pygame.mixer.music.pause()
                                    music_paused = True
                            elif music_play_method == 'channel' and bgm_channel is not None:
                                if music_paused:
                                    try:
                                        bgm_channel.unpause()
                                    except Exception:
                                        pass
                                    music_paused = False
                                else:
                                    try:
                                        bgm_channel.pause()
                                    except Exception:
                                        pass
                                    music_paused = True
                        except Exception:
                            pass
                except Exception:
                    pass
            if event.type == pygame.KEYDOWN:
                # editor toggle
                if event.key == pygame.K_e:
                    editor = not editor
                    status_msg = 'EDYTOR: ON' if editor else ''
                # toggle music pause
                if event.key == pygame.K_m:
                    # toggle music pause/play for either mixer.music or fallback channel
                    try:
                        if music_play_method == 'music':
                            if music_paused:
                                pygame.mixer.music.unpause()
                                music_paused = False
                            else:
                                pygame.mixer.music.pause()
                                music_paused = True
                        elif music_play_method == 'channel' and bgm_channel is not None:
                            if music_paused:
                                try:
                                    bgm_channel.unpause()
                                except Exception:
                                    # some channel implementations may not support unpause
                                    pass
                                music_paused = False
                            else:
                                try:
                                    bgm_channel.pause()
                                except Exception:
                                    pass
                                music_paused = True
                        else:
                            # no music loaded
                            pass
                    except Exception:
                        pass
                # tool keys when in editor
                if editor and event.key == pygame.K_1:
                    edit_tool = 'platform'
                if editor and event.key == pygame.K_5:
                    edit_tool = 'spike'
                if editor and event.key == pygame.K_2:
                    edit_tool = 'start'
                if editor and event.key == pygame.K_3:
                    edit_tool = 'goal'
                if editor and event.key == pygame.K_4:
                    edit_tool = 'erase'
                # save/load when in editor
                if editor and event.key == pygame.K_s:
                    # save current levels to file in base coords
                    try:
                        ok, err = save_levels_file(levels_path, levels)
                        if ok:
                            status_msg = f'Zapisano {levels_path}'
                        else:
                            status_msg = f'Blad zapisu: {err}'
                    except Exception as ex:
                        status_msg = f'Blad zapisu: {ex}'
                if editor and event.key == pygame.K_l:
                    loaded = load_levels_file(levels_path)
                    if loaded:
                        levels[:] = loaded
                        try:
                            assign_default_textures(levels)
                        except Exception:
                            pass
                        platforms, plat_tex_keys, goal, start, spikes, spike_tex_keys = build_level(level_idx)
                        place_player_safe(player, start, platforms)
                        status_msg = 'Wczytano levels.json'
                    else:
                        status_msg = 'Brak levels.json'
                if event.key == pygame.K_ESCAPE:
                    running = False
                # jump is handled per-frame (edge detection) after event processing

            # mouse handling for editor
            if event.type == pygame.MOUSEBUTTONDOWN and editor:
                if event.button == 1:  # left click
                    vx, vy = screen_to_virtual(event.pos)
                    if edit_tool == 'platform':
                        drag_start = (vx, vy)
                        temp_rect = None
                    elif edit_tool == 'spike':
                        drag_start = (vx, vy)
                        temp_rect = None
                    elif edit_tool == 'start':
                        bx = vx // TILE
                        by = vy // TILE
                        levels[level_idx]['start'] = (bx, by)
                        platforms, plat_tex_keys, goal, start, spikes, spike_tex_keys = build_level(level_idx)
                        place_player_safe(player, start, platforms)
                    elif edit_tool == 'goal':
                        gw = 4
                        gh = 4
                        gx = (vx // TILE) - (gw // 2)
                        gy = (vy // TILE) - (gh // 2)
                        levels[level_idx]['goal'] = (gx, gy, gw, gh)
                        platforms, plat_tex_keys, goal, start, spikes, spike_tex_keys = build_level(level_idx)
                    elif edit_tool == 'erase':
                        # try remove spike first
                        removed = False
                        for i, sbase in enumerate(levels[level_idx].get('spikes', [])):
                            rx = sbase[0] * TILE
                            ry = sbase[1] * TILE
                            rw = sbase[2] * TILE
                            rh = sbase[3] * TILE
                            rect = pygame.Rect(rx, ry, rw, rh)
                            if rect.collidepoint((vx, vy)):
                                del levels[level_idx]['spikes'][i]
                                platforms, plat_tex_keys, goal, start, spikes, spike_tex_keys = build_level(level_idx)
                                status_msg = 'Usunieto spike'
                                removed = True
                                break
                        if not removed:
                            # find platform under virtual cursor and remove, but protect ground
                            for i, pbase in enumerate(levels[level_idx].get('platforms', [])):
                                rx = pbase[0] * TILE
                                ry = pbase[1] * TILE
                                rw = pbase[2] * TILE
                                rh = pbase[3] * TILE
                                rect = pygame.Rect(rx, ry, rw, rh)
                                if rect.collidepoint((vx, vy)):
                                    # do not allow deleting the ground platform
                                    try:
                                        if int(pbase[2]) >= GRID_COLS and int(pbase[3]) >= 1:
                                            status_msg = 'Nie mozna usunac podstawy'
                                            removed = True
                                            break
                                    except Exception:
                                        pass
                                    del levels[level_idx]['platforms'][i]
                                    platforms, plat_tex_keys, goal, start, spikes, spike_tex_keys = build_level(level_idx)
                                    status_msg = 'Usunieto platforme'
                                    break

            if event.type == pygame.MOUSEBUTTONUP and editor:
                if event.button == 1 and drag_start:
                    x1, y1 = drag_start  # virtual coords
                    vx2, vy2 = screen_to_virtual(event.pos)
                    rx = min(x1, vx2)
                    ry = min(y1, vy2)
                    rw = abs(vx2 - x1)
                    rh = abs(vy2 - y1)
                    if rw > 4 and rh > 4:
                        bx = rx // TILE
                        by = ry // TILE
                        bw = max(1, (rw + TILE - 1) // TILE)
                        bh = max(1, (rh + TILE - 1) // TILE)
                        if edit_tool == 'platform':
                            # attach default platform texture depending on level group
                            try:
                                if level_idx < 5:
                                    tex = 'stone'
                                else:
                                    tex = 'bricks'
                            except Exception:
                                tex = None
                            if tex:
                                levels[level_idx]['platforms'].append((bx, by, bw, bh, tex))
                            else:
                                levels[level_idx]['platforms'].append((bx, by, bw, bh))
                        elif edit_tool == 'spike':
                            # attach default spike texture depending on level group
                            try:
                                if level_idx < 5:
                                    stex = 'lava@1'
                                else:
                                    stex = 'lava@2'
                            except Exception:
                                stex = None
                            if stex:
                                levels[level_idx].setdefault('spikes', []).append((bx, by, bw, bh, stex))
                            else:
                                levels[level_idx].setdefault('spikes', []).append((bx, by, bw, bh))
                        platforms, plat_tex_keys, goal, start, spikes, spike_tex_keys = build_level(level_idx)
                    drag_start = None
                    temp_rect = None

            if event.type == pygame.MOUSEMOTION and editor and drag_start:
                x1, y1 = drag_start
                vx2, vy2 = screen_to_virtual(event.pos)
                rx = min(x1, vx2)
                ry = min(y1, vy2)
                rw = abs(vx2 - x1)
                rh = abs(vy2 - y1)
                temp_rect = pygame.Rect(rx, ry, rw, rh)

        # editor mouse handling (no-op placeholder)
        if editor:
            pass

        player.update(dt, platforms)

        # handle jump input with edge detection for smoothness
        keys = pygame.key.get_pressed()
        jump_pressed = keys[pygame.K_w] or keys[pygame.K_UP] or keys[pygame.K_SPACE]
        if jump_pressed and not jump_was_pressed:
            # on press edge
            if player.on_ground:
                player.vel_y = -player.jump_strength
                # allow double-jump after leaving ground
                player.can_double_jump = True
                try:
                    if jump_sfx:
                        jump_sfx.play()
                except Exception:
                    pass
            elif player.wall_dir != 0:
                now = pygame.time.get_ticks()
                if now - player.last_jump_time >= player.jump_cooldown:
                    # wall jump: push away from wall with small horizontal impulse
                    player.vel_y = -player.jump_strength
                    player.vel_x = 350 * -player.wall_dir
                    player.last_jump_time = now
                    # after wall-jump allow double-jump again
                    player.can_double_jump = True
                    try:
                        if jump_sfx:
                            jump_sfx.play()
                    except Exception:
                        pass
            elif player.can_double_jump:
                # double jump in air
                player.vel_y = -player.jump_strength
                player.can_double_jump = False
                try:
                    if jump_sfx:
                        jump_sfx.play()
                except Exception:
                    pass
        jump_was_pressed = jump_pressed

        # spike collision -> death (respawn at start)
        for s in spikes:
            if s.colliderect(player.rect):
                # play die SFX
                try:
                    if die_sfx:
                        die_sfx.play()
                except Exception:
                    pass
                # respawn at start (virtual coords)
                place_player_safe(player, start, platforms)
                player.vel_x = player.vel_y = 0
                status_msg = 'Zginąłeś!'
                break

        # check goal collision -> next level
        if player.rect.colliderect(goal):
            level_idx += 1
            if level_idx >= len(levels):
                # loop back to first level
                level_idx = 0
            platforms, plat_tex_keys, goal, start, spikes, spike_tex_keys = build_level(level_idx)
            place_player_safe(player, start, platforms)
            player.vel_x = player.vel_y = 0
            player.can_double_jump = True
            player.on_ground = False

        # select palette based on level groups of 5
        palette = palettes[(level_idx // 5) % len(palettes)]
        bg_color, plat_color, goal_color, player_color = palette

        # input shortcuts
        keys = pygame.key.get_pressed()
        if keys[pygame.K_r]:
            # restart level
            platforms, plat_tex_keys, goal, start, spikes, spike_tex_keys = build_level(level_idx)
            place_player_safe(player, start, platforms)
            player.vel_x = player.vel_y = 0
        # level skip: press N to request skip; actual change occurs after skip_delay_ms
        skip_pressed = keys[pygame.K_n]
        if skip_pressed and not skip_was_pressed:
            # start skip timer
            try:
                skip_request_time = pygame.time.get_ticks()
                status_msg = 'Pomijanie poziomu...'
            except Exception:
                skip_request_time = 0
        skip_was_pressed = skip_pressed
        # if skip requested and delay elapsed, advance level
        if skip_request_time:
            try:
                nowt = pygame.time.get_ticks()
                if nowt - skip_request_time >= skip_delay_ms:
                    level_idx += 1
                    if level_idx >= len(levels):
                        level_idx = 0
                    platforms, plat_tex_keys, goal, start, spikes, spike_tex_keys = build_level(level_idx)
                    place_player_safe(player, start, platforms)
                    player.vel_x = player.vel_y = 0
                    player.can_double_jump = True
                    player.on_ground = False
                    status_msg = 'Poziom pominięty'
                    skip_request_time = 0
            except Exception:
                skip_request_time = 0
        # NOTE: level-skip key disabled to prevent accidental map changes

        # render to virtual surface with current palette
        vs.fill(bg_color)
        # draw platforms (use textures if available)
        for i, p in enumerate(platforms):
            tex_key = None
            try:
                tex_key = plat_tex_keys[i]
            except Exception:
                tex_key = None
            if tex_key and tex_key in platform_textures:
                tex_info = platform_textures[tex_key]
                tex = tex_info['surf']
                tile_factor = int(tex_info.get('tiles', 1))
                desired_w = tile_factor * TILE
                desired_h = tile_factor * TILE
                # compute offset so tiling aligns with global grid
                offset_x = p.left % desired_w
                offset_y = p.top % desired_h
                cache_key = (tex_key, p.width, p.height, offset_x, offset_y)
                surf_cached = platform_cache.get(cache_key)
                if surf_cached is None:
                    try:
                        tex_scaled = tex
                        if tex.get_width() != desired_w or tex.get_height() != desired_h:
                            tex_scaled = pygame.transform.scale(tex, (desired_w, desired_h))
                    except Exception:
                        tex_scaled = tex
                    surf_cached = pygame.Surface((p.width, p.height), pygame.SRCALPHA)
                    # tile starting from -offset to ensure global alignment
                    x = -offset_x
                    while x < p.width:
                        y = -offset_y
                        while y < p.height:
                            try:
                                surf_cached.blit(tex_scaled, (x, y))
                            except Exception:
                                pass
                            y += desired_h
                        x += desired_w
                    platform_cache[cache_key] = surf_cached
                # blit cached surface
                try:
                    vs.blit(surf_cached, p.topleft)
                except Exception:
                    pygame.draw.rect(vs, plat_color, p)
            else:
                pygame.draw.rect(vs, plat_color, p)

        # draw spikes (use per-spike texture if specified)
        spike_color = (200, 30, 30)
        for si, s in enumerate(spikes):
            # choose per-spike texture if provided, otherwise fall back to global 'lava' if available
            spike_tex = None
            try:
                spike_tex = spike_tex_keys[si]
            except Exception:
                spike_tex = None
            # choose fallback based on level group: use lava@1 for levels 1..5, lava@2 for 6..10
            if not spike_tex:
                if level_idx < 5:
                    if 'lava@1' in spike_textures:
                        spike_tex = 'lava@1'
                    elif 'lava' in spike_textures:
                        spike_tex = 'lava'
                elif 5 <= level_idx < 10:
                    if 'lava@2' in spike_textures:
                        spike_tex = 'lava@2'
                    elif 'lava' in spike_textures:
                        spike_tex = 'lava'
            if spike_tex and spike_tex in spike_textures:
                try:
                    tex_info = spike_textures[spike_tex]
                    tex = tex_info['surf']
                    tile_factor = int(tex_info.get('tiles', 1))
                    desired_w = tile_factor * TILE
                    desired_h = tile_factor * TILE
                    offset_x = s.left % desired_w
                    offset_y = s.top % desired_h
                    cache_key = ('spike', spike_tex, s.width, s.height, offset_x, offset_y)
                    surf_cached = spike_cache.get(cache_key)
                    if surf_cached is None:
                        try:
                            tex_scaled = tex
                            if tex.get_width() != desired_w or tex.get_height() != desired_h:
                                tex_scaled = pygame.transform.scale(tex, (desired_w, desired_h))
                        except Exception:
                            tex_scaled = tex
                        surf_cached = pygame.Surface((s.width, s.height), pygame.SRCALPHA)
                        x = -offset_x
                        while x < s.width:
                            y = -offset_y
                            while y < s.height:
                                try:
                                    surf_cached.blit(tex_scaled, (x, y))
                                except Exception:
                                    pass
                                y += desired_h
                            x += desired_w
                        spike_cache[cache_key] = surf_cached
                    try:
                        vs.blit(surf_cached, s.topleft)
                    except Exception:
                        pygame.draw.rect(vs, spike_color, s)
                except Exception:
                    pygame.draw.rect(vs, spike_color, s)
            else:
                pygame.draw.rect(vs, spike_color, s)

        # draw goal (use goal_texture if available)
        if goal_texture is not None:
            try:
                gt = goal_texture
                # always scale the texture to exactly match goal rect size
                gt_scaled = pygame.transform.scale(gt, (goal.width, goal.height))
                vs.blit(gt_scaled, goal.topleft)
            except Exception:
                pygame.draw.rect(vs, goal_color, goal)
        else:
            pygame.draw.rect(vs, goal_color, goal)

        # draw player (use animation frames if available)
        drawn = False
        frames = getattr(player, 'anim_frames', None)
        if frames:
            # choose state
            if not player.on_ground and frames.get('jump'):
                state = 'jump'
            elif abs(player.vel_x) > 1 and frames.get('walk'):
                state = 'walk'
            else:
                state = 'idle'
            lst = frames.get(state) or frames.get('idle') or []
            if lst:
                # update animation timer
                ft = 1.0 / max(1.0, getattr(player, 'anim_fps', 8.0))
                player.anim_timer += dt
                if player.anim_timer >= ft:
                    player.anim_timer -= ft
                    player.anim_index = (player.anim_index + 1) % len(lst)
                img = lst[player.anim_index]
                # visual skin size: 2x2 tiles
                skin_w, skin_h = 2 * TILE, 2 * TILE
                if img.get_size() != (skin_w, skin_h):
                    img = pygame.transform.scale(img, (skin_w, skin_h))
                # flip horizontally if facing left
                if getattr(player, 'facing', 1) < 0:
                    try:
                        img = pygame.transform.flip(img, True, False)
                    except Exception:
                        pass
                # blit so feet align with player's rect bottom and centered horizontally
                blit_x = player.rect.centerx - (skin_w // 2)
                blit_y = player.rect.bottom - skin_h
                vs.blit(img, (blit_x, blit_y))
                drawn = True
        if not drawn:
            pygame.draw.rect(vs, player_color, player.rect)

        # scale virtual surface to real screen and present
        scaled = pygame.transform.scale(vs, (WIDTH, HEIGHT))
        screen.blit(scaled, (0, 0))

        # draw editor temp rect and HUD on actual screen (keeps text crisp)
        scale_x = WIDTH / BASE_WIDTH
        scale_y = HEIGHT / BASE_HEIGHT
        # draw music toggle button
        try:
            btn_col = (100, 200, 100) if not music_paused else (160, 160, 160)
            pygame.draw.rect(screen, btn_col, music_button_rect)
            txt = 'Music: ON' if not music_paused else 'Music: OFF'
            imgb = font.render(txt, True, (10, 10, 10))
            screen.blit(imgb, (music_button_rect.x + 8, music_button_rect.y + 4))
        except Exception:
            pass
        if editor:
            if temp_rect:
                rect_scaled = pygame.Rect(int(temp_rect.x * scale_x), int(temp_rect.y * scale_y), int(temp_rect.w * scale_x), int(temp_rect.h * scale_y))
                pygame.draw.rect(screen, (255, 255, 255), rect_scaled, 2)
            hud_lines = [
                "EDYTOR: ON",
                f"Narzędzie: {edit_tool} (1=platform,2=start,3=goal,4=erase,5=spike)",
                "LPM drag = dodaj platformę, LPM klik = ustaw start/goal",
                "S = zapisz levels.json, L = wczytaj levels.json, E = wyłącz edytor",
            ]
            for i, t in enumerate(hud_lines):
                img = font.render(t, True, (240, 240, 240))
                screen.blit(img, (10, HEIGHT - 20 * (len(hud_lines) - i)))
            if status_msg:
                img = font.render(status_msg, True, (255, 220, 120))
                screen.blit(img, (10, HEIGHT - 20 * (len(hud_lines) + 1)))

        info = [
            f"Level: {level_idx + 1} / {len(levels)}",
            "A/D or ←/→ = lewo/prawo",
            "W / ↑ / Space = skok",
            "Parkour: wall-jump (przytrzymaj przy scianie i nacisnij skok)",
            "R = restart, Esc = wyjscie",
            "N = pomin poziom (1s opóźnienia)",
        ]
        # render info in yellow
        info_color = (255, 220, 0)
        for i, t in enumerate(info):
            img = font.render(t, True, info_color)
            screen.blit(img, (10, 10 + i * 22))

        # optional on-screen debug (disabled by default)
        if SHOW_DEBUG:
            try:
                img = font.render(skin_msg, True, (200, 200, 120))
                screen.blit(img, (10, 10 + (len(info) + 1) * 22))
            except Exception:
                pass
            try:
                used_keys = [k for k in (plat_tex_keys or []) if k]
                missing = [k for k in used_keys if k not in platform_textures_keys]
                loaded_txt = ','.join(platform_textures_keys) if platform_textures_keys else 'none'
                used_txt = ','.join(used_keys) if used_keys else 'none'
                miss_txt = ','.join(missing) if missing else 'none'
                plat_msg = f"Plat tex - loaded:[{loaded_txt}] used_this_level:[{used_txt}] missing:[{miss_txt}]"
                img2 = font.render(plat_msg, True, (200, 200, 120))
                screen.blit(img2, (10, 10 + (len(info) + 3) * 22))
            except Exception:
                pass

        pygame.display.flip()

    # autosave levels on exit
    try:
        ok, err = save_levels_file(levels_path, levels)
        if not ok:
            print(f"Błąd zapisu levels.json przy zamykaniu: {err}")
        else:
            print(f"Zapisano levels: {levels_path}")
    except Exception as e:
        print(f"Nie udało się zapisać levels.json przy zamykaniu: {e}")

    pygame.quit()
    sys.exit()


if __name__ == '__main__':
    main()

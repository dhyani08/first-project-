# first-project-
This is my first GitHub project 
import pygame
import random
import math

pygame.init()

# -----------------------------
# SETTINGS
# -----------------------------
WIDTH = 1000
HEIGHT = 650
FPS = 60

PLAYER_SPEED = 5
BOT_SPEED = 2
BLASTER_COOLDOWN = 250
MAX_HEALTH = 100

screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Neon Squad - Team Deathmatch")
clock = pygame.time.Clock()

font = pygame.font.Font(None, 32)
big_font = pygame.font.Font(None, 60)


# -----------------------------
# COLORS
# -----------------------------
BACKGROUND = (25, 28, 35)
BLUE = (60, 160, 255)
RED = (255, 80, 90)
WHITE = (240, 240, 240)
GREEN = (70, 220, 120)
YELLOW = (255, 220, 70)
GREY = (70, 75, 85)


# -----------------------------
# PLAYER CLASS
# -----------------------------
class Player:
    def __init__(self, x, y, team):
        self.x = x
        self.y = y
        self.team = team
        self.health = MAX_HEALTH
        self.score = 0
        self.radius = 18
        self.last_shot = 0
        self.alive = True
        self.respawn_timer = 0

    def move(self, dx, dy):
        if not self.alive:
            return

        length = math.sqrt(dx ** 2 + dy ** 2)

        if length > 0:
            dx /= length
            dy /= length

        self.x += dx * PLAYER_SPEED
        self.y += dy * PLAYER_SPEED

        self.x = max(self.radius, min(WIDTH - self.radius, self.x))
        self.y = max(self.radius, min(HEIGHT - self.radius, self.y))

    def shoot(self, target_x, target_y, bullets):
        current_time = pygame.time.get_ticks()

        if not self.alive:
            return

        if current_time - self.last_shot < BLASTER_COOLDOWN:
            return

        self.last_shot = current_time

        dx = target_x - self.x
        dy = target_y - self.y

        distance = math.sqrt(dx ** 2 + dy ** 2)

        if distance == 0:
            return

        dx /= distance
        dy /= distance

        bullets.append(
            Bullet(
                self.x,
                self.y,
                dx,
                dy,
                self.team
            )
        )

    def take_damage(self, amount):
        self.health -= amount

        if self.health <= 0:
            self.health = 0
            self.alive = False
            self.respawn_timer = pygame.time.get_ticks() + 2000


# -----------------------------
# BULLET CLASS
# -----------------------------
class Bullet:
    def __init__(self, x, y, dx, dy, team):
        self.x = x
        self.y = y
        self.dx = dx
        self.dy = dy
        self.team = team
        self.speed = 12
        self.radius = 5
        self.damage = 25

    def update(self):
        self.x += self.dx * self.speed
        self.y += self.dy * self.speed

    def draw(self):
        pygame.draw.circle(
            screen,
            YELLOW,
            (int(self.x), int(self.y)),
            self.radius
        )


# -----------------------------
# CREATE TEAMS
# -----------------------------
player = Player(200, HEIGHT // 2, "BLUE")

bots = []

for i in range(3):
    bots.append(
        Player(
            random.randint(100, 400),
            random.randint(100, HEIGHT - 100),
            "BLUE"
        )
    )

for i in range(4):
    bots.append(
        Player(
            random.randint(WIDTH - 400, WIDTH - 100),
            random.randint(100, HEIGHT - 100),
            "RED"
        )
    )

bullets = []


# -----------------------------
# RESPAWN
# -----------------------------
def respawn(character):
    if character.team == "BLUE":
        character.x = random.randint(100, 300)
    else:
        character.x = random.randint(WIDTH - 300, WIDTH - 100)

    character.y = random.randint(100, HEIGHT - 100)
    character.health = MAX_HEALTH
    character.alive = True


# -----------------------------
# DRAW CHARACTER
# -----------------------------
def draw_character(character):
    if not character.alive:
        return

    color = BLUE if character.team == "BLUE" else RED

    pygame.draw.circle(
        screen,
        color,
        (int(character.x), int(character.y)),
        character.radius
    )

    # Health bar
    bar_width = 40
    health_width = int(
        bar_width * character.health / MAX_HEALTH
    )

    pygame.draw.rect(
        screen,
        GREY,
        (character.x - 20, character.y - 30, bar_width, 6)
    )

    pygame.draw.rect(
        screen,
        GREEN,
        (character.x - 20, character.y - 30, health_width, 6)
    )


# -----------------------------
# BOT AI
# -----------------------------
def bot_behavior(bot):
    if not bot.alive:
        return

    targets = []

    if bot.team != player.team and player.alive:
        targets.append(player)

    for other in bots:
        if other != bot and other.team != bot.team and other.alive:
            targets.append(other)

    if not targets:
        return

    target = min(
        targets,
        key=lambda target:
        math.hypot(target.x - bot.x, target.y - bot.y)
    )

    dx = target.x - bot.x
    dy = target.y - bot.y

    distance = math.hypot(dx, dy)

    if distance > 180:
        if distance != 0:
            dx /= distance
            dy /= distance

            bot.x += dx * BOT_SPEED
            bot.y += dy * BOT_SPEED

    bot.shoot(target.x, target.y, bullets)


# -----------------------------
# GAME LOOP
# -----------------------------
running = True

while running:

    clock.tick(FPS)

    for event in pygame.event.get():

        if event.type == pygame.QUIT:
            running = False

        if event.type == pygame.MOUSEBUTTONDOWN:
            if event.button == 1:
                mouse_x, mouse_y = pygame.mouse.get_pos()
                player.shoot(mouse_x, mouse_y, bullets)

    # -------------------------
    # PLAYER MOVEMENT
    # -------------------------
    keys = pygame.key.get_pressed()

    dx = 0
    dy = 0

    if keys[pygame.K_w]:
        dy -= 1

    if keys[pygame.K_s]:
        dy += 1

    if keys[pygame.K_a]:
        dx -= 1

    if keys[pygame.K_d]:
        dx += 1

    player.move(dx, dy)

    # -------------------------
    # BOT AI
    # -------------------------
    for bot in bots:
        bot_behavior(bot)

    # -------------------------
    # BULLET UPDATE
    # -------------------------
    for bullet in bullets[:]:

        bullet.update()

        if (
            bullet.x < 0
            or bullet.x > WIDTH
            or bullet.y < 0
            or bullet.y > HEIGHT
        ):
            bullets.remove(bullet)
            continue

        characters = [player] + bots

        for character in characters:

            if not character.alive:
                continue

            if character.team == bullet.team:
                continue

            distance = math.hypot(
                character.x - bullet.x,
                character.y - bullet.y
            )

            if distance < character.radius + bullet.radius:

                character.take_damage(bullet.damage)

                if not character.alive:

                    if bullet.team == player.team:
                        player.score += 1

                if bullet in bullets:
                    bullets.remove(bullet)

                break

    # -------------------------
    # RESPAWN
    # -------------------------
    characters = [player] + bots

    for character in characters:

        if not character.alive:
            if pygame.time.get_ticks() >= character.respawn_timer:
                respawn(character)

    # -------------------------
    # DRAW
    # -------------------------
    screen.fill(BACKGROUND)

    # Arena dividing line
    pygame.draw.line(
        screen,
        GREY,
        (WIDTH // 2, 0),
        (WIDTH // 2, HEIGHT),
        2
    )

    # Draw bullets
    for bullet in bullets:
        bullet.draw()

    # Draw characters
    for character in characters:
        draw_character(character)

    # Score
    blue_score = sum(
        1 for bot in bots
        if bot.team == "BLUE"
    )

    score_text = font.render(
        f"BLUE: {player.score}     RED TEAM",
        True,
        WHITE
    )

    screen.blit(score_text, (20, 20))

    # Controls
    controls = font.render(
        "WASD = Move   |   Left Click = Blaster",
        True,
        WHITE
    )

    screen.blit(
        controls,
        (20, HEIGHT - 40)
    )

    pygame.display.flip()


pygame.quit()

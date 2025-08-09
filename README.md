# EVA-C64-Cluster
# Artillery Duel Game using pygame with terrain and user controls
# Lizenzinformationen:
# Dawid Szymon Kazek
# sq-hq sq.z.o.o. 2024

import pygame
import math
import random

# Initialize pygame
pygame.init()

# Screen dimensions
SCREEN_WIDTH = 1920
SCREEN_HEIGHT = 1080
BACKGROUND_COLOR = (135, 206, 235)

# Colors
BLACK = (0, 0, 0)
YELLOW = (255, 255, 0)
RED = (255, 0, 0)
GREEN = (0, 255, 0)
BROWN = (139, 69, 19)
GRASS_GREEN = (34, 139, 34)
WHITE = (255, 255, 255)

# Constants
TILE_SIZE = 10
GRAVITY = 9.8
EXPLOSION_RADIUS = 2
PLAYER_HEIGHT_TILES = 4 # Höhe des Spielerkörpers in Tiles

# Player class
class Player:
    def __init__(self, name, start_x_percent, color):
        self.name = name
        self.start_x_percent = start_x_percent # Startposition als Prozent der Bildschirmbreite
        self.position = (0, 0) # Wird später im generate_terrain gesetzt
        self.color = color
        self.health = 100
        self.turret_angle = 90
        self.power = 50

    def is_alive(self):
        return self.health > 0

# Game class
class Game:
    def __init__(self, players):
        self.players = players
        self.screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
        pygame.display.set_caption("Artillery Duel")
        self.terrain = self._generate_terrain_with_player_placement()
        self.font = pygame.font.Font(None, 24)
        self.current_player_index = 0

    def _generate_terrain_with_player_placement(self):
            terrain = [[0 for _ in range(SCREEN_WIDTH // TILE_SIZE)] for _ in range(SCREEN_HEIGHT // TILE_SIZE)]
            max_initial_height = 25
            min_initial_height = 15
            max_height_change = 2

            player_positions_x_tiles = []
            for player in self.players:
                player_x = int(player.start_x_percent * SCREEN_WIDTH // TILE_SIZE)
                player_positions_x_tiles.append(player_x)
                print(f"{player.name} soll bei x-Tile: {player_x} platziert werden.")

            # 1. Generiere das Grundterrain
            heights = [random.randint(min_initial_height, max_initial_height) for _ in range(SCREEN_WIDTH // TILE_SIZE)]
            for x in range(SCREEN_WIDTH // TILE_SIZE):
                change = random.randint(-max_height_change, max_height_change)
                if x > 0:
                    heights[x] = max(0, min(SCREEN_HEIGHT // TILE_SIZE, heights[x-1] + change))
                for y in range(SCREEN_HEIGHT // TILE_SIZE - heights[x], SCREEN_HEIGHT // TILE_SIZE):
                    terrain[y][x] = 2 if y == SCREEN_HEIGHT // TILE_SIZE - heights[x] else 1

            # 2. Forme das Terrain um die Spielerpositionen
            for i, player in enumerate(self.players):
                player_x_tile = player_positions_x_tiles[i]

                # Finde die Höhe des Terrains an der Spielerposition
                ground_y_tile = SCREEN_HEIGHT // TILE_SIZE
                for y in range(SCREEN_HEIGHT // TILE_SIZE):
                    if terrain[y][player_x_tile] != 0:
                        ground_y_tile = y
                        break

                # Setze die Spielerposition
                player_base_y_tile = ground_y_tile - PLAYER_HEIGHT_TILES
                player.position = (player_x_tile * TILE_SIZE + TILE_SIZE // 2, player_base_y_tile * TILE_SIZE)
                print(f"{player.name} platziert bei: {player.position} (Tile: {player_x_tile}, {player_base_y_tile})")

                # Flatten the terrain around the player and ensure correct tile order
                for x_offset in range(-2, 3):
                    for y_offset in range(0, PLAYER_HEIGHT_TILES + 2):
                        tile_x = player_x_tile + x_offset
                        tile_y = ground_y_tile - y_offset
                        if 0 <= tile_x < SCREEN_WIDTH // TILE_SIZE and 0 <= tile_y < SCREEN_HEIGHT // TILE_SIZE:
                            if y_offset == 0:
                                terrain[tile_y][tile_x] = 2  # Oberste Schicht ist Gras
                            elif y_offset > 0 and tile_y < SCREEN_HEIGHT // TILE_SIZE:
                                terrain[tile_y][tile_x] = 1  # Darunter ist Dirt
                            elif tile_y >= SCREEN_HEIGHT // TILE_SIZE -1 and y_offset > 0: # Sicherheit für unterste Reihe
                                terrain[tile_y][tile_x] = 1

            return terrain

    def draw_terrain(self):
        for y in range(SCREEN_HEIGHT // TILE_SIZE):
            for x in range(SCREEN_WIDTH // TILE_SIZE):
                if self.terrain[y][x] == 1:
                    pygame.draw.rect(self.screen, BROWN, (x * TILE_SIZE, y * TILE_SIZE, TILE_SIZE, TILE_SIZE))
                elif self.terrain[y][x] == 2:
                    pygame.draw.rect(self.screen, GRASS_GREEN, (x * TILE_SIZE, y * TILE_SIZE, TILE_SIZE, TILE_SIZE))

    def draw_turret(self, player):
        x, y = player.position
        length = 30
        angle_rad = math.radians(player.turret_angle)
        turret_base_y = y - PLAYER_HEIGHT_TILES * TILE_SIZE
        end_x = x + length * math.cos(angle_rad)
        end_y = turret_base_y - length * math.sin(angle_rad)
        pygame.draw.line(self.screen, BLACK, (x, turret_base_y), (end_x, end_y), 5)

    def draw_player(self, player):
        x, y = player.position
        pygame.draw.rect(self.screen, player.color, (x - 15, y - PLAYER_HEIGHT_TILES * TILE_SIZE, 30, PLAYER_HEIGHT_TILES * TILE_SIZE))
        self.draw_turret(player)

    def draw_bullet(self, bullet):
        x, y = bullet["position"]
        pygame.draw.circle(self.screen, YELLOW, (int(x), int(y)), 5)

    def handle_input(self, player, keys):
        if keys[pygame.K_LEFT]:
            player.turret_angle = min((player.turret_angle + 2) % 360, 180)
        if keys[pygame.K_RIGHT]:
            player.turret_angle = max((player.turret_angle - 2) % 360, 0)
        if keys[pygame.K_UP]:
            player.power = min(player.power + 1, 200)
        if keys[pygame.K_DOWN]:
            player.power = max(player.power - 1, 0)

    def display_stats(self, player, position):
        angle_text = self.font.render(f"Angle: {player.turret_angle:.1f}", True, WHITE)
        power_text = self.font.render(f"Power: {player.power:.1f}", True, WHITE)
        health_text = self.font.render(f"Health: {player.health}", True, WHITE)
        name_text = self.font.render(f"{player.name}", True, player.color)
        self.screen.blit(name_text, (position[0], position[1]))
        self.screen.blit(angle_text, (position[0], position[1] + 30))
        self.screen.blit(power_text, (position[0], position[1] + 60))
        self.screen.blit(health_text, (position[0], position[1] + 90))

    def destroy_terrain(self, x, y):
        radius_tiles = EXPLOSION_RADIUS
        center_x = int(x)
        center_y = int(y)
        for i in range(max(0, center_y - radius_tiles), min(SCREEN_HEIGHT // TILE_SIZE, center_y + radius_tiles + 1)):
            for j in range(max(0, center_x - radius_tiles), min(SCREEN_WIDTH // TILE_SIZE, center_x + radius_tiles + 1)):
                dist_sq = (i - center_y) ** 2 + (j - center_x) ** 2
                if dist_sq <= radius_tiles ** 2:
                    self.terrain[i][j] = 0

    def take_turn(self, shooter, target):
        angle_rad = math.radians(shooter.turret_angle)
        start_x = shooter.position[0]
        start_y = shooter.position[1] - PLAYER_HEIGHT_TILES * TILE_SIZE

        turret_length = 30
        bullet_start_offset_x = turret_length * math.cos(angle_rad)
        bullet_start_offset_y = -turret_length * math.sin(angle_rad)

        bullet_start_x = start_x + bullet_start_offset_x
        bullet_start_y = start_y + bullet_start_offset_y

        bullet = {
            "position": [bullet_start_x, bullet_start_y],
            "velocity": [shooter.power * math.cos(angle_rad) * 0.05, -shooter.power * math.sin(angle_rad) * 0.05]
        }

        while 0 <= bullet["position"][0] < SCREEN_WIDTH and bullet["position"][1] < SCREEN_HEIGHT:
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    pygame.quit()
                    return False

            bullet["position"][0] += bullet["velocity"][0]
            bullet["position"][1] += bullet["velocity"][1]
            bullet["velocity"][1] += GRAVITY * 0.1

            self.screen.fill(BACKGROUND_COLOR)
            self.draw_terrain()
            self.draw_player(self.players[0])
            self.draw_player(self.players[1])
            self.draw_bullet(bullet)
            self.display_stats(self.players[0], (10, SCREEN_HEIGHT - 120))
            self.display_stats(self.players[1], (SCREEN_WIDTH - 160, SCREEN_HEIGHT - 120))
            pygame.display.flip()
            pygame.time.delay(20)

            bullet_x_tile = int(bullet["position"][0] // TILE_SIZE)
            bullet_y_tile = int(bullet["position"][1] // TILE_SIZE)

            if 0 <= bullet_y_tile < SCREEN_HEIGHT // TILE_SIZE and 0 <= bullet_x_tile < SCREEN_WIDTH // TILE_SIZE and self.terrain[bullet_y_tile][bullet_x_tile] != 0:
                self.destroy_terrain(bullet_x_tile, bullet_y_tile)
                break

            target_rect = pygame.Rect(target.position[0] - 15, target.position[1] - PLAYER_HEIGHT_TILES * TILE_SIZE, 30, PLAYER_HEIGHT_TILES * TILE_SIZE)
            bullet_rect = pygame.Rect(int(bullet["position"][0]) - 5, int(bullet["position"][1]) - 5, 10, 10)
            if bullet_rect.colliderect(target_rect):
                damage = random.randint(20, 50)
                target.health -= damage
                print(f"Treffer! {target.name} hat {damage} Schaden genommen und hat noch {target.health} Gesundheit.")
                self.destroy_terrain(bullet_x_tile, bullet_y_tile)
                break

        if not (0 <= bullet["position"][0] < SCREEN_WIDTH and bullet["position"][1] < SCREEN_HEIGHT):
            print(f"Schuss von {shooter.name} ging daneben.")

        return True

    def distance(self, pos1, pos2):
        return abs(pos1 - pos2)

    def play(self):
        running = True
        while running and all(player.is_alive() for player in self.players):
            current_player = self.players[self.current_player_index]
            target_player = self.players[(self.current_player_index + 1) % len(self.players)]

            while running:
                for event in pygame.event.get():
                    if event.type == pygame.QUIT:
                        running = False
                    if event.type == pygame.KEYDOWN and event.key == pygame.K_SPACE:
                        if self.take_turn(current_player, target_player):
                            self.current_player_index = (self.current_player_index + 1) % len(self.players)
                            break
                if not running:
                    break

                keys = pygame.key.get_pressed()
                self.handle_input(current_player, keys)

                self.screen.fill(BACKGROUND_COLOR)
                self.draw_terrain()
                self.draw_player(self.players[0])
                self.draw_player(self.players[1])
                self.display_stats(self.players[0], (10, SCREEN_HEIGHT - 120))
                self.display_stats(self.players[1], (SCREEN_WIDTH - 160, SCREEN_HEIGHT - 120))
                pygame.display.flip()
                pygame.time.delay(30)

        winner = [player for player in self.players if player.is_alive()]
        if winner:
            print(f"Glückwunsch, {winner[0].name} hat gewonnen!")
        else:
            print("Das Spiel ist unentschieden!")

        pygame.quit()

def main():
    player1 = Player(name="Spieler 1", start_x_percent=0.2, color=RED)
    player2 = Player(name="Spieler 2", start_x_percent=0.8, color=GREEN)

    game = Game(players=[player1, player2])
    game.play()

if __name__ == "__main__":
    main()

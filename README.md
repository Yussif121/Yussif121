import pygame
import random
import sys
import json
import os

# Initialize Pygame
pygame.init()
pygame.mixer.init()  # Initialize sound mixer

# Screen dimensions
WIDTH = 800
HEIGHT = 600
GRID_SIZE = 20
GRID_WIDTH = WIDTH // GRID_SIZE
GRID_HEIGHT = HEIGHT // GRID_SIZE

# Colors
BLACK = (0, 0, 0)
GREEN = (0, 255, 0)
RED = (255, 0, 0)
WHITE = (255, 255, 255)
GRAY = (128, 128, 128)
BLUE = (0, 0, 255)
YELLOW = (255, 255, 0)
PURPLE = (128, 0, 128)

# Create the screen
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption('Advanced Snake Game')

# Fonts
font = pygame.font.Font(None, 74)
small_font = pygame.font.Font(None, 36)

# Clock to control game speed
clock = pygame.time.Clock()

# Sound effects
try:
    eat_sound = pygame.mixer.Sound('eat.wav')
    wall_hit_sound = pygame.mixer.Sound('wall_hit.wav')
    powerup_sound = pygame.mixer.Sound('powerup.wav')
    poison_sound = pygame.mixer.Sound('poison.wav')
except:
    # Placeholder for sound loading errors
    eat_sound = pygame.mixer.Sound(buffer=bytearray([]))
    wall_hit_sound = pygame.mixer.Sound(buffer=bytearray([]))
    powerup_sound = pygame.mixer.Sound(buffer=bytearray([]))
    poison_sound = pygame.mixer.Sound(buffer=bytearray([]))

# High Score Management
HIGH_SCORE_FILE = 'high_score.json'

def load_high_score():
    try:
        with open(HIGH_SCORE_FILE, 'r') as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return 0

def save_high_score(score):
    with open(HIGH_SCORE_FILE, 'w') as f:
        json.dump(score, f)

def draw_game_over(score, death_reason):
    screen.fill(BLACK)
    game_over_text = font.render('GAME OVER', True, RED)
    score_text = small_font.render(f'Score: {score}', True, WHITE)
    reason_text = small_font.render(str(death_reason), True, WHITE)
    restart_text = small_font.render('Press SPACE to Restart', True, WHITE)
    exit_text = small_font.render('Press ESC to Exit', True, WHITE)

    screen.blit(game_over_text, (WIDTH//2 - game_over_text.get_width()//2, HEIGHT//2 - 150))
    screen.blit(score_text, (WIDTH//2 - score_text.get_width()//2, HEIGHT//2 - 50))
    screen.blit(reason_text, (WIDTH//2 - reason_text.get_width()//2, HEIGHT//2))
    screen.blit(restart_text, (WIDTH//2 - restart_text.get_width()//2, HEIGHT//2 + 100))
    screen.blit(exit_text, (WIDTH//2 - exit_text.get_width()//2, HEIGHT//2 + 150))
    
    pygame.display.flip()

class DifficultyManager:
    def __init__(self):
        self.level = 1
        self.obstacles_count = 3
        self.speed_increment = 0

    def update(self, score):
        # Increase difficulty every 10 points
        new_level = score // 10 + 1
        if new_level > self.level:
            self.level = new_level
            self.obstacles_count = min(10, 3 + self.level)
            self.speed_increment = min(5, self.level)

class PowerUp:
    def __init__(self):
        self.pos = None
        self.type = None
        self.timer = 0
        self.spawn_timer = random.randint(50, 200)

    def spawn(self, snake):
        if self.spawn_timer > 0:
            self.spawn_timer -= 1
            return None

        # Reset spawn timer
        self.spawn_timer = random.randint(50, 200)

        # Randomly choose power-up type
        power_up_types = ['speed_boost', 'extra_points', 'invincibility', 'poison']
        self.type = random.choice(power_up_types)

        # Find a random position not on the snake
        while True:
            pos = (random.randint(0, GRID_WIDTH-1), random.randint(0, GRID_HEIGHT-1))
            if pos not in snake.body:
                self.pos = pos
                self.timer = 300  # 5 seconds of effect
                return self

        return None

    def draw(self, screen):
        if self.pos:
            color_map = {
                'speed_boost': BLUE,
                'extra_points': YELLOW,
                'invincibility': (255, 165, 0),  # Orange
                'poison': PURPLE
            }
            power_up_rect = pygame.Rect(
                self.pos[0]*GRID_SIZE, 
                self.pos[1]*GRID_SIZE, 
                GRID_SIZE-1, GRID_SIZE-1
            )
            pygame.draw.rect(screen, color_map[self.type], power_up_rect)

class Food:
    def __init__(self):
        self.pos = None
        self.type = None

    def spawn(self, snake, obstacles):
        while True:
            pos = (random.randint(0, GRID_WIDTH-1), random.randint(0, GRID_HEIGHT-1))
            if pos not in snake.body and pos not in obstacles.positions:
                self.pos = pos
                # Different food types with different effects
                self.type = random.choices(
                    ['normal', 'bonus', 'growth'], 
                    weights=[0.7, 0.2, 0.1]
                )[0]
                break

    def draw(self, screen):
        color_map = {
            'normal': RED,
            'bonus': YELLOW,
            'growth': GREEN
        }
        food_rect = pygame.Rect(
            self.pos[0]*GRID_SIZE, 
            self.pos[1]*GRID_SIZE, 
            GRID_SIZE-1, GRID_SIZE-1
        )
        pygame.draw.rect(screen, color_map[self.type], food_rect)

class Obstacle:
    def __init__(self, snake, num_obstacles=None):
        # Generate obstacles avoiding snake body
        self.positions = []
        if num_obstacles is None:
            num_obstacles = random.randint(3, 7)
        
        for _ in range(num_obstacles):
            while True:
                pos = (random.randint(0, GRID_WIDTH-1), random.randint(0, GRID_HEIGHT-1))
                if pos not in self.positions and pos not in snake.body:
                    self.positions.append(pos)
                    break

    def draw(self, screen):
        for pos in self.positions:
            obstacle_rect = pygame.Rect(
                pos[0]*GRID_SIZE, 
                pos[1]*GRID_SIZE, 
                GRID_SIZE-1, GRID_SIZE-1
            )
            pygame.draw.rect(screen, GRAY, obstacle_rect)

class Snake:
    def __init__(self):
        self.body = [(GRID_WIDTH // 2, GRID_HEIGHT // 2)]
        self.direction = (1, 0)
        self.grow_to = 0
        self.alive = True
        self.death_reason = None
        self.speed_boost = False
        self.invincibility = False
        self.speed_boost_timer = 0
        self.invincibility_timer = 0
        self.is_poisoned = False
        self.poison_timer = 0

    def wrap_around(self):
        head = self.body[0]
        new_head = list(head)
        
        if head[0] < 0:
            new_head[0] = GRID_WIDTH - 1
        elif head[0] >= GRID_WIDTH:
            new_head[0] = 0
        
        if head[1] < 0:
            new_head[1] = GRID_HEIGHT - 1
        elif head[1] >= GRID_HEIGHT:
            new_head[1] = 0
        
        return tuple(new_head)

    def move(self, obstacles):
        if not self.alive:
            return False
            
        head = self.body[0]
        
        new_head = (
            head[0] + self.direction[0],
            head[1] + self.direction[1]
        )
        
        # Teleportation instead of wall death
        if (new_head[0] < 0 or new_head[0] >= GRID_WIDTH or 
            new_head[1] < 0 or new_head[1] >= GRID_HEIGHT):
            if not self.invincibility:
                new_head = self.wrap_around()
        
        # Check if snake hits itself
        if new_head in self.body:
            if not self.invincibility:
                self.alive = False
                self.death_reason = "You hit yourself!"
                return False
        
        # Check obstacle collision
        if new_head in [obs for obs in obstacles.positions]:
            if not self.invincibility:
                self.alive = False
                self.death_reason = "You hit an obstacle!"
                return False
        
        # Add new head
        self.body.insert(0, new_head)
        
        # Grow snake if needed
        if self.grow_to > 0:
            self.grow_to -= 1
        else:
            self.body.pop()
        
        return True

    def change_direction(self, new_direction):
        if self.is_poisoned:
            # Reverse the direction when poisoned
            new_direction = (new_direction[0] * -1, new_direction[1] * -1)
        
        # Prevent 180-degree turns
        if (new_direction[0] * -1, new_direction[1] * -1) != self.direction:
            self.direction = new_direction

def main():
    # Load high score
    high_score = load_high_score()

    # Initialize snake, food, obstacles, and power-ups
    snake = Snake()
    food = Food()
    difficulty = DifficultyManager()
    obstacles = Obstacle(snake, difficulty.obstacles_count)
    power_up = PowerUp()
    score = 0
    game_speed = 10
    game_over = False

    # Spawn initial food
    food.spawn(snake, obstacles)

    # Game loop
    while True:
        # Event handling
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            
            if game_over:
                if event.type == pygame.KEYDOWN:
                    if event.key == pygame.K_SPACE:
                        # Restart game
                        snake = Snake()
                        food = Food()
                        difficulty = DifficultyManager()
                        obstacles = Obstacle(snake, difficulty.obstacles_count)
                        food.spawn(snake, obstacles)
                        score = 0
                        game_speed = 10
                        game_over = False
                    elif event.key == pygame.K_ESCAPE:
                        pygame.quit()
                        sys.exit()
                continue

            # Handle key presses for direction
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_UP:
                    snake.change_direction((0, -1))
                elif event.key == pygame.K_DOWN:
                    snake.change_direction((0, 1))
                elif event.key == pygame.K_LEFT:
                    snake.change_direction((-1, 0))
                elif event.key == pygame.K_RIGHT:
                    snake.change_direction((1, 0))
                elif event.key == pygame.K_ESCAPE:
                    pygame.quit()
                    sys.exit()

        if not game_over:
            # Move snake
            if not snake.move(obstacles):
                game_over = True
                # Update high score
                high_score = max(high_score, score)
                save_high_score(high_score)

            # Power-up mechanics
            if power_up.pos is None:
                power_up.spawn(snake)
            else:
                # Check power-up collection
                if snake.body[0] == power_up.pos:
                    powerup_sound.play()
                    if power_up.type == 'speed_boost':
                        snake.speed_boost = True
                        snake.speed_boost_timer = 300  # 5 seconds
                        game_speed = 15
                    elif power_up.type == 'extra_points':
                        score += 5  # Bonus points
                    elif power_up.type == 'invincibility':
                        snake.invincibility = True
                        snake.invincibility_timer = 300  # 5 seconds
                    elif power_up.type == 'poison':
                        poison_sound.play()
                        # Reverse controls for a few seconds
                        snake.poison_timer = 300  # 5 seconds
                        snake.is_poisoned = True
                    
                    power_up.pos = None

                # Manage power-up timers
                if snake.speed_boost_timer > 0:
                    snake.speed_boost_timer -= 1
                    if snake.speed_boost_timer == 0:
                        snake.speed_boost = False
                        game_speed = 10

                if snake.invincibility_timer > 0:
                    snake.invincibility_timer -= 1
                    if snake.invincibility_timer == 0:
                        snake.invincibility = False

                if snake.poison_timer > 0:
                    snake.poison_timer -= 1
                    if snake.poison_timer == 0:
                        snake.is_poisoned = False

            # Check if snake eats food
            if snake.body[0] == food.pos:
                eat_sound.play()
                if food.type == 'normal':
                    snake.grow_to += 1
                    score += 1
                elif food.type == 'bonus':
                    snake.grow_to += 1
                    score += 3
                elif food.type == 'growth':
                    snake.grow_to += 3
                    score += 2
                
                # Update difficulty and spawn new food
                difficulty.update(score)
                game_speed = 10 + difficulty.speed_increment
                food.spawn(snake, obstacles)

        # Clear screen
        screen.fill(BLACK)

        # Draw grid lines
        for x in range(0, WIDTH, GRID_SIZE):
            pygame.draw.line(screen, GRAY, (x, 0), (x, HEIGHT), 1)
        for y in range(0, HEIGHT, GRID_SIZE):
            pygame.draw.line(screen, GRAY, (0, y), (WIDTH, y), 1)

        # Draw obstacles
        obstacles.draw(screen)

        # Draw power-up
        if power_up.pos:
            power_up.draw(screen)

        # Draw snake
        snake_color = GREEN
        if snake.is_poisoned:
            snake_color = PURPLE  # Different color when poisoned
    
        for segment in snake.body:
            rect = pygame.Rect(segment[0]*GRID_SIZE, segment[1]*GRID_SIZE, GRID_SIZE-1, GRID_SIZE-1)
            pygame.draw.rect(screen, snake_color, rect)

        # Draw food
        food.draw(screen)

        # Draw score and high score
        score_text = small_font.render(f'Score: {score}', True, WHITE)
        high_score_text = small_font.render(f'High Score: {high_score}', True, WHITE)
        
        # Draw difficulty level
        level_text = small_font.render(f'Level: {difficulty.level}', True, WHITE)
        
        screen.blit(score_text, (10, 10))
        screen.blit(high_score_text, (10, 50))
        screen.blit(level_text, (10, 90))

        # If game is over, show game over screen
        if game_over:
            draw_game_over(score, snake.death_reason)
        else:
            # Update display
            pygame.display.flip()

        # Control game speed
        clock.tick(game_speed)

if __name__ == '__main__':
    main()

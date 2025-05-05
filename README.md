import random
import sys
from enum import Enum

# Типы клеток карты
class CellType(Enum):
    EMPTY = ' '
    WALL = '#'
    PLAYER = 'P'
    ENEMY = 'E'
    ITEM = 'I'

# Направления движения
class Direction(Enum):
    UP = (0, -1)
    DOWN = (0, 1)
    LEFT = (-1, 0)
    RIGHT = (1, 0)

# Базовый класс для персонажей
class Character:
    def __init__(self, x, y, hp, mp, armor, damage, symbol):
        self.x = x
        self.y = y
        self.max_hp = hp
        self.hp = hp
        self.max_mp = mp
        self.mp = mp
        self.armor = armor
        self.damage = damage
        self.symbol = symbol
    
    def take_damage(self, damage):
        actual_damage = max(1, damage - self.armor)
        self.hp -= actual_damage
        return actual_damage
    
    def is_alive(self):
        return self.hp > 0
    
    def move(self, dx, dy, game_map):
        new_x, new_y = self.x + dx, self.y + dy
        if game_map.is_valid_position(new_x, new_y) and not game_map.is_wall(new_x, new_y):
            self.x = new_x
            self.y = new_y
            return True
        return False

# Игрок
class Player(Character):
    def __init__(self, x, y):
        super().__init__(x, y, 100, 50, 5, 10, CellType.PLAYER)
        self.level = 1
        self.exp = 0
    
    def attack(self, enemy):
        damage = random.randint(self.damage - 2, self.damage + 2)
        actual_damage = enemy.take_damage(damage)
        return actual_damage
    
    def gain_exp(self, amount):
        self.exp += amount
        if self.exp >= self.level * 100:
            self.level_up()
    
    def level_up(self):
        self.level += 1
        self.max_hp += 20
        self.hp = self.max_hp
        self.max_mp += 10
        self.mp = self.max_mp
        self.damage += 3
        self.armor += 1
        self.exp = 0
        print(f"Level up! Now you are level {self.level}")

# Враг
class Enemy(Character):
    def __init__(self, x, y):
        super().__init__(x, y, 50, 0, 2, 5, CellType.ENEMY)
    
    def attack(self, player):
        damage = random.randint(self.damage - 1, self.damage + 1)
        actual_damage = player.take_damage(damage)
        return actual_damage

# Карта игры
class GameMap:
    def __init__(self, width, height, auto_generate=True):
        self.width = width
        self.height = height
        self.cells = [[CellType.EMPTY for _ in range(height)] for _ in range(width)]
        self.auto_generated = auto_generate
        
        if auto_generate:
            self.generate_map()
    
    def generate_map(self):
        # Генерация стен
        for x in range(self.width):
            for y in range(self.height):
                if random.random() < 0.2 and not (x == 1 and y == 1):
                    self.cells[x][y] = CellType.WALL
        
        # Гарантированный проход
        for x in range(1, self.width - 1):
            self.cells[x][self.height // 2] = CellType.EMPTY
    
    def is_valid_position(self, x, y):
        return 0 <= x < self.width and 0 <= y < self.height
    
    def is_wall(self, x, y):
        return self.cells[x][y] == CellType.WALL
    
    def get_cell(self, x, y):
        return self.cells[x][y]
    
    def set_cell(self, x, y, cell_type):
        self.cells[x][y] = cell_type
    
    def render(self, player, enemies):
        # Создаем временную карту с персонажами
        temp_map = [row.copy() for row in self.cells]
        
        # Добавляем игрока
        temp_map[player.x][player.y] = player.symbol
        
        # Добавляем врагов
        for enemy in enemies:
            if enemy.is_alive():
                temp_map[enemy.x][enemy.y] = enemy.symbol
        
        # Рендерим карту
        print('+' + '-' * self.width + '+')
        for y in range(self.height):
            print('|', end='')
            for x in range(self.width):
                print(temp_map[x][y].value, end='')
            print('|')
        print('+' + '-' * self.width + '+')

# Игровой интерфейс
class GameUI:
    @staticmethod
    def show_status(player, enemies_alive):
        print(f"HP: {player.hp}/{player.max_hp} | MP: {player.mp}/{player.max_mp} | Level: {player.level} | EXP: {player.exp}/{player.level * 100}")
        print(f"Armor: {player.armor} | Damage: {player.damage} | Enemies: {enemies_alive}")
    
    @staticmethod
    def show_actions():
        print("Actions: (W)Up, (S)Down, (A)Left, (D)Right, (F)Attack, (Q)Quit")
    
    @staticmethod
    def show_debug_info(player, enemies):
        print("=== DEBUG INFO ===")
        print(f"Player position: ({player.x}, {player.y})")
        for i, enemy in enumerate(enemies):
            status = "ALIVE" if enemy.is_alive() else "DEAD"
            print(f"Enemy {i}: ({enemy.x}, {enemy.y}) HP: {enemy.hp} {status}")
        print("=================")

# Основной класс игры
class Game:
    def __init__(self, map_width=10, map_height=10, auto_generate=True):
        self.map = GameMap(map_width, map_height, auto_generate)
        self.player = Player(1, 1)
        self.enemies = [Enemy(random.randint(3, map_width-2), random.randint(3, map_height-2)) for _ in range(3)]
        self.game_over = False
        self.debug_mode = False
    
    def handle_input(self, command):
        moved = False
        
        if command.lower() == 'w':
            moved = self.player.move(0, -1, self.map)
        elif command.lower() == 's':
            moved = self.player.move(0, 1, self.map)
        elif command.lower() == 'a':
            moved = self.player.move(-1, 0, self.map)
        elif command.lower() == 'd':
            moved = self.player.move(1, 0, self.map)
        elif command.lower() == 'f':
            self.attack_enemy()
        elif command.lower() == 'q':
            self.game_over = True
        elif command.lower() == '`':
            self.debug_mode = not self.debug_mode
        
        if moved:
            self.enemy_turn()
    
    def attack_enemy(self):
        for enemy in self.enemies:
            if not enemy.is_alive():
                continue
                
            # Проверяем соседние клетки
            if abs(self.player.x - enemy.x) <= 1 and abs(self.player.y - enemy.y) <= 1:
                damage = self.player.attack(enemy)
                print(f"You hit the enemy for {damage} damage!")
                if not enemy.is_alive():
                    exp_gain = 20
                    self.player.gain_exp(exp_gain)
                    print(f"Enemy defeated! Gained {exp_gain} EXP.")
                return
        
        print("No enemies nearby to attack!")
    
    def enemy_turn(self):
        for enemy in self.enemies:
            if not enemy.is_alive():
                continue
                
            # Простой ИИ: двигаться к игроку
            dx = 0
            dy = 0
            
            if enemy.x < self.player.x:
                dx = 1
            elif enemy.x > self.player.x:
                dx = -1
            
            if enemy.y < self.player.y:
                dy = 1
            elif enemy.y > self.player.y:
                dy = -1
            
            # Случайное движение (50% шанс)
            if random.random() < 0.5:
                if random.random() < 0.5:
                    enemy.move(dx, 0, self.map)
                else:
                    enemy.move(0, dy, self.map)
            
            # Атака, если рядом с игроком
            if abs(enemy.x - self.player.x) <= 1 and abs(enemy.y - self.player.y) <= 1:
                damage = enemy.attack(self.player)
                print(f"Enemy hits you for {damage} damage!")
                if not self.player.is_alive():
                    self.game_over = True
                    print("You have been defeated! Game Over.")
    
    def run(self):
        print("Welcome to Simple RPG Game!")
        print("Controls: W, A, S, D to move, F to attack, Q to quit, ` for debug")
        
        while not self.game_over:
            # Рендер карты
            self.map.render(self.player, self.enemies)
            
            # Показать статус
            alive_enemies = sum(1 for enemy in self.enemies if enemy.is_alive())
            GameUI.show_status(self.player, alive_enemies)
            
            # Показать действия
            GameUI.show_actions()
            
            # Debug информация
            if self.debug_mode:
                GameUI.show_debug_info(self.player, self.enemies)
            
            # Проверка победы
            if alive_enemies == 0:
                print("Congratulations! You defeated all enemies!")
                self.game_over = True
                break
            
            # Ввод команды
            command = input("> ")
            self.handle_input(command)
            
            print()  # Пустая строка для разделения ходов

# Выбор типа карты
def choose_map_type():
    print("Choose map type:")
    print("1. Standard (10x10)")
    print("2. Custom size")
    
    choice = input("> ")
    if choice == '1':
        return 10, 10, True
    elif choice == '2':
        width = int(input("Enter map width (5-20): "))
        height = int(input("Enter map height (5-20): "))
        return max(5, min(width, 20)), max(5, min(height, 20)), True
    else:
        print("Invalid choice, using standard map")
        return 10, 10, True

# Основная функция
def main():
    width, height, auto_generate = choose_map_type()
    game = Game(width, height, auto_generate)
    game.run()

if __name__ == "__main__":
    main()

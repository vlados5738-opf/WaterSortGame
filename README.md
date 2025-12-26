# WaterSortGame
import pygame
import sys
import random

# Инициализация Pygame
pygame.init()

# Константы
WIDTH, HEIGHT = 900, 600
FPS = 60
BACKGROUND_COLOR = (25, 25, 40)
BUTTON_COLOR = (70, 100, 150)
BUTTON_HOVER_COLOR = (90, 130, 190)
BUTTON_DISABLED_COLOR = (50, 70, 100)
TEXT_COLOR = (230, 230, 250)
GRID_COLOR = (60, 60, 80)

# Цвета жидкостей
LIQUID_COLORS = [
    (255, 50, 50),  # Красный
    (50, 255, 50),  # Зеленый
    (50, 50, 255),  # Синий
    (255, 255, 50),  # Желтый
    (255, 50, 255),  # Пурпурный
    (50, 255, 255),  # Голубой
    (255, 150, 50),  # Оранжевый
    (200, 50, 200),  # Фиолетовый
    (50, 200, 150),  # Бирюзовый
    (255, 100, 100),  # Розовый
]


class Flask:
    def __init__(self, x, y, width=80, height=200, max_layers=4):
        self.x = x
        self.y = y
        self.width = width
        self.height = height
        self.max_layers = max_layers
        self.layers = []
        self.selected = False
        self.rect = pygame.Rect(x - width // 2, y - height // 2, width, height)
        self.neck_rect = pygame.Rect(x - width // 3, y - height // 2 - 10, width * 2 // 3, 20)

    def add_layer(self, color):
        if len(self.layers) < self.max_layers:
            self.layers.append(color)
            return True
        return False

    def remove_layer(self):
        if self.layers:
            return self.layers.pop()
        return None

    def top_color(self):
        if self.layers:
            return self.layers[-1]
        return None

    def is_empty(self):
        return len(self.layers) == 0

    def is_full(self):
        return len(self.layers) == self.max_layers

    def is_completed(self):
        # Исправлено: пустая колба НЕ считается завершенной
        if self.is_empty():
            return False
        if len(self.layers) < self.max_layers:
            return False
        # Проверяем, что все слои одного цвета
        first_color = self.layers[0]
        for layer in self.layers:
            if layer != first_color:
                return False
        return True

    def is_usable(self):
        """Колба может быть использована (пустая или одного цвета)"""
        if self.is_empty():
            return True
        # Проверяем, что все слои одного цвета
        first_color = self.layers[0]
        for layer in self.layers:
            if layer != first_color:
                return False
        return True

    def draw(self, screen):
        # Колба
        pygame.draw.rect(screen, (200, 200, 220), self.rect, border_radius=10)
        pygame.draw.rect(screen, (180, 180, 200), self.neck_rect, border_radius=5)

        # Жидкость
        layer_height = self.height // self.max_layers
        for i, color in enumerate(self.layers):
            y_pos = self.rect.bottom - (i + 1) * layer_height
            liquid_rect = pygame.Rect(
                self.rect.x + 5,
                y_pos + 2,
                self.width - 10,
                layer_height - 4
            )
            pygame.draw.rect(screen, color, liquid_rect, border_radius=7)

            # Блики
            highlight = pygame.Rect(
                self.rect.x + 8,
                y_pos + 5,
                self.width // 3,
                5
            )
            pygame.draw.rect(screen, (255, 255, 255, 128), highlight, border_radius=2)

        # Выделение
        if self.selected:
            pygame.draw.rect(screen, (255, 255, 100), self.rect, 4, border_radius=10)
            pygame.draw.rect(screen, (255, 255, 100), self.neck_rect, 4, border_radius=5)

        # Ободок
        pygame.draw.rect(screen, (100, 100, 120), self.rect, 2, border_radius=10)
        pygame.draw.rect(screen, (100, 100, 120), self.neck_rect, 2, border_radius=5)


class Button:
    def __init__(self, x, y, width, height, text):
        self.rect = pygame.Rect(x, y, width, height)
        self.text = text
        self.hovered = False
        self.enabled = True

    def draw(self, screen, font):
        if not self.enabled:
            color = BUTTON_DISABLED_COLOR
            text_color = (150, 150, 170)
        else:
            color = BUTTON_HOVER_COLOR if self.hovered else BUTTON_COLOR
            text_color = TEXT_COLOR

        pygame.draw.rect(screen, color, self.rect, border_radius=10)
        pygame.draw.rect(screen, (40, 60, 100), self.rect, 3, border_radius=10)

        text_surf = font.render(self.text, True, text_color)
        text_rect = text_surf.get_rect(center=self.rect.center)
        screen.blit(text_surf, text_rect)

    def check_hover(self, pos):
        if self.enabled:
            self.hovered = self.rect.collidepoint(pos)
        else:
            self.hovered = False
        return self.hovered

    def is_clicked(self, pos, event_type):
        if event_type == pygame.MOUSEBUTTONDOWN and self.enabled:
            if self.rect.collidepoint(pos):
                return True
        return False

    def enable(self):
        self.enabled = True

    def disable(self):
        self.enabled = False
        self.hovered = False


class Game:
    def __init__(self):
        self.screen = pygame.display.set_mode((WIDTH, HEIGHT))
        pygame.display.set_caption("Сортировка жидкостей в колбах")
        self.clock = pygame.time.Clock()
        self.font = pygame.font.SysFont('arial', 24)
        self.big_font = pygame.font.SysFont('arial', 48)
        self.flasks = []
        self.selected_flask = None
        self.moves = 0
        self.level = 1
        self.game_won = False
        self.max_level = 5

        # Кнопки
        self.restart_btn = Button(WIDTH - 200, 20, 180, 40, "Перезапуск")
        self.next_btn = Button(WIDTH - 200, 70, 180, 40, "Следующий уровень")
        self.next_btn.disable()  # Изначально выключена

        self.init_level()

    def init_level(self):
        self.flasks = []
        self.selected_flask = None
        self.moves = 0
        self.game_won = False
        self.next_btn.disable()  # Выключаем кнопку "Следующий уровень"

        # Конфигурации уровней
        levels = [
            (4, 3),  # Уровень 1: 4 колбы, 3 цвета
            (5, 4),  # Уровень 2: 5 колб, 4 цвета
            (6, 5),  # Уровень 3: 6 колб, 5 цветов
            (7, 6),  # Уровень 4: 7 колб, 6 цветов
            (8, 7),  # Уровень 5: 8 колб, 7 цветов
        ]

        level_index = min(self.level - 1, len(levels) - 1)
        num_flasks, num_colors = levels[level_index]

        # Создаем цвета
        colors = LIQUID_COLORS[:num_colors]

        # Создаем жидкость - по 4 слоя каждого цвета
        all_liquids = []
        for color in colors:
            for _ in range(4):
                all_liquids.append(color)

        random.shuffle(all_liquids)

        # Распределяем по колбам
        flasks_data = []
        colors_per_flask = num_colors
        for i in range(colors_per_flask):
            flask_liquids = all_liquids[i * 4:(i + 1) * 4]
            flasks_data.append(flask_liquids)

        # Добавляем пустые колбы
        empty_flasks = num_flasks - colors_per_flask
        for _ in range(empty_flasks):
            flasks_data.append([])

        # Перемешиваем колбы
        random.shuffle(flasks_data)

        # Создаем и размещаем колбы
        flask_spacing = 100
        start_x = (WIDTH - (num_flasks * flask_spacing)) // 2 + flask_spacing // 2

        for i, liquids in enumerate(flasks_data):
            x = start_x + i * flask_spacing
            y = HEIGHT // 2
            flask = Flask(x, y)
            for color in liquids:
                flask.add_layer(color)
            self.flasks.append(flask)

    def check_victory(self):
        # Уровень пройден, если:
        # 1. Все непустые колбы заполнены (по 4 слоя)
        # 2. Все непустые колбы содержат жидкость одного цвета
        # 3. Пустые колбы игнорируются

        all_completed = True
        has_non_empty = False

        for flask in self.flasks:
            if not flask.is_empty():
                has_non_empty = True
                if not flask.is_completed():
                    all_completed = False
                    break

        # Уровень не может быть пройден, если нет ни одной непустой колбы
        if not has_non_empty:
            return False

        if all_completed and not self.game_won:
            self.game_won = True
            self.next_btn.enable()  # Включаем кнопку только когда уровень пройден
            return True

        return False

    def draw(self):
        self.screen.fill(BACKGROUND_COLOR)

        # Сетка
        for x in range(0, WIDTH, 40):
            pygame.draw.line(self.screen, GRID_COLOR, (x, 0), (x, HEIGHT), 1)
        for y in range(0, HEIGHT, 40):
            pygame.draw.line(self.screen, GRID_COLOR, (0, y), (WIDTH, y), 1)

        # Колбы
        for flask in self.flasks:
            flask.draw(self.screen)

        # Информация
        moves_text = self.font.render(f"Ходы: {self.moves}", True, TEXT_COLOR)
        level_text = self.font.render(f"Уровень: {self.level}/{self.max_level}", True, TEXT_COLOR)
        self.screen.blit(moves_text, (20, 20))
        self.screen.blit(level_text, (20, 50))

        # Инструкция
        instr = self.font.render("Кликните на колбу чтобы выбрать, затем на другую чтобы перелить",
                                 True, TEXT_COLOR)
        self.screen.blit(instr, (WIDTH // 2 - instr.get_width() // 2, HEIGHT - 40))

        # Кнопки
        mouse_pos = pygame.mouse.get_pos()
        self.restart_btn.check_hover(mouse_pos)
        self.next_btn.check_hover(mouse_pos)

        self.restart_btn.draw(self.screen, self.font)
        self.next_btn.draw(self.screen, self.font)

        # Проверяем победу каждый кадр
        victory = self.check_victory()

        if victory:
            # Полупрозрачный фон
            win_surface = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
            win_surface.fill((0, 0, 0, 180))
            self.screen.blit(win_surface, (0, 0))

            # Текст победы
            win_text = self.big_font.render("Уровень пройден!", True, (255, 255, 100))
            moves_text = self.font.render(f"Ходов сделано: {self.moves}", True, TEXT_COLOR)

            if self.level < self.max_level:
                next_text = self.font.render("Нажмите 'Следующий уровень' или N", True, (200, 200, 100))
                self.screen.blit(next_text, (WIDTH // 2 - next_text.get_width() // 2, HEIGHT // 2 + 50))
            else:
                final_text = self.font.render("Вы прошли все уровни! Начните заново.", True, (255, 200, 50))
                self.screen.blit(final_text, (WIDTH // 2 - final_text.get_width() // 2, HEIGHT // 2 + 50))

            self.screen.blit(win_text, (WIDTH // 2 - win_text.get_width() // 2, HEIGHT // 2 - 60))
            self.screen.blit(moves_text, (WIDTH // 2 - moves_text.get_width() // 2, HEIGHT // 2 + 10))

            # Отладочная информация (можно убрать)
            debug_text = self.font.render(
                f"Завершено колб: {sum(1 for f in self.flasks if f.is_completed())}/{len(self.flasks)}",
                True, (150, 255, 150))
            self.screen.blit(debug_text, (WIDTH // 2 - debug_text.get_width() // 2, HEIGHT // 2 + 100))

        pygame.display.flip()

    def handle_click(self, pos, event_type):
        # Проверка кнопок
        if self.restart_btn.is_clicked(pos, event_type):
            self.init_level()
            return

        if self.next_btn.is_clicked(pos, event_type):
            if self.game_won and self.level < self.max_level:
                self.level += 1
                self.init_level()
            return

        # Клики по колбам (только если игра не закончена)
        if not self.game_won:
            for i, flask in enumerate(self.flasks):
                if flask.rect.collidepoint(pos) or flask.neck_rect.collidepoint(pos):
                    if self.selected_flask is None:
                        if not flask.is_empty():
                            self.selected_flask = i
                            flask.selected = True
                    else:
                        if self.selected_flask == i:
                            # Отмена выбора
                            self.flasks[self.selected_flask].selected = False
                            self.selected_flask = None
                        else:
                            # Переливание
                            self.pour_liquid(self.selected_flask, i)
                    break

    def pour_liquid(self, from_idx, to_idx):
        from_flask = self.flasks[from_idx]
        to_flask = self.flasks[to_idx]

        if from_flask.is_empty() or to_flask.is_full():
            self.flasks[self.selected_flask].selected = False
            self.selected_flask = None
            return

        top_color = from_flask.top_color()

        # Проверка возможности переливания
        if to_flask.is_empty() or to_flask.top_color() == top_color:
            # Переливаем все слои одного цвета
            while (not from_flask.is_empty() and
                   from_flask.top_color() == top_color and
                   not to_flask.is_full()):
                color = from_flask.remove_layer()
                to_flask.add_layer(color)

            self.moves += 1

        # Сбрасываем выделение
        self.flasks[self.selected_flask].selected = False
        self.selected_flask = None

    def run(self):
        running = True
        while running:
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    running = False
                elif event.type == pygame.MOUSEBUTTONDOWN:
                    if event.button == 1:
                        self.handle_click(event.pos, event.type)
                elif event.type == pygame.KEYDOWN:
                    if event.key == pygame.K_r:
                        self.init_level()
                    elif event.key == pygame.K_n and self.game_won and self.level < self.max_level:
                        self.level += 1
                        self.init_level()
                    elif event.key == pygame.K_ESCAPE:
                        running = False

            self.draw()
            self.clock.tick(FPS)

        pygame.quit()
        sys.exit()


# Главная часть программы
if __name__ == "__main__":
    game = Game()
    game.run()

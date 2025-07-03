<template>
  <!-- Основной контейнер симуляции -->
  <div class="simulation-container">
    <div class="simulation-wrapper">
      <!-- Canvas для отрисовки игрового поля -->
      <canvas ref="canvas" width="1000" height="1000"></canvas>
      
      <!-- Панель статистики и управления -->
      <div class="statistics-panel">
        <h2>Статистика</h2>
        <div class="stats">
          <!-- Основные показатели симуляции -->
          <p>Умерли от голода: {{ stats.energyDeaths }}</p>
          <p>Съедено существ: {{ stats.predationDeaths }}</p>
          <p>Новых существ: {{ stats.newCreatures }}</p>
          <p>Всего существ: {{ creatures.length }}</p>
          <p>Еды: {{ foods.length }}</p>
          
          <!-- Средние характеристики популяции -->
          <h3>Характеристики</h3>
          <p>Средний размер: {{ stats.avgSize.toFixed(2) }}</p>
          <p>Средняя скорость: {{ stats.avgSpeed.toFixed(2) }}</p>
          <p>Средняя энергия: {{ stats.avgEnergy.toFixed(2) }}</p>
          <p>Среднее зрение: {{ stats.avgVision.toFixed(2) }}</p>
        </div>

        <!-- Блок управления симуляцией -->
        <div class="controls">
          <h3>Управление</h3>
          <!-- Кнопка паузы/продолжения -->
          <button @click="togglePause">
            {{ isPaused ? 'Продолжить' : 'Пауза' }}
          </button>
          
          <!-- Управление спавном объектов -->
          <div class="spawn-controls">
            <button @click="spawnRandomCreature">Добавить случайное существо</button>
            <button @click="spawnFood(10)">Добавить еду</button>
          </div>
          
          <!-- Создание кастомного существа -->
          <div class="custom-creature">
            <h3>Создать существо</h3>
            <div class="settings">
              <div>
                <label>Размер:</label>
                <input type="number" v-model.number="customCreature.size" min="10" max="100">
              </div>
              <div>
                <label>Скорость:</label>
                <input type="number" v-model.number="customCreature.speed" min="0.1" max="5" step="0.1">
              </div>
              <div>
                <label>Зрение:</label>
                <input type="number" v-model.number="customCreature.vision" min="50" max="300">
              </div>
              <div>
                <label>Цвет:</label>
                <input type="color" v-model="customCreature.color">
              </div>
            </div>
            <button @click="spawnCustomCreature">Добавить</button>
          </div>
          
          <!-- Настройки спавна еды -->
          <div class="settings">
            <div>
              <label>Интервал еды (сек):</label>
              <input type="number" v-model.number="foodSpawnInterval" min="1" @change="restartFoodSpawner">
            </div>
            <div>
              <label>Количество еды:</label>
              <input type="number" v-model.number="foodSpawnAmount" min="1">
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
/**
 * Класс Creature представляет существо в экосистеме.
 * Содержит логику поведения, движения и размножения существ.
 */
class Creature {
  constructor(x, y, size, speed, vision, color) {
    // Основные параметры существа
    this.x = x; // Позиция по X
    this.y = y; // Позиция по Y
    this.size = Math.min(100, size || 20); // Размер (10-100)
    this.speed = speed || 1.0; // Скорость движения (0.1-5)
    this.vision = vision || 100; // Радиус зрения (50-300)
    this.type = 'creature'; // Тип объекта
    this.energy = 100; // Уровень энергии (0-100)
    this.dx = Math.random() * 2 - 1; // Направление движения по X
    this.dy = Math.random() * 2 - 1; // Направление движения по Y
    this.color = color || this.getRandomColor(); // Цвет в HSL/HEX
    this.foodEaten = 0; // Количество съеденной еды
    this.isAggressive = false; // Режим охоты
    this.fleeing = false; // Режим бегства
    this.fleeTimer = 0; // Таймер бегства
    this.fleeDirection = { x: 0, y: 0 }; // Направление бегства
  }

  /**
   * Генерирует случайный цвет в HSL формате
   * @return {string} HSL цвет
   */
  getRandomColor() {
    const hue = Math.floor(Math.random() * 360);
    return `hsl(${hue}, 70%, 50%)`;
  }

  /**
   * Обновляет состояние существа
   * @param {number} deltaTime - Время с последнего обновления в секундах
   * @return {boolean} Возвращает true, если существо умерло
   */
  update(deltaTime) {
    // Уменьшение энергии пропорционально размеру
    this.energy -= 1.5 * deltaTime * (this.size / 20);

    // Обработка режима бегства
    if (this.fleeing) {
      this.fleeTimer -= deltaTime;
      if (this.fleeTimer <= 0) {
        this.fleeing = false;
      }
    }

    return this.energy <= 0; // Смерть от голода
  }

  /**
   * Вычисляет расстояние до цели
   * @param {Object} target - Цель с координатами x, y
   * @return {number} Расстояние до цели
   */
  distanceTo(target) {
    return Math.sqrt(Math.pow(this.x - target.x, 2) + Math.pow(this.y - target.y, 2));
  }

  /**
   * Движение к цели
   * @param {Object} target - Цель для движения
   * @param {number} deltaTime - Время с последнего обновления
   */
  moveTowards(target, deltaTime) {
    const dx = target.x - this.x;
    const dy = target.y - this.y;
    const dist = Math.sqrt(dx * dx + dy * dy);

    if (dist > 1) {
      // Нормализация направления
      this.dx = dx / dist;
      this.dy = dy / dist;
    }

    this.move(deltaTime);
  }

  /**
   * Бегство от хищника
   * @param {Creature} predator - Хищник, от которого убегаем
   * @param {number} deltaTime - Время с последнего обновления
   */
  fleeFrom(predator, deltaTime) {
    const dx = this.x - predator.x;
    const dy = this.y - predator.y;
    const dist = Math.sqrt(dx * dx + dy * dy);

    if (dist > 0) {
      this.fleeDirection.x = dx / dist;
      this.fleeDirection.y = dy / dist;
    }

    this.dx = this.fleeDirection.x;
    this.dy = this.fleeDirection.y;
    this.move(deltaTime, 1.5); // Ускоренное движение
  }

  /**
   * Случайное блуждание
   * @param {number} deltaTime - Время с последнего обновления
   */
  moveRandomly(deltaTime) {
    // Отскок от границ игрового поля
    if (this.x <= 100 || this.x >= 900 - this.size ||
        this.y <= 100 || this.y >= 900 - this.size) {
      this.bounceFromWall();
    }

    // Случайное изменение направления
    if (Math.random() < 0.01) {
      this.dx = Math.random() * 2 - 1;
      this.dy = Math.random() * 2 - 1;
    }

    this.move(deltaTime);
  }

  /**
   * Отскок от границы с случайным направлением
   */
  bounceFromWall() {
    const angle = Math.random() * 2 * Math.PI;
    this.dx = Math.cos(angle);
    this.dy = Math.sin(angle);
  }

  /**
   * Основное движение существа
   * @param {number} deltaTime - Время с последнего обновления
   * @param {number} speedMultiplier - Множитель скорости (по умолчанию 1)
   */
  move(deltaTime, speedMultiplier = 1) {
    // Обновление позиции
    this.x += this.dx * this.speed * 30 * deltaTime * speedMultiplier;
    this.y += this.dy * this.speed * 30 * deltaTime * speedMultiplier;

    // Проверка границ и отскок
    let bounced = false;
    if (this.x <= 100) {
      this.x = 100;
      bounced = true;
    }
    if (this.x >= 900 - this.size) {
      this.x = 900 - this.size;
      bounced = true;
    }
    if (this.y <= 100) {
      this.y = 100;
      bounced = true;
    }
    if (this.y >= 900 - this.size) {
      this.y = 900 - this.size;
      bounced = true;
    }
    if (bounced) {
      this.bounceFromWall();
    }
  }

  /**
   * Создает потомка с мутациями
   * @return {Creature} Новое существо-потомок
   */
  clone() {
    const mutationRate = 0.1; // Вероятность мутации
    const sizeMutation = (Math.random() * 2 - 1) * this.size * mutationRate;
    const speedMutation = (Math.random() * 2 - 1) * this.speed * mutationRate;
    const visionMutation = (Math.random() * 2 - 1) * this.vision * mutationRate;

    // Создание потомка с мутациями
    const clone = new Creature(
      this.x,
      this.y,
      Math.min(100, Math.max(10, this.size + sizeMutation)),
      Math.max(0.1, this.speed + speedMutation),
      Math.max(50, this.vision + visionMutation),
      this.color
    );
    clone.energy = this.energy * 0.8; // Потомок получает 80% энергии
    clone.dx = this.dx;
    clone.dy = this.dy;
    clone.isAggressive = this.isAggressive;
    return clone;
  }
}

/**
 * Класс Food представляет еду в экосистеме.
 * Содержит только базовые свойства без поведения.
 */
class Food {
  constructor(x, y) {
    this.x = x; // Позиция по X
    this.y = y; // Позиция по Y
    this.energy = 45; // Количество энергии при съедении
    this.color = '#2ecc71'; // Зеленый цвет
    this.size = 5; // Размер еды
  }
}

/**
 * Основной класс симуляции экосистемы.
 * Управляет всеми процессами: созданием объектов, обновлением, отрисовкой.
 */
class EcosystemSimulation {
  constructor(canvas) {
    // Инициализация основных свойств
    this.canvas = canvas; // Ссылка на canvas элемент
    this.ctx = canvas.getContext('2d'); // Контекст рисования
    this.creatures = []; // Массив существ
    this.foods = []; // Массив еды
    this.isPaused = false; // Состояние паузы
    this.lastTime = 0; // Время последнего кадра
    this.animationFrameId = null; // ID анимационного кадра
    this.foodSpawnTimer = null; // Таймер спавна еды
    this.foodSpawnInterval = 15; // Интервал спавна еды (сек)
    this.foodSpawnAmount = 10; // Количество еды за спавн
    
    // Настройки кастомного существа
    this.customCreature = {
      size: 20,
      speed: 1.0,
      vision: 100,
      color: '#ff0000'
    };
    
    // Статистика симуляции
    this.stats = {
      energyDeaths: 0, // Умерло от голода
      predationDeaths: 0, // Съедено хищниками
      newCreatures: 0, // Новых существ
      avgSize: 20, // Средний размер
      avgSpeed: 1.0, // Средняя скорость
      avgEnergy: 100, // Средняя энергия
      avgVision: 100 // Среднее зрение
    };
  }

  /**
   * Инициализация симуляции
   */
  initSimulation() {
    // Создание начальных существ и еды
    for (let i = 0; i < 15; i++) this.spawnRandomCreature();
    this.spawnFood(15);
    this.updateStats();
  }

  /**
   * Создание случайного существа
   */
  spawnRandomCreature() {
    const x = 100 + Math.random() * 800; // Случайная позиция X
    const y = 100 + Math.random() * 800; // Случайная позиция Y
    const size = Math.min(100, 15 + Math.random() * 15); // Случайный размер
    const speed = 0.5 + Math.random() * 1.5; // Случайная скорость
    const vision = 80 + Math.random() * 100; // Случайное зрение
    this.creatures.push(new Creature(x, y, size, speed, vision));
  }

  /**
   * Создание кастомного существа
   */
  spawnCustomCreature() {
    const x = 100 + Math.random() * 800;
    const y = 100 + Math.random() * 800;
    this.creatures.push(new Creature(
      x, 
      y, 
      Math.min(100, this.customCreature.size),
      this.customCreature.speed,
      this.customCreature.vision,
      this.customCreature.color
    ));
  }

  /**
   * Создание еды на поле
   * @param {number} amount - Количество создаваемой еды
   */
  spawnFood(amount) {
    const spawnMargin = 100; // Отступ от краев
    const gameArea = {
      x1: 100, y1: 100, // Левый верхний угол
      x2: 900, y2: 900  // Правый нижний угол
    };
    
    for (let i = 0; i < amount; i++) {
      let x, y;
      let validPosition = false;
      let attempts = 0;
      
      // Поиск валидной позиции (не слишком близко к краям)
      while (!validPosition && attempts < 100) {
        x = gameArea.x1 + Math.random() * (gameArea.x2 - gameArea.x1);
        y = gameArea.y1 + Math.random() * (gameArea.y2 - gameArea.y1);
        
        validPosition = 
          x > gameArea.x1 + spawnMargin && 
          x < gameArea.x2 - spawnMargin &&
          y > gameArea.y1 + spawnMargin && 
          y < gameArea.y2 - spawnMargin;
        
        attempts++;
      }
      
      if (validPosition) {
        this.foods.push(new Food(x, y));
      }
    }
  }

  /**
   * Запуск автоматического спавна еды
   */
  startFoodSpawner() {
    if (this.foodSpawnTimer) clearInterval(this.foodSpawnTimer);
    this.foodSpawnTimer = setInterval(() => {
      if (!this.isPaused) this.spawnFood(this.foodSpawnAmount);
    }, this.foodSpawnInterval * 1000);
  }

  /**
   * Перезапуск таймера спавна еды
   */
  restartFoodSpawner() {
    this.startFoodSpawner();
  }

  /**
   * Запуск игрового цикла
   */
  startAnimation() {
    this.lastTime = performance.now();
    this.animationFrameId = requestAnimationFrame(this.update.bind(this));
  }

  /**
   * Остановка симуляции
   */
  stopSimulation() {
    cancelAnimationFrame(this.animationFrameId);
    clearInterval(this.foodSpawnTimer);
  }

  /**
   * Переключение паузы
   */
  togglePause() {
    this.isPaused = !this.isPaused;
    if (!this.isPaused) this.startAnimation();
  }

  /**
   * Основной игровой цикл
   * @param {number} currentTime - Текущее время
   */
  update(currentTime) {
    if (this.isPaused) return;
    
    // Вычисление времени между кадрами
    const deltaTime = (currentTime - this.lastTime) / 1000;
    this.lastTime = currentTime;
    
    // Обновление состояния
    this.updateCreatures(deltaTime);
    this.checkCollisions();
    this.draw();
    
    // Запрос следующего кадра
    this.animationFrameId = requestAnimationFrame(this.update.bind(this));
  }

  /**
   * Обновление всех существ
   * @param {number} deltaTime - Время с последнего обновления
   */
  updateCreatures(deltaTime) {
    const creaturesToRemove = [];
    
    this.creatures.forEach((creature, index) => {
      // Обновление состояния существа
      const isDead = creature.update(deltaTime);
      
      if (isDead) {
        creaturesToRemove.push(index);
        this.stats.energyDeaths++;
        return;
      }
      
      // Обработка режима бегства
      if (creature.fleeing) {
        creature.move(deltaTime, 1.5);
        return;
      }
      
      // Поиск целей
      let target = null;
      let predator = null;
      let minDist = Infinity;
      
      // Поиск хищников
      this.creatures.forEach(other => {
        if (other === creature) return;
        
        const dist = creature.distanceTo(other);
        if (dist < creature.vision && other.size > creature.size * 1.2) {
          if (dist < minDist) {
            minDist = dist;
            predator = other;
          }
        }
      });
      
      // Активация бегства
      if (predator) {
        creature.fleeing = true;
        creature.fleeTimer = 2;
        creature.fleeFrom(predator, deltaTime);
        return;
      }
      
      // Поиск добычи (меньших существ)
      this.creatures.forEach(other => {
        if (other === creature || other.size >= creature.size * 0.8) return;
        
        const dist = creature.distanceTo(other);
        if (dist < creature.vision && dist < minDist) {
          minDist = dist;
          target = other;
          creature.isAggressive = true;
        }
      });
      
      // Если добычи нет, ищем еду
      if (!target) {
        creature.isAggressive = false;
        this.foods.forEach(food => {
          const dist = creature.distanceTo(food);
          if (dist < creature.vision && dist < minDist) {
            minDist = dist;
            target = food;
          }
        });
      }
      
      // Движение к цели или случайное блуждание
      if (target) {
        creature.moveTowards(target, deltaTime);
      } else {
        creature.moveRandomly(deltaTime);
      }
    });
    
    // Удаление умерших существ
    creaturesToRemove.sort((a, b) => b - a).forEach(index => {
      this.creatures.splice(index, 1);
    });
    
    this.updateStats();
  }

  /**
   * Проверка коллизий между объектами
   */
  checkCollisions() {
    // Коллизии существ с едой
    for (let i = 0; i < this.creatures.length; i++) {
      const creature = this.creatures[i];
      
      for (let j = 0; j < this.foods.length; j++) {
        const food = this.foods[j];
        
        if (this.distance(creature, food) < creature.size / 2) {
          // Существо съедает еду
          creature.size = Math.min(100, creature.size * 1.03);
          creature.speed *= 0.97;
          creature.vision *= 1.03;
          creature.energy = 100;
          creature.foodEaten++;
          this.foods.splice(j, 1);
          j--;
          
          // Размножение после 5 съеденных единиц еды
          if (creature.foodEaten >= 5) {
            creature.foodEaten = 0;
            const offspring = creature.clone();
            offspring.x += (Math.random() * 40 - 20);
            offspring.y += (Math.random() * 40 - 20);
            this.creatures.push(offspring);
            this.stats.newCreatures++;
          }
        }
      }
    }
    
    // Коллизии между существами
    for (let i = 0; i < this.creatures.length; i++) {
      for (let j = i + 1; j < this.creatures.length; j++) {
        const c1 = this.creatures[i];
        const c2 = this.creatures[j];
        
        if (this.distance(c1, c2) < (c1.size + c2.size) / 2) {
          // Одно существо значительно больше другого - съедание
          if (c1.size > c2.size * 1.2) {
            this.creatureEatsCreature(c1, c2);
            j--;
          } 
          else if (c2.size > c1.size * 1.2) {
            this.creatureEatsCreature(c2, c1);
            i--;
            break;
          }
          else {
            // Отталкивание при столкновении равных существ
            const dx = c2.x - c1.x;
            const dy = c2.y - c1.y;
            const dist = Math.sqrt(dx * dx + dy * dy);
            
            if (dist > 0) {
              const pushForce = 0.5;
              c1.x -= dx / dist * pushForce;
              c1.y -= dy / dist * pushForce;
              c2.x += dx / dist * pushForce;
              c2.y += dy / dist * pushForce;
            }
          }
        }
      }
    }
  }

  /**
   * Обработка съедания одного существа другим
   * @param {Creature} predator - Хищник
   * @param {Creature} prey - Добыча
   */
  creatureEatsCreature(predator, prey) {
    // Увеличение характеристик хищника
    predator.size = Math.min(100, predator.size * 1.03);
    predator.speed *= 0.97;
    predator.vision *= 1.03;
    predator.energy = 100;
    predator.foodEaten++;
    
    // Удаление съеденного существа
    const preyIndex = this.creatures.indexOf(prey);
    if (preyIndex !== -1) {
      this.creatures.splice(preyIndex, 1);
      this.stats.predationDeaths++;
    }
    
    // Размножение хищника
    if (predator.foodEaten >= 5) {
      predator.foodEaten = 0;
      const offspring = predator.clone();
      offspring.x += (Math.random() * 40 - 20);
      offspring.y += (Math.random() * 40 - 20);
      this.creatures.push(offspring);
      this.stats.newCreatures++;
    }
  }

  /**
   * Вычисление расстояния между двумя объектами
   * @param {Object} a - Первый объект с координатами x, y
   * @param {Object} b - Второй объект с координатами x, y
   * @return {number} Расстояние между объектами
   */
  distance(a, b) {
    return Math.sqrt(Math.pow(a.x - b.x, 2) + Math.pow(a.y - b.y, 2));
  }

  /**
   * Обновление статистики симуляции
   */
  updateStats() {
    if (this.creatures.length === 0) {
      // Обнуление статистики, если нет существ
      this.stats.avgSize = 0;
      this.stats.avgSpeed = 0;
      this.stats.avgEnergy = 0;
      this.stats.avgVision = 0;
      return;
    }
    
    // Вычисление средних значений
    const totalSize = this.creatures.reduce((sum, c) => sum + c.size, 0);
    const totalSpeed = this.creatures.reduce((sum, c) => sum + c.speed, 0);
    const totalEnergy = this.creatures.reduce((sum, c) => sum + c.energy, 0);
    const totalVision = this.creatures.reduce((sum, c) => sum + c.vision, 0);
    
    this.stats.avgSize = totalSize / this.creatures.length;
    this.stats.avgSpeed = totalSpeed / this.creatures.length;
    this.stats.avgEnergy = totalEnergy / this.creatures.length;
    this.stats.avgVision = totalVision / this.creatures.length;
  }

  /**
   * Отрисовка всех объектов на canvas
   */
  draw() {
    if (!this.ctx) return;
    
    // Очистка canvas
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    
    // Отрисовка еды
    this.foods.forEach(food => {
      this.ctx.fillStyle = food.color;
      this.ctx.beginPath();
      this.ctx.arc(food.x, food.y, food.size, 0, Math.PI * 2);
      this.ctx.fill();
    });
    
    // Отрисовка существ
    this.creatures.forEach(creature => {
      // Тело существа
      this.ctx.fillStyle = creature.color;
      this.ctx.beginPath();
      this.ctx.arc(creature.x, creature.y, creature.size / 2, 0, Math.PI * 2);
      this.ctx.fill();
      
      // Глаз (белый или красный в агрессивном режиме)
      this.ctx.fillStyle = creature.isAggressive ? '#ff0000' : '#ffffff';
      this.ctx.beginPath();
      this.ctx.arc(
        creature.x + creature.dx * creature.size * 0.3,
        creature.y + creature.dy * creature.size * 0.3,
        creature.size * 0.15,
        0,
        Math.PI * 2
      );
      this.ctx.fill();
      
      // Обводка для крупных существ
      if (creature.size > 20) {
        this.ctx.strokeStyle = creature.isAggressive ? '#ff0000' : '#00ff00';
        this.ctx.lineWidth = 2;
        this.ctx.beginPath();
        this.ctx.arc(creature.x, creature.y, creature.size / 2 + 2, 0, Math.PI * 2);
        this.ctx.stroke();
      }
      
      // Обводка для убегающих существ
      if (creature.fleeing) {
        this.ctx.strokeStyle = '#ffff00';
        this.ctx.lineWidth = 2;
        this.ctx.beginPath();
        this.ctx.arc(creature.x, creature.y, creature.size / 2 + 4, 0, Math.PI * 2);
        this.ctx.stroke();
      }
    });
  }
}

/**
 * Vue-компонент для отображения и управления симуляцией
 */
export default {
  data() {
    return {
      simulation: null, // Экземпляр симуляции
      customCreature: { // Параметры кастомного существа
        size: 20,
        speed: 1.0,
        vision: 100,
        color: '#ff0000'
      },
      foodSpawnInterval: 15, // Интервал спавна еды
      foodSpawnAmount: 10 // Количество еды за спавн
    };
  },
  
  mounted() {
    // Инициализация canvas и симуляции
    this.canvas = this.$refs.canvas;
    this.canvas.width = 1000;
    this.canvas.height = 1000;
    
    this.simulation = new EcosystemSimulation(this.canvas);
    this.simulation.initSimulation();
    this.simulation.startAnimation();
    this.simulation.startFoodSpawner();
    window.addEventListener('resize', this.handleResize);
  },
  
  beforeUnmount() {
    // Очистка при уничтожении компонента
    this.simulation.stopSimulation();
    window.removeEventListener('resize', this.handleResize);
  },
  
  methods: {
    /**
     * Обработчик изменения размера окна
     */
    handleResize() {
      const container = this.$el.querySelector('.simulation-wrapper');
      const availableWidth = container.clientWidth - 320;
      const newSize = Math.min(1000, availableWidth);
      
      this.canvas.style.width = `${newSize}px`;
      this.canvas.style.height = `${newSize}px`;
    },
    
    /**
     * Переключение паузы симуляции
     */
    togglePause() {
      this.simulation.togglePause();
    },
    
    /**
     * Создание случайного существа
     */
    spawnRandomCreature() {
      this.simulation.spawnRandomCreature();
    },
    
    /**
     * Создание кастомного существа
     */
    spawnCustomCreature() {
      this.simulation.customCreature = {...this.customCreature};
      this.simulation.spawnCustomCreature();
    },
    
    /**
     * Создание еды
     * @param {number} amount - Количество еды
     */
    spawnFood(amount) {
      this.simulation.spawnFood(amount);
    },
    
    /**
     * Перезапуск спавнера еды
     */
    restartFoodSpawner() {
      this.simulation.foodSpawnInterval = this.foodSpawnInterval;
      this.simulation.foodSpawnAmount = this.foodSpawnAmount;
      this.simulation.restartFoodSpawner();
    }
  },
  
  computed: {
    /**
     * Статистика симуляции
     */
    stats() {
      return this.simulation ? this.simulation.stats : {
        energyDeaths: 0,
        predationDeaths: 0,
        newCreatures: 0,
        avgSize: 20,
        avgSpeed: 1.0,
        avgEnergy: 100,
        avgVision: 100
      };
    },
    
    /**
     * Массив существ
     */
    creatures() {
      return this.simulation ? this.simulation.creatures : [];
    },
    
    /**
     * Массив еды
     */
    foods() {
      return this.simulation ? this.simulation.foods : [];
    },
    
    /**
     * Состояние паузы
     */
    isPaused() {
      return this.simulation ? this.simulation.isPaused : false;
    }
  }
};
</script>

<style>
/* Основные стили контейнера */
.simulation-container {
  display: flex;
  justify-content: center;
  padding: 10px;
}

/* Обертка для canvas и панели управления */
.simulation-wrapper {
  display: flex;
  gap: 15px;
  max-width: 1320px;
  width: 100%;
}

/* Стили для игрового поля */
canvas {
  background-color: #000000;
  border-radius: 6px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  width: 1000px;
  height: 1000px;
  image-rendering: crisp-edges;
}

/* Стили панели статистики */
.statistics-panel {
  width: 300px;
  background-color: #fff;
  border-radius: 6px;
  padding: 15px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  overflow-y: auto;
}

.stats {
  margin-bottom: 15px;
  font-size: 0.9em;
}

.stats p {
  margin: 5px 0;
}

/* Стили блока управления */
.controls {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

/* Стили кнопок */
button {
  padding: 8px 12px;
  font-size: 0.85em;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
}

button:hover {
  background-color: #2980b9;
}

/* Стили для управления спавном */
.spawn-controls {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

/* Стили для создания кастомных существ */
.custom-creature {
  margin: 10px 0;
  padding: 10px;
  background-color: #f8f9fa;
  border-radius: 4px;
}

.custom-creature h3 {
  margin: 5px 0 10px 0;
  font-size: 1em;
}

.settings {
  padding: 10px;
  gap: 8px;
  background-color: #f8f9fa;
  border-radius: 4px;
}

.settings div {
  margin-bottom: 5px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

/* Стили для полей ввода */
input {
  padding: 6px;
  width: 50px;
  font-size: 0.85em;
  border: 1px solid #ddd;
  border-radius: 4px;
}

input[type="color"] {
  width: 30px;
  height: 25px;
  padding: 2px;
}

/* Заголовки */
h2 {
  font-size: 1.3em;
  margin-bottom: 10px;
  color: #2c3e50;
}

h3 {
  font-size: 1.1em;
  margin: 5px 0;
  color: #2c3e50;
}

/* Адаптивные стили */
@media (max-width: 1400px) {
  .simulation-wrapper {
    flex-direction: column;
    align-items: center;
  }
  
  .statistics-panel {
    width: 1000px;
    max-height: 300px;
  }
}

@media (max-width: 1050px) {
  canvas {
    width: 100%;
    height: auto;
    max-width: 100vw;
    max-height: 100vw;
  }
  
  .statistics-panel {
    width: 100%;
  }
}
</style>
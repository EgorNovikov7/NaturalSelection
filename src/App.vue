<template>
  <div class="simulation-container">
    <div class="simulation-wrapper">
      <canvas ref="canvas" width="700" height="700"></canvas>
      <div class="statistics-panel">
        <h2>Статистика</h2>
        <div class="stats">
          <p>Умерли от голода: {{ stats.energyDeaths }}</p>
          <p>Съедено существ: {{ stats.predationDeaths }}</p>
          <p>Новых существ: {{ stats.newCreatures }}</p>
          <p>Всего существ: {{ creatures.length }}</p>
          <p>Еды: {{ foods.length }}</p>
          
          <h3>Характеристики</h3>
          <p>Средний размер: {{ stats.avgSize.toFixed(2) }}</p>
          <p>Средняя скорость: {{ stats.avgSpeed.toFixed(2) }}</p>
          <p>Средняя энергия: {{ stats.avgEnergy.toFixed(2) }}</p>
          <p>Среднее зрение: {{ stats.avgVision.toFixed(2) }}</p>
        </div>

        <div class="controls">
          <h3>Управление</h3>
          <button @click="togglePause">
            {{ isPaused ? 'Продолжить' : 'Пауза' }}
          </button>
          
          <div class="spawn-controls">
            <button @click="spawnRandomCreature">Добавить случайное существо</button>
            <button @click="spawnFood(10)">Добавить еду</button>
          </div>
          
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
class Creature {
  constructor(x, y, size, speed, vision, color) {
    this.x = x;
    this.y = y;
    this.size = Math.min(100, size || 20); // Добавлено ограничение
    this.speed = speed || 1.0;
    this.vision = vision || 100;
    this.type = 'creature';
    this.energy = 100;
    this.dx = Math.random() * 2 - 1;
    this.dy = Math.random() * 2 - 1;
    this.color = color || this.getRandomColor();
    this.foodEaten = 0;
    this.isAggressive = false;
    this.fleeing = false;
    this.fleeTimer = 0;
    this.fleeDirection = { x: 0, y: 0 };
  }

  getRandomColor() {
    const hue = Math.floor(Math.random() * 360);
    return `hsl(${hue}, 70%, 50%)`;
  }

  update(deltaTime) {
    this.energy -= 1.5 * deltaTime * (this.size / 20);

    if (this.fleeing) {
      this.fleeTimer -= deltaTime;
      if (this.fleeTimer <= 0) {
        this.fleeing = false;
      }
    }

    return this.energy <= 0;
  }

  distanceTo(target) {
    return Math.sqrt(Math.pow(this.x - target.x, 2) + Math.pow(this.y - target.y, 2));
  }

  moveTowards(target, deltaTime) {
    const dx = target.x - this.x;
    const dy = target.y - this.y;
    const dist = Math.sqrt(dx * dx + dy * dy);

    if (dist > 1) {
      this.dx = dx / dist;
      this.dy = dy / dist;
    }

    this.move(deltaTime);
  }

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
    this.move(deltaTime, 1.5);
  }

  moveRandomly(deltaTime) {
    if (this.x <= 50 || this.x >= 650 - this.size ||
        this.y <= 50 || this.y >= 650 - this.size) {
      this.bounceFromWall();
    }

    if (Math.random() < 0.01) {
      this.dx = Math.random() * 2 - 1;
      this.dy = Math.random() * 2 - 1;
    }

    this.move(deltaTime);
  }

  bounceFromWall() {
    const angle = Math.random() * 2 * Math.PI;
    this.dx = Math.cos(angle);
    this.dy = Math.sin(angle);
  }

  move(deltaTime, speedMultiplier = 1) {
    this.x += this.dx * this.speed * 30 * deltaTime * speedMultiplier;
    this.y += this.dy * this.speed * 30 * deltaTime * speedMultiplier;

    let bounced = false;
    if (this.x <= 50) {
      this.x = 50;
      bounced = true;
    }
    if (this.x >= 650 - this.size) {
      this.x = 650 - this.size;
      bounced = true;
    }
    if (this.y <= 50) {
      this.y = 50;
      bounced = true;
    }
    if (this.y >= 650 - this.size) {
      this.y = 650 - this.size;
      bounced = true;
    }
    if (bounced) {
      this.bounceFromWall();
    }
  }

  clone() {
    const mutationRate = 0.1;
    const sizeMutation = (Math.random() * 2 - 1) * this.size * mutationRate;
    const speedMutation = (Math.random() * 2 - 1) * this.speed * mutationRate;
    const visionMutation = (Math.random() * 2 - 1) * this.vision * mutationRate;

    const clone = new Creature(
      this.x,
      this.y,
      Math.min(100, Math.max(10, this.size + sizeMutation)), // Добавлено ограничение
      Math.max(0.1, this.speed + speedMutation),
      Math.max(50, this.vision + visionMutation),
      this.color
    );
    clone.energy = this.energy * 0.8;
    clone.dx = this.dx;
    clone.dy = this.dy;
    clone.isAggressive = this.isAggressive;
    return clone;
  }
}

class Food {
  constructor(x, y) {
    this.x = x;
    this.y = y;
    this.energy = 45;
    this.color = '#2ecc71';
    this.size = 5;
  }
}

export default {
  data() {
    return {
      canvas: null,
      ctx: null,
      creatures: [],
      foods: [],
      isPaused: false,
      lastTime: 0,
      animationFrameId: null,
      foodSpawnTimer: null,
      foodSpawnInterval: 15,
      foodSpawnAmount: 10,
      customCreature: {
        size: 20,
        speed: 1.0,
        vision: 100,
        color: '#ff0000'
      },
      stats: {
        energyDeaths: 0,
        predationDeaths: 0,
        newCreatures: 0,
        avgSize: 20,
        avgSpeed: 1.0,
        avgEnergy: 100,
        avgVision: 100
      }
    };
  },
  mounted() {
    this.canvas = this.$refs.canvas;
    this.canvas.width = 700;
    this.canvas.height = 700;
    this.ctx = this.canvas.getContext('2d');
    this.initSimulation();
    this.startAnimation();
    this.startFoodSpawner();
    window.addEventListener('resize', this.handleResize);
  },
  beforeUnmount() {
    this.stopSimulation();
    window.removeEventListener('resize', this.handleResize);
  },
  methods: {
    handleResize() {
      const container = this.$el.querySelector('.simulation-wrapper');
      const availableWidth = container.clientWidth - 320;
      const newSize = Math.min(700, availableWidth);
      
      this.canvas.style.width = `${newSize}px`;
      this.canvas.style.height = `${newSize}px`;
    },
    
    initSimulation() {
      for (let i = 0; i < 15; i++) this.spawnRandomCreature();
      this.spawnFood(15);
      this.updateStats();
    },
    
    spawnRandomCreature() {
      const x = 50 + Math.random() * 600;
      const y = 50 + Math.random() * 600;
      const size = Math.min(100, 15 + Math.random() * 15); // Добавлено ограничение
      const speed = 0.5 + Math.random() * 1.5;
      const vision = 80 + Math.random() * 100;
      this.creatures.push(new Creature(x, y, size, speed, vision));
    },
    
    spawnCustomCreature() {
      const x = 50 + Math.random() * 600;
      const y = 50 + Math.random() * 600;
      this.creatures.push(new Creature(
        x, 
        y, 
        Math.min(100, this.customCreature.size), // Добавлено ограничение
        this.customCreature.speed,
        this.customCreature.vision,
        this.customCreature.color
      ));
    },
    
    getValidFoodPosition() {
      const margin = 60;
      const maxAttempts = 100;
      let x, y, valid = false;
      
      for (let i = 0; i < maxAttempts; i++) {
        x = margin + Math.random() * (this.canvas.width - 2 * margin);
        y = margin + Math.random() * (this.canvas.height - 2 * margin);
        
        const tooCloseToCreature = this.creatures.some(creature => {
          return Math.sqrt(Math.pow(x - creature.x, 2) + Math.pow(y - creature.y, 2)) < creature.size;
        });
        
        if (!tooCloseToCreature) {
          valid = true;
          break;
        }
      }
      
      return valid ? { x, y } : null;
    },
    
    spawnFood(amount) {
      const spawnMargin = 30;
      const gameArea = {
        x1: 50, y1: 50,
        x2: 650, y2: 650
      };
      
      for (let i = 0; i < amount; i++) {
        let x, y;
        let validPosition = false;
        let attempts = 0;
        
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
    },
    
    startFoodSpawner() {
      if (this.foodSpawnTimer) clearInterval(this.foodSpawnTimer);
      this.foodSpawnTimer = setInterval(() => {
        if (!this.isPaused) this.spawnFood(this.foodSpawnAmount);
      }, this.foodSpawnInterval * 1000);
    },
    
    restartFoodSpawner() {
      this.startFoodSpawner();
    },
    
    startAnimation() {
      this.lastTime = performance.now();
      this.animationFrameId = requestAnimationFrame(this.update);
    },
    
    stopSimulation() {
      cancelAnimationFrame(this.animationFrameId);
      clearInterval(this.foodSpawnTimer);
    },
    
    togglePause() {
      this.isPaused = !this.isPaused;
      if (!this.isPaused) this.startAnimation();
    },
    
    update(currentTime) {
      if (this.isPaused) return;
      
      const deltaTime = (currentTime - this.lastTime) / 1000;
      this.lastTime = currentTime;
      
      this.updateCreatures(deltaTime);
      this.checkCollisions();
      this.draw();
      
      this.animationFrameId = requestAnimationFrame(this.update);
    },
    
    updateCreatures(deltaTime) {
      const creaturesToRemove = [];
      
      this.creatures.forEach((creature, index) => {
        const isDead = creature.update(deltaTime);
        
        if (isDead) {
          creaturesToRemove.push(index);
          this.stats.energyDeaths++;
          this.foods.push(new Food(creature.x, creature.y));
          return;
        }
        
        if (creature.fleeing) {
          creature.move(deltaTime, 1.5);
          return;
        }
        
        let target = null;
        let predator = null;
        let minDist = Infinity;
        
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
        
        if (predator) {
          creature.fleeing = true;
          creature.fleeTimer = 2;
          creature.fleeFrom(predator, deltaTime);
          return;
        }
        
        this.creatures.forEach(other => {
          if (other === creature || other.size >= creature.size * 0.8) return;
          
          const dist = creature.distanceTo(other);
          if (dist < creature.vision && dist < minDist) {
            minDist = dist;
            target = other;
            creature.isAggressive = true;
          }
        });
        
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
        
        if (target) {
          creature.moveTowards(target, deltaTime);
        } else {
          creature.moveRandomly(deltaTime);
        }
      });
      
      creaturesToRemove.sort((a, b) => b - a).forEach(index => {
        this.creatures.splice(index, 1);
      });
      
      this.updateStats();
    },
    
    checkCollisions() {
      for (let i = 0; i < this.creatures.length; i++) {
        const creature = this.creatures[i];
        
        for (let j = 0; j < this.foods.length; j++) {
          const food = this.foods[j];
          
          if (this.distance(creature, food) < creature.size / 2) {
            creature.size = Math.min(100, creature.size * 1.03); // Добавлено ограничение
            creature.speed *= 0.97;
            creature.vision *= 1.03;
            creature.energy = 100;
            creature.foodEaten++;
            this.foods.splice(j, 1);
            j--;
            
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
      
      for (let i = 0; i < this.creatures.length; i++) {
        for (let j = i + 1; j < this.creatures.length; j++) {
          const c1 = this.creatures[i];
          const c2 = this.creatures[j];
          
          if (this.distance(c1, c2) < (c1.size + c2.size) / 2) {
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
    },
    
    creatureEatsCreature(predator, prey) {
      predator.size = Math.min(100, predator.size * 1.03); // Добавлено ограничение
      predator.speed *= 0.97;
      predator.vision *= 1.03;
      predator.energy = 100;
      predator.foodEaten++;
      
      const preyIndex = this.creatures.indexOf(prey);
      if (preyIndex !== -1) {
        this.creatures.splice(preyIndex, 1);
        this.stats.predationDeaths++;
      }
      
      if (predator.foodEaten >= 5) {
        predator.foodEaten = 0;
        const offspring = predator.clone();
        offspring.x += (Math.random() * 40 - 20);
        offspring.y += (Math.random() * 40 - 20);
        this.creatures.push(offspring);
        this.stats.newCreatures++;
      }
    },
    
    distance(a, b) {
      return Math.sqrt(Math.pow(a.x - b.x, 2) + Math.pow(a.y - b.y, 2));
    },
    
    updateStats() {
      if (this.creatures.length === 0) {
        this.stats.avgSize = 0;
        this.stats.avgSpeed = 0;
        this.stats.avgEnergy = 0;
        this.stats.avgVision = 0;
        return;
      }
      
      const totalSize = this.creatures.reduce((sum, c) => sum + c.size, 0);
      const totalSpeed = this.creatures.reduce((sum, c) => sum + c.speed, 0);
      const totalEnergy = this.creatures.reduce((sum, c) => sum + c.energy, 0);
      const totalVision = this.creatures.reduce((sum, c) => sum + c.vision, 0);
      
      this.stats.avgSize = totalSize / this.creatures.length;
      this.stats.avgSpeed = totalSpeed / this.creatures.length;
      this.stats.avgEnergy = totalEnergy / this.creatures.length;
      this.stats.avgVision = totalVision / this.creatures.length;
    },
    
    draw() {
      if (!this.ctx) return;
      
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      
      this.ctx.strokeStyle = '#ffffff';
      this.ctx.lineWidth = 2;
      this.ctx.strokeRect(50, 50, 600, 600);
      
      this.foods.forEach(food => {
        this.ctx.fillStyle = food.color;
        this.ctx.beginPath();
        this.ctx.arc(food.x, food.y, food.size, 0, Math.PI * 2);
        this.ctx.fill();
      });
      
      this.creatures.forEach(creature => {
        this.ctx.fillStyle = creature.color;
        this.ctx.beginPath();
        this.ctx.arc(creature.x, creature.y, creature.size / 2, 0, Math.PI * 2);
        this.ctx.fill();
        
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
        
        if (creature.size > 20) {
          this.ctx.strokeStyle = creature.isAggressive ? '#ff0000' : '#00ff00';
          this.ctx.lineWidth = 2;
          this.ctx.beginPath();
          this.ctx.arc(creature.x, creature.y, creature.size / 2 + 2, 0, Math.PI * 2);
          this.ctx.stroke();
        }
        
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
};
</script>




<style>
.simulation-container {
  display: flex;
  justify-content: center;
  padding: 10px;
}

.simulation-wrapper {
  display: flex;
  gap: 15px;
  max-width: 1020px;
  width: 100%;
}

canvas {
  background-color: #000000;
  border-radius: 6px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  width: 700px;
  height: 700px;
  image-rendering: crisp-edges;
}

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

.controls {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

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

.spawn-controls {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

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

@media (max-width: 1000px) {
  .simulation-wrapper {
    flex-direction: column;
    align-items: center;
  }
  
  .statistics-panel {
    width: 700px;
    max-height: 300px;
  }
}

@media (max-width: 750px) {
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
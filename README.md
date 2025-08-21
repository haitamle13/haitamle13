"""
TaskMaster Pro - Système Avancé de Gestion de Tâches
Développé par : Haitam Faddi
Technologies : Python, SQLite, Logging, Typing
"""

import sqlite3
import json
import logging
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
from enum import Enum
from dataclasses import dataclass

# Configuration du logging professionnel
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('taskmaster.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger('TaskMaster')

class Priority(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    URGENT = 4

class Status(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    CANCELLED = "cancelled"

@dataclass
class Task:
    id: Optional[int]
    title: str
    description: str
    priority: Priority
    status: Status
    due_date: datetime
    created_at: datetime
    updated_at: datetime
    tags: List[str]
    assigned_to: Optional[str]
    
    def to_dict(self) -> Dict:
        return {
            'id': self.id,
            'title': self.title,
            'description': self.description,
            'priority': self.priority.name,
            'status': self.status.value,
            'due_date': self.due_date.isoformat(),
            'created_at': self.created_at.isoformat(),
            'updated_at': self.updated_at.isoformat(),
            'tags': self.tags,
            'assigned_to': self.assigned_to
        }

class TaskManager:
    """Système professionnel de gestion de tâches avec persistance des données"""
    
    def __init__(self, db_path: str = 'taskmaster.db'):
        self.db_path = db_path
        self._init_database()
        logger.info("TaskManager initialisé avec succès")
    
    def _init_database(self) -> None:
        """Initialisation de la base de données avec schéma optimisé"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            
            # Table des tâches
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS tasks (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    title TEXT NOT NULL CHECK(length(title) > 0),
                    description TEXT,
                    priority INTEGER NOT NULL CHECK(priority BETWEEN 1 AND 4),
                    status TEXT NOT NULL,
                    due_date TIMESTAMP NOT NULL,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    tags TEXT DEFAULT '[]',
                    assigned_to TEXT,
                    completed_at TIMESTAMP NULL
                )
            ''')
            
            # Index pour les performances
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_status ON tasks(status)')
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_priority ON tasks(priority)')
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_due_date ON tasks(due_date)')
            
            conn.commit()
    
    def create_task(self, title: str, description: str, priority: Priority, 
                   due_date: datetime, tags: List[str] = None, assigned_to: str = None) -> Task:
        """Crée une nouvelle tâche avec validation des données"""
        if not title or len(title.strip()) == 0:
            raise ValueError("Le titre de la tâche ne peut pas être vide")
        
        if due_date < datetime.now():
            raise ValueError("La date d'échéance ne peut pas être dans le passé")
        
        tags = tags or []
        current_time = datetime.now()
        
        task = Task(
            id=None,
            title=title.strip(),
            description=description.strip(),
            priority=priority,
            status=Status.PENDING,
            due_date=due_date,
            created_at=current_time,
            updated_at=current_time,
            tags=tags,
            assigned_to=assigned_to
        )
        
        try:
            with sqlite3.connect(self.db_path) as conn:
                cursor = conn.cursor()
                cursor.execute('''
                    INSERT INTO tasks (title, description, priority, status, due_date, tags, assigned_to)
                    VALUES (?, ?, ?, ?, ?, ?, ?)
                ''', (
                    task.title, 
                    task.description, 
                    task.priority.value, 
                    task.status.value,
                    task.due_date.isoformat(),
                    json.dumps(task.tags),
                    task.assigned_to
                ))
                task.id = cursor.lastrowid
                conn.commit()
            
            logger.info(f"Tâche créée: '{task.title}' (ID: {task.id})")
            return task
            
        except sqlite3.Error as e:
            logger.error(f"Erreur lors de la création de la tâche: {e}")
            raise
    
    def get_task(self, task_id: int) -> Optional[Task]:
        """Récupère une tâche par son ID"""
        try:
            with sqlite3.connect(self.db_path) as conn:
                cursor = conn.cursor()
                cursor.execute('SELECT * FROM tasks WHERE id = ?', (task_id,))
                row = cursor.fetchone()
                
                return self._row_to_task(row) if row else None
                
        except sqlite3.Error as e:
            logger.error(f"Erreur lors de la récupération de la tâche {task_id}: {e}")
            return None
    
    def update_task_status(self, task_id: int, status: Status) -> Optional[Task]:
        """Met à jour le statut d'une tâche"""
        try:
            with sqlite3.connect(self.db_path) as conn:
                cursor = conn.cursor()
                cursor.execute('''
                    UPDATE tasks 
                    SET status = ?, updated_at = ?, completed_at = ?
                    WHERE id = ?
                ''', (
                    status.value,
                    datetime.now().isoformat(),
                    datetime.now().isoformat() if status == Status.COMPLETED else None,
                    task_id
                ))
                
                if cursor.rowcount == 0:
                    return None
                
                conn.commit()
                logger.info(f"Statut de la tâche {task_id} mis à jour: {status.value}")
                return self.get_task(task_id)
                
        except sqlite3.Error as e:
            logger.error(f"Erreur lors de la mise à jour de la tâche {task_id}: {e}")
            return None
    
    def delete_task(self, task_id: int) -> bool:
        """Supprime une tâche"""
        try:
            with sqlite3.connect(self.db_path) as conn:
                cursor = conn.cursor()
                cursor.execute('DELETE FROM tasks WHERE id = ?', (task_id,))
                conn.commit()
                
                success = cursor.rowcount > 0
                if success:
                    logger.info(f"Tâche {task_id} supprimée avec succès")
                else:
                    logger.warning(f"Tâche {task_id} non trouvée pour suppression")
                
                return success
                
        except sqlite3.Error as e:
            logger.error(f"Erreur lors de la suppression de la tâche {task_id}: {e}")
            return False
    
    def list_tasks(self, filters: Dict = None) -> List[Task]:
        """Liste les tâches avec filtres optionnels"""
        filters = filters or {}
        query = 'SELECT * FROM tasks WHERE 1=1'
        params = []
        
        if 'status' in filters:
            query += ' AND status = ?'
            params.append(filters['status'].value)
        
        if 'priority' in filters:
            query += ' AND priority = ?'
            params.append(filters['priority'].value)
        
        if 'assigned_to' in filters:
            query += ' AND assigned_to = ?'
            params.append(filters['assigned_to'])
        
        query += ' ORDER BY priority DESC, due_date ASC'
        
        try:
            with sqlite3.connect(self.db_path) as conn:
                cursor = conn.cursor()
                cursor.execute(query, params)
                rows = cursor.fetchall()
                
                return [self._row_to_task(row) for row in rows]
                
        except sqlite3.Error as e:
            logger.error(f"Erreur lors de la récupération des tâches: {e}")
            return []
    
    def get_tasks_due_soon(self, days: int = 7) -> List[Task]:
        """Récupère les tâches dues dans les prochains jours"""
        due_date = (datetime.now() + timedelta(days=days)).isoformat()
        
        try:
            with sqlite3.connect(self.db_path) as conn:
                cursor = conn.cursor()
                cursor.execute('''
                    SELECT * FROM tasks 
                    WHERE due_date <= ? AND status NOT IN (?, ?)
                    ORDER BY due_date ASC
                ''', (due_date, Status.COMPLETED.value, Status.CANCELLED.value))
                
                rows = cursor.fetchall()
                return [self._row_to_task(row) for row in rows]
                
        except sqlite3.Error as e:
            logger.error(f"Erreur lors de la récupération des tâches dues: {e}")
            return []
    
    def _row_to_task(self, row: Tuple) -> Task:
        """Convertit une ligne de base de données en objet Task"""
        return Task(
            id=row[0],
            title=row[1],
            description=row[2],
            priority=Priority(row[3]),
            status=Status(row[4]),
            due_date=datetime.fromisoformat(row[5]),
            created_at=datetime.fromisoformat(row[6]),
            updated_at=datetime.fromisoformat(row[7]),
            tags=json.loads(row[8]),
            assigned_to=row[9]
        )

# Exemple d'utilisation professionnelle
def demo_systeme_taches():
    """Démonstration du système de gestion de tâches"""
    print("🚀 TaskMaster Pro - Démonstration")
    print("=" * 50)
    
    # Initialisation
    tm = TaskManager()
    
    # Création de tâches
    try:
        # Tâche urgente
        tache_urgente = tm.create_task(
            title="Présentation Client Final",
            description="Préparer la présentation pour la réunion client importante",
            priority=Priority.URGENT,
            due_date=datetime.now() + timedelta(hours=24),
            tags=["client", "présentation", "important"],
            assigned_to="Haitam"
        )
        
        # Tâche moyenne
        tache_medium = tm.create_task(
            title="Refactorisation Code",
            description="Nettoyer et optimiser le code du module principal",
            priority=Priority.MEDIUM,
            due_date=datetime.now() + timedelta(days=3),
            tags=["développement", "optimisation"],
            assigned_to="Développeur"
        )
        
        print("✅ Tâches créées avec succès!")
        
        # Liste toutes les tâches
        print("\n📋 Toutes les tâches:")
        tasks = tm.list_tasks()
        for task in tasks:
            print(f"  • {task.title} ({task.priority.name}) - Échéance: {task.due_date.strftime('%d/%m/%Y')}")
        
        # Marquer une tâche comme complétée
        tm.update_task_status(tache_medium.id, Status.COMPLETED)
        print(f"\n✅ Tâche '{tache_medium.title}' marquée comme complétée!")
        
        # Tâches dues bientôt
        print("\n⏰ Tâches dues dans les 7 jours:")
        due_soon = tm.get_tasks_due_soon(7)
        for task in due_soon:
            print(f"  • {task.title} - {task.due_date.strftime('%d/%m/%Y')}")
            
    except Exception as e:
        logger.error(f"Erreur lors de la démonstration: {e}")
        print(f"❌ Erreur: {e}")

if __name__ == "__main__":
    demo_systeme_taches()

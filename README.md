"""
Task Management System - Professional Grade
Author: Haitam Faddi
Date: 2024
Description: A complete task management system with SQLite database,
             RESTful API endpoints, and advanced features.
"""

import sqlite3
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
import json
from enum import Enum
import logging
from dataclasses import dataclass

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class Priority(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4

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
            'tags': self.tags
        }

class TaskManager:
    """Advanced task management system with database persistence"""
    
    def __init__(self, db_path: str = 'tasks.db'):
        self.db_path = db_path
        self._init_database()
    
    def _init_database(self) -> None:
        """Initialize database schema"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS tasks (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    title TEXT NOT NULL,
                    description TEXT,
                    priority INTEGER NOT NULL,
                    status TEXT NOT NULL,
                    due_date TIMESTAMP NOT NULL,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    tags TEXT DEFAULT '[]'
                )
            ''')
            conn.commit()
        logger.info("Database initialized successfully")
    
    def create_task(self, title: str, description: str, priority: Priority, 
                   due_date: datetime, tags: List[str] = None) -> Task:
        """Create a new task with validation"""
        if not title or len(title.strip()) == 0:
            raise ValueError("Task title cannot be empty")
        
        if due_date < datetime.now():
            raise ValueError("Due date cannot be in the past")
        
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
            tags=tags
        )
        
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT INTO tasks (title, description, priority, status, due_date, tags)
                VALUES (?, ?, ?, ?, ?, ?)
            ''', (
                task.title, 
                task.description, 
                task.priority.value, 
                task.status.value,
                task.due_date.isoformat(),
                json.dumps(task.tags)
            ))
            task.id = cursor.lastrowid
            conn.commit()
        
        logger.info(f"Task created successfully: {task.title}")
        return task
    
    def get_task(self, task_id: int) -> Optional[Task]:
        """Retrieve a task by ID"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT * FROM tasks WHERE id = ?', (task_id,))
            row = cursor.fetchone()
            
            if not row:
                return None
            
            return self._row_to_task(row)
    
    def update_task(self, task_id: int, **kwargs) -> Optional[Task]:
        """Update task properties with validation"""
        allowed_fields = {'title', 'description', 'priority', 'status', 'due_date', 'tags'}
        update_fields = {k: v for k, v in kwargs.items() if k in allowed_fields}
        
        if not update_fields:
            raise ValueError("No valid fields to update")
        
        # Build dynamic update query
        set_clause = ', '.join([f"{field} = ?" for field in update_fields.keys()])
        set_clause += ', updated_at = ?'
        values = list(update_fields.values())
        values.append(datetime.now().isoformat())
        values.append(task_id)
        
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute(f'''
                UPDATE tasks 
                SET {set_clause}
                WHERE id = ?
            ''', values)
            
            if cursor.rowcount == 0:
                return None
            
            conn.commit()
        
        logger.info(f"Task {task_id} updated successfully")
        return self.get_task(task_id)
    
    def delete_task(self, task_id: int) -> bool:
        """Delete a task by ID"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute('DELETE FROM tasks WHERE id = ?', (task_id,))
            conn.commit()
            
            success = cursor.rowcount > 0
            if success:
                logger.info(f"Task {task_id} deleted successfully")
            else:
                logger.warning(f"Task {task_id} not found for deletion")
            
            return success
    
    def list_tasks(self, status: Optional[Status] = None, 
                  priority: Optional[Priority] = None) -> List[Task]:
        """List tasks with optional filtering"""
        query = 'SELECT * FROM tasks'
        conditions = []
        params = []
        
        if status:
            conditions.append('status = ?')
            params.append(status.value)
        
        if priority:
            conditions.append('priority = ?')
            params.append(priority.value)
        
        if conditions:
            query += ' WHERE ' + ' AND '.join(conditions)
        
        query += ' ORDER BY due_date ASC, priority DESC'
        
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute(query, params)
            rows = cursor.fetchall()
            
            return [self._row_to_task(row) for row in rows]
    
    def get_overdue_tasks(self) -> List[Task]:
        """Retrieve all overdue tasks"""
        current_time = datetime.now().isoformat()
        
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute('''
                SELECT * FROM tasks 
                WHERE due_date < ? AND status NOT IN (?, ?)
                ORDER BY due_date ASC
            ''', (current_time, Status.COMPLETED.value, Status.CANCELLED.value))
            
            rows = cursor.fetchall()
            return [self._row_to_task(row) for row in rows]
    
    def _row_to_task(self, row: Tuple) -> Task:
        """Convert database row to Task object"""
        return Task(
            id=row[0],
            title=row[1],
            description=row[2],
            priority=Priority(row[3]),
            status=Status(row[4]),
            due_date=datetime.fromisoformat(row[5]),
            created_at=datetime.fromisoformat(row[6]),
            updated_at=datetime.fromisoformat(row[7]),
            tags=json.loads(row[8])
        )

# Example usage and demonstration
def demonstrate_task_manager():
    """Demonstrate the TaskManager functionality"""
    print("🚀 Task Management System Demonstration")
    print("=" * 50)
    
    # Initialize task manager
    tm = TaskManager()
    
    # Create sample tasks
    try:
        task1 = tm.create_task(
            title="Complete Project Proposal",
            description="Draft and review the project proposal document",
            priority=Priority.HIGH,
            due_date=datetime.now() + timedelta(days=2),
            tags=["work", "urgent", "documentation"]
        )
        
        task2 = tm.create_task(
            title="Team Meeting Preparation",
            description="Prepare agenda and materials for team meeting",
            priority=Priority.MEDIUM,
            due_date=datetime.now() + timedelta(days=1),
            tags=["meeting", "preparation"]
        )
        
        print("✅ Tasks created successfully!")
        
        # List all tasks
        print("\n📋 All Tasks:")
        tasks = tm.list_tasks()
        for task in tasks:
            print(f"  - {task.title} (Due: {task.due_date.strftime('%Y-%m-%d')})")
        
        # Update a task
        updated_task = tm.update_task(
            task1.id, 
            status=Status.IN_PROGRESS,
            description="Draft, review, and finalize project proposal"
        )
        
        print(f"\n🔄 Task updated: {updated_task.title} - Status: {updated_task.status.value}")
        
        # Demonstrate filtering
        print("\n🔍 In-Progress Tasks:")
        in_progress_tasks = tm.list_tasks(status=Status.IN_PROGRESS)
        for task in in_progress_tasks:
            print(f"  - {task.title}")
            
    except Exception as e:
        logger.error(f"Error during demonstration: {e}")
        print(f"❌ Error: {e}")

if __name__ == "__main__":
    demonstrate_task_manager()

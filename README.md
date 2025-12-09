# **API интерфейсы для интеграции с системой колледжа**

## **1. API: Получение списка студентов группы**

### **Информация об API:**
- **Название:** Students API
- **Путь:** `/api/v1/groups/{group_id}/students`
- **Описание:** Получение списка студентов конкретной группы

### **Запрос:**
```
GET /api/v1/groups/IT-21-1/students
Headers:
  Authorization: Bearer {token}
  Content-Type: application/json
```

### **Ответ (200 OK):**
```json
{
  "status": "success",
  "data": {
    "group_id": "IT-21-1",
    "group_name": "Информационные технологии 2021",
    "students": [
      {
        "student_id": "ST2021001",
        "last_name": "Иванов",
        "first_name": "Иван",
        "middle_name": "Иванович",
        "email": "ivanov@college.ru",
        "phone": "+79991234567",
        "status": "active",
        "enrollment_date": "2021-09-01"
      },
      {
        "student_id": "ST2021002",
        "last_name": "Петрова",
        "first_name": "Мария",
        "middle_name": "Сергеевна",
        "email": "petrova@college.ru",
        "phone": "+79991234568",
        "status": "active",
        "enrollment_date": "2021-09-01"
      }
    ],
    "total": 25,
    "curator": {
      "teacher_id": "TC2020001",
      "last_name": "Сидоров",
      "first_name": "Алексей"
    }
  }
}
```

### **Ответ (404 Not Found):**
```json
{
  "status": "error",
  "message": "Группа не найдена",
  "code": "GROUP_NOT_FOUND"
}
```

### **Контроллер на стороне колледжа (Python Flask):**
```python
from flask import Flask, jsonify, request
from flask_jwt_extended import jwt_required, get_jwt_identity
import logging

app = Flask(__name__)

class CollegeStudentsController:
    
    @app.route('/api/v1/groups/<string:group_id>/students', methods=['GET'])
    @jwt_required()
    def get_group_students(group_id):
        """
        Получение списка студентов группы
        """
        try:
            # Проверка прав доступа
            user = get_jwt_identity()
            if not has_access_to_group(user, group_id):
                return jsonify({
                    "status": "error",
                    "message": "Доступ запрещен",
                    "code": "ACCESS_DENIED"
                }), 403
            
            # Получение данных из БД колледжа
            students = get_students_from_database(group_id)
            
            if not students:
                return jsonify({
                    "status": "error",
                    "message": "Группа не найдена",
                    "code": "GROUP_NOT_FOUND"
                }), 404
            
            # Формирование ответа
            response = {
                "status": "success",
                "data": {
                    "group_id": group_id,
                    "group_name": students[0]['group_name'] if students else "",
                    "students": [
                        {
                            "student_id": s['student_id'],
                            "last_name": s['last_name'],
                            "first_name": s['first_name'],
                            "middle_name": s['middle_name'],
                            "email": s['email'],
                            "phone": s['phone'],
                            "status": s['status'],
                            "enrollment_date": s['enrollment_date'].isoformat()
                        }
                        for s in students
                    ],
                    "total": len(students),
                    "curator": get_group_curator(group_id)
                }
            }
            
            return jsonify(response), 200
            
        except Exception as e:
            logging.error(f"Error in get_group_students: {str(e)}")
            return jsonify({
                "status": "error",
                "message": "Внутренняя ошибка сервера",
                "code": "INTERNAL_SERVER_ERROR"
            }), 500
    
    def has_access_to_group(user, group_id):
        """Проверка прав доступа пользователя к группе"""
        # Реализация проверки прав
        return True
    
    def get_students_from_database(group_id):
        """Получение студентов из БД колледжа"""
        # Заглушка для примера
        return [
            {
                'student_id': 'ST2021001',
                'last_name': 'Иванов',
                'first_name': 'Иван',
                'middle_name': 'Иванович',
                'email': 'ivanov@college.ru',
                'phone': '+79991234567',
                'status': 'active',
                'enrollment_date': '2021-09-01',
                'group_name': 'Информационные технологии 2021'
            }
        ]
    
    def get_group_curator(group_id):
        """Получение куратора группы"""
        return {
            'teacher_id': 'TC2020001',
            'last_name': 'Сидоров',
            'first_name': 'Алексей'
        }
```

### **Юнит-тесты:**
```python
import unittest
import json
from flask_testing import TestCase
from app import app

class TestStudentsAPI(TestCase):
    
    def create_app(self):
        app.config['TESTING'] = True
        app.config['JWT_SECRET_KEY'] = 'test-secret-key'
        return app
    
    def setUp(self):
        self.client = app.test_client()
        self.valid_token = 'valid-test-token'
    
    def test_get_group_students_success(self):
        """Тест успешного получения списка студентов"""
        headers = {
            'Authorization': f'Bearer {self.valid_token}',
            'Content-Type': 'application/json'
        }
        
        response = self.client.get(
            '/api/v1/groups/IT-21-1/students',
            headers=headers
        )
        
        self.assertEqual(response.status_code, 200)
        data = json.loads(response.data)
        
        self.assertEqual(data['status'], 'success')
        self.assertIn('students', data['data'])
        self.assertIsInstance(data['data']['students'], list)
        self.assertGreater(len(data['data']['students']), 0)
    
    def test_get_group_students_not_found(self):
        """Тест запроса несуществующей группы"""
        headers = {
            'Authorization': f'Bearer {self.valid_token}',
            'Content-Type': 'application/json'
        }
        
        response = self.client.get(
            '/api/v1/groups/NONEXISTENT/students',
            headers=headers
        )
        
        self.assertEqual(response.status_code, 404)
        data = json.loads(response.data)
        self.assertEqual(data['code'], 'GROUP_NOT_FOUND')
    
    def test_get_group_students_unauthorized(self):
        """Тест запроса без авторизации"""
        response = self.client.get('/api/v1/groups/IT-21-1/students')
        
        self.assertEqual(response.status_code, 401)
    
    def test_get_group_students_invalid_token(self):
        """Тест запроса с невалидным токеном"""
        headers = {
            'Authorization': 'Bearer invalid-token',
            'Content-Type': 'application/json'
        }
        
        response = self.client.get(
            '/api/v1/groups/IT-21-1/students',
            headers=headers
        )
        
        self.assertEqual(response.status_code, 401)

if __name__ == '__main__':
    unittest.main()
```

---

## **2. API: Получение расписания группы**

### **Информация об API:**
- **Название:** Schedule API
- **Путь:** `/api/v1/groups/{group_id}/schedule`
- **Описание:** Получение расписания занятий группы

### **Запрос:**
```
GET /api/v1/groups/IT-21-1/schedule?week=2024-12-09
Headers:
  Authorization: Bearer {token}
  Content-Type: application/json
Query Parameters:
  week: Дата любой день недели (формат: YYYY-MM-DD)
```

### **Ответ (200 OK):**
```json
{
  "status": "success",
  "data": {
    "group_id": "IT-21-1",
    "week_start": "2024-12-09",
    "week_end": "2024-12-15",
    "schedule": {
      "monday": [
        {
          "lesson_id": "LES001",
          "time": "09:00-10:30",
          "subject": "Программирование",
          "teacher": "Сидоров А.А.",
          "room": "А-301",
          "type": "lecture"
        }
      ],
      "tuesday": [
        {
          "lesson_id": "LES002",
          "time": "10:40-12:10",
          "subject": "Базы данных",
          "teacher": "Петрова М.С.",
          "room": "Б-205",
          "type": "practice"
        }
      ]
    }
  }
}
```

### **Контроллер:**
```python
class CollegeScheduleController:
    
    @app.route('/api/v1/groups/<string:group_id>/schedule', methods=['GET'])
    @jwt_required()
    def get_group_schedule(group_id):
        """
        Получение расписания группы
        """
        try:
            week_date = request.args.get('week')
            
            # Валидация параметров
            if week_date:
                try:
                    datetime.strptime(week_date, '%Y-%m-%d')
                except ValueError:
                    return jsonify({
                        "status": "error",
                        "message": "Неверный формат даты. Используйте YYYY-MM-DD",
                        "code": "INVALID_DATE_FORMAT"
                    }), 400
            
            # Получение расписания
            schedule_data = get_schedule_from_database(group_id, week_date)
            
            if not schedule_data:
                return jsonify({
                    "status": "error",
                    "message": "Расписание не найдено",
                    "code": "SCHEDULE_NOT_FOUND"
                }), 404
            
            return jsonify({
                "status": "success",
                "data": schedule_data
            }), 200
            
        except Exception as e:
            logging.error(f"Error in get_group_schedule: {str(e)}")
            return jsonify({
                "status": "error",
                "message": "Внутренняя ошибка сервера",
                "code": "INTERNAL_SERVER_ERROR"
            }), 500
```

---

## **3. API: Создание нового задания в журнале успеваемости**

### **Информация об API:**
- **Название:** Create Assignment API
- **Путь:** `/api/v1/assignments`
- **Описание:** Создание нового задания в системе колледжа

### **Запрос:**
```
POST /api/v1/assignments
Headers:
  Authorization: Bearer {token}
  Content-Type: application/json

Body:
{
  "subject_id": "SUB001",
  "teacher_id": "TC2020001",
  "group_ids": ["IT-21-1", "IT-21-2"],
  "title": "Лабораторная работа №3",
  "description": "Реализация алгоритма сортировки",
  "deadline": "2024-12-20T23:59:59",
  "max_points": 100,
  "assignment_type": "lab_work"
}
```

### **Ответ (201 Created):**
```json
{
  "status": "success",
  "data": {
    "assignment_id": "ASG20241215001",
    "subject_id": "SUB001",
    "teacher_id": "TC2020001",
    "groups": ["IT-21-1", "IT-21-2"],
    "title": "Лабораторная работа №3",
    "deadline": "2024-12-20T23:59:59",
    "max_points": 100,
    "created_at": "2024-12-15T10:30:00",
    "status": "active"
  }
}
```

### **Контроллер:**
```python
class CollegeAssignmentsController:
    
    @app.route('/api/v1/assignments', methods=['POST'])
    @jwt_required()
    def create_assignment():
        """
        Создание нового задания
        """
        try:
            data = request.get_json()
            
            # Валидация данных
            required_fields = ['subject_id', 'teacher_id', 'group_ids', 'title', 'deadline']
            for field in required_fields:
                if field not in data:
                    return jsonify({
                        "status": "error",
                        "message": f"Отсутствует обязательное поле: {field}",
                        "code": "MISSING_REQUIRED_FIELD"
                    }), 400
            
            # Проверка существования предмета и преподавателя
            if not validate_subject_teacher(data['subject_id'], data['teacher_id']):
                return jsonify({
                    "status": "error",
                    "message": "Предмет или преподаватель не найдены",
                    "code": "SUBJECT_TEACHER_NOT_FOUND"
                }), 404
            
            # Создание задания в БД
            assignment_id = create_assignment_in_database(data)
            
            return jsonify({
                "status": "success",
                "data": {
                    "assignment_id": assignment_id,
                    **data,
                    "created_at": datetime.now().isoformat(),
                    "status": "active"
                }
            }), 201
            
        except Exception as e:
            logging.error(f"Error in create_assignment: {str(e)}")
            return jsonify({
                "status": "error",
                "message": "Внутренняя ошибка сервера",
                "code": "INTERNAL_SERVER_ERROR"
            }), 500
```

---

## **4. API: Получение успеваемости студента**

### **Информация об API:**
- **Название:** Student Performance API
- **Путь:** `/api/v1/students/{student_id}/performance`
- **Описание:** Получение успеваемости студента по предметам

### **Запрос:**
```
GET /api/v1/students/ST2021001/performance?semester=1&academic_year=2024-2025
Headers:
  Authorization: Bearer {token}
  Content-Type: application/json
```

### **Ответ (200 OK):**
```json
{
  "status": "success",
  "data": {
    "student_id": "ST2021001",
    "student_name": "Иванов Иван Иванович",
    "academic_year": "2024-2025",
    "semester": 1,
    "performance": [
      {
        "subject_id": "SUB001",
        "subject_name": "Программирование",
        "teacher": "Сидоров А.А.",
        "total_assignments": 10,
        "completed_assignments": 8,
        "average_score": 85.5,
        "final_grade": 4,
        "last_activity": "2024-12-10T14:30:00"
      },
      {
        "subject_id": "SUB002",
        "subject_name": "Базы данных",
        "teacher": "Петрова М.С.",
        "total_assignments": 8,
        "completed_assignments": 6,
        "average_score": 78.2,
        "final_grade": 3,
        "last_activity": "2024-12-08T10:15:00"
      }
    ],
    "summary": {
      "gpa": 3.5,
      "total_subjects": 6,
      "completed_subjects": 4,
      "attendance_rate": 92.5
    }
  }
}
```

---

## **5. API: Обновление оценки за задание**

### **Информация об API:**
- **Название:** Update Grade API
- **Путь:** `/api/v1/assignments/{assignment_id}/grades`
- **Описание:** Обновление оценки студента за задание

### **Запрос:**
```
PUT /api/v1/assignments/ASG20241215001/grades
Headers:
  Authorization: Bearer {token}
  Content-Type: application/json

Body:
{
  "student_id": "ST2021001",
  "grade": 95,
  "teacher_comment": "Отличная работа, все требования выполнены",
  "graded_by": "TC2020001"
}
```

### **Ответ (200 OK):**
```json
{
  "status": "success",
  "data": {
    "assignment_id": "ASG20241215001",
    "student_id": "ST2021001",
    "grade": 95,
    "previous_grade": null,
    "graded_at": "2024-12-15T14:20:00",
    "graded_by": "TC2020001",
    "status": "graded"
  }
}
```

---

## **6. API: Получение списка предметов преподавателя**

### **Информация об API:**
- **Название:** Teacher Subjects API
- **Путь:** `/api/v1/teachers/{teacher_id}/subjects`
- **Описание:** Получение списка предметов, которые ведет преподаватель

### **Запрос:**
```
GET /api/v1/teachers/TC2020001/subjects?academic_year=2024-2025
Headers:
  Authorization: Bearer {token}
```

### **Ответ (200 OK):**
```json
{
  "status": "success",
  "data": {
    "teacher_id": "TC2020001",
    "teacher_name": "Сидоров Алексей Алексеевич",
    "academic_year": "2024-2025",
    "subjects": [
      {
        "subject_id": "SUB001",
        "subject_name": "Программирование",
        "subject_code": "PRG101",
        "groups": ["IT-21-1", "IT-21-2", "IT-22-1"],
        "total_students": 75,
        "hours_per_week": 6,
        "semester": 1
      },
      {
        "subject_id": "SUB003",
        "subject_name": "Алгоритмы и структуры данных",
        "subject_code": "ALG201",
        "groups": ["IT-21-1"],
        "total_students": 25,
        "hours_per_week": 4,
        "semester": 2
      }
    ]
  }
}
```

---

## **7. API: Отметка посещаемости**

### **Информация об API:**
- **Название:** Attendance API
- **Путь:** `/api/v1/attendance`
- **Описание:** Отметка посещаемости студента на занятии

### **Запрос:**
```
POST /api/v1/attendance
Headers:
  Authorization: Bearer {token}
  Content-Type: application/json

Body:
{
  "lesson_id": "LES001",
  "student_id": "ST2021001",
  "attendance_date": "2024-12-15",
  "status": "present",
  "marked_by": "TC2020001",
  "notes": "Присутствовал все занятие"
}
```

### **Ответ (201 Created):**
```json
{
  "status": "success",
  "data": {
    "attendance_id": "ATT20241215001",
    "lesson_id": "LES001",
    "student_id": "ST2021001",
    "attendance_date": "2024-12-15",
    "status": "present",
    "marked_at": "2024-12-15T09:05:00",
    "marked_by": "TC2020001"
  }
}
```

---

## **8. API: Получение статистики по группе**

### **Информация об API:**
- **Название:** Group Statistics API
- **Путь:** `/api/v1/groups/{group_id}/statistics`
- **Описание:** Получение статистических данных по группе

### **Запрос:**
```
GET /api/v1/groups/IT-21-1/statistics?period=month&start_date=2024-12-01&end_date=2024-12-31
Headers:
  Authorization: Bearer {token}
```

### **Ответ (200 OK):**
```json
{
  "status": "success",
  "data": {
    "group_id": "IT-21-1",
    "period": {
      "start": "2024-12-01",
      "end": "2024-12-31"
    },
    "statistics": {
      "attendance": {
        "total_lessons": 48,
        "average_attendance": 89.5,
        "best_attendance_student": "ST2021001",
        "worst_attendance_student": "ST2021005"
      },
      "performance": {
        "average_gpa": 3.8,
        "best_student": "ST2021001",
        "worst_student": "ST2021005",
        "distribution": {
          "excellent": 8,
          "good": 12,
          "satisfactory": 4,
          "unsatisfactory": 1
        }
      },
      "assignments": {
        "total_assigned": 25,
        "average_completion_rate": 92.3,
        "average_score": 82.7
      }
    },
    "trends": {
      "attendance_trend": "up",
      "performance_trend": "stable",
      "completion_rate_trend": "up"
    }
  }
}
```

---

## **Общий контроллер с обработкой ошибок:**

```python
class CollegeAPIController:
    """
    Основной контроллер API системы колледжа
    """
    
    @app.errorhandler(404)
    def not_found(error):
        return jsonify({
            "status": "error",
            "message": "Ресурс не найден",
            "code": "RESOURCE_NOT_FOUND"
        }), 404
    
    @app.errorhandler(400)
    def bad_request(error):
        return jsonify({
            "status": "error",
            "message": "Некорректный запрос",
            "code": "BAD_REQUEST"
        }), 400
    
    @app.errorhandler(401)
    def unauthorized(error):
        return jsonify({
            "status": "error",
            "message": "Требуется авторизация",
            "code": "UNAUTHORIZED"
        }), 401
    
    @app.errorhandler(403)
    def forbidden(error):
        return jsonify({
            "status": "error",
            "message": "Доступ запрещен",
            "code": "FORBIDDEN"
        }), 403
    
    @app.errorhandler(500)
    def internal_error(error):
        return jsonify({
            "status": "error",
            "message": "Внутренняя ошибка сервера",
            "code": "INTERNAL_SERVER_ERROR"
        }), 500
```

---

## **Юнит-тесты для всех API:**

```python
import unittest
import json
from datetime import datetime
from flask_testing import TestCase
from app import app

class TestCollegeAPIs(TestCase):
    
    def create_app(self):
        app.config['TESTING'] = True
        app.config['JWT_SECRET_KEY'] = 'test-secret-key'
        return app
    
    def setUp(self):
        self.client = app.test_client()
        self.valid_token = 'valid-test-token'
        self.headers = {
            'Authorization': f'Bearer {self.valid_token}',
            'Content-Type': 'application/json'
        }
    
    def test_all_apis_require_auth(self):
        """Проверка, что все API требуют авторизацию"""
        endpoints = [
            ('/api/v1/groups/IT-21-1/students', 'GET'),
            ('/api/v1/assignments', 'POST'),
            ('/api/v1/students/ST2021001/performance', 'GET'),
        ]
        
        for endpoint, method in endpoints:
            if method == 'GET':
                response = self.client.get(endpoint)
            elif method == 'POST':
                response = self.client.post(endpoint)
            
            self.assertEqual(response.status_code, 401)
    
    def test_create_assignment_validation(self):
        """Тест валидации при создании задания"""
        # Тест без обязательных полей
        invalid_data = {
            "title": "Тестовое задание"
            # Нет subject_id, teacher_id и других обязательных полей
        }
        
        response = self.client.post(
            '/api/v1/assignments',
            headers=self.headers,
            data=json.dumps(invalid_data)
        )
        
        self.assertEqual(response.status_code, 400)
        data = json.loads(response.data)
        self.assertEqual(data['code'], 'MISSING_REQUIRED_FIELD')
    
    def test_performance_api_with_filters(self):
        """Тест API успеваемости с фильтрами"""
        # Тест с валидными параметрами
        response = self.client.get(
            '/api/v1/students/ST2021001/performance?semester=1&academic_year=2024-2025',
            headers=self.headers
        )
        
        self.assertEqual(response.status_code, 200)
        data = json.loads(response.data)
        
        # Проверка структуры ответа
        self.assertIn('performance', data['data'])
        self.assertIn('summary', data['data'])
        self.assertEqual(data['data']['student_id'], 'ST2021001')
    
    def test_update_grade_scenarios(self):
        """Тест различных сценариев обновления оценки"""
        test_cases = [
            {
                "name": "Валидное обновление",
                "data": {
                    "student_id": "ST2021001",
                    "grade": 95,
                    "teacher_comment": "Хорошая работа",
                    "graded_by": "TC2020001"
                },
                "expected_status": 200
            },
            {
                "name": "Оценка вне диапазона",
                "data": {
                    "student_id": "ST2021001",
                    "grade": 150,  # Некорректная оценка
                    "teacher_comment": "Тест",
                    "graded_by": "TC2020001"
                },
                "expected_status": 400
            }
        ]
        
        for test_case in test_cases:
            with self.subTest(test_case['name']):
                response = self.client.put(
                    '/api/v1/assignments/ASG001/grades',
                    headers=self.headers,
                    data=json.dumps(test_case['data'])
                )
                
                self.assertEqual(response.status_code, test_case['expected_status'])
    
    def test_schedule_api_date_validation(self):
        """Тест валидации даты в API расписания"""
        # Тест с невалидной датой
        response = self.client.get(
            '/api/v1/groups/IT-21-1/schedule?week=invalid-date',
            headers=self.headers
        )
        
        self.assertEqual(response.status_code, 400)
        data = json.loads(response.data)
        self.assertEqual(data['code'], 'INVALID_DATE_FORMAT')
        
        # Тест с валидной датой
        response = self.client.get(
            '/api/v1/groups/IT-21-1/schedule?week=2024-12-09',
            headers=self.headers
        )
        
        self.assertEqual(response.status_code, 200)
    
    def test_teacher_subjects_api(self):
        """Тест API предметов преподавателя"""
        response = self.client.get(
            '/api/v1/teachers/TC2020001/subjects',
            headers=self.headers
        )
        
        self.assertEqual(response.status_code, 200)
        data = json.loads(response.data)
        
        # Проверка структуры ответа
        self.assertEqual(data['data']['teacher_id'], 'TC2020001')
        self.assertIsInstance(data['data']['subjects'], list)
        
        if data['data']['subjects']:
            subject = data['data']['subjects'][0]
            self.assertIn('subject_id', subject)
            self.assertIn('subject_name', subject)
            self.assertIn('groups', subject)
    
    def test_attendance_api_scenarios(self):
        """Тест различных сценариев отметки посещаемости"""
        valid_attendance = {
            "lesson_id": "LES001",
            "student_id": "ST2021001",
            "attendance_date": "2024-12-15",
            "status": "present",
            "marked_by": "TC2020001"
        }
        
        # Тест успешного создания
        response = self.client.post(
            '/api/v1/attendance',
            headers=self.headers,
            data=json.dumps(valid_attendance)
        )
        
        self.assertEqual(response.status_code, 201)
        data = json.loads(response.data)
        self.assertEqual(data['data']['status'], 'present')
        
        # Тест с невалидным статусом
        invalid_attendance = valid_attendance.copy()
        invalid_attendance['status'] = 'invalid_status'
        
        response = self.client.post(
            '/api/v1/attendance',
            headers=self.headers,
            data=json.dumps(invalid_attendance)
        )
        
        self.assertEqual(response.status_code, 400)
    
    def test_group_statistics_api(self):
        """Тест API статистики группы"""
        # Тест с параметрами периода
        response = self.client.get(
            '/api/v1/groups/IT-21-1/statistics?period=month&start_date=2024-12-01&end_date=2024-12-31',
            headers=self.headers
        )
        
        self.assertEqual(response.status_code, 200)
        data = json.loads(response.data)
        
        # Проверка структуры ответа
        self.assertIn('statistics', data['data'])
        self.assertIn('attendance', data['data']['statistics'])
        self.assertIn('performance', data['data']['statistics'])
        self.assertIn('assignments', data['data']['statistics'])
        self.assertIn('trends', data['data'])

if __name__ == '__main__':
    unittest.main(verbosity=2)
```

---

## **Документация по использованию API:**

### **Аутентификация:**
Все API требуют Bearer токен в заголовке Authorization:
```
Authorization: Bearer {your_jwt_token}
```

### **Коды ошибок:**
- `GROUP_NOT_FOUND` - Группа не найдена
- `STUDENT_NOT_FOUND` - Студент не найден
- `TEACHER_NOT_FOUND` - Преподаватель не найден
- `SUBJECT_NOT_FOUND` - Предмет не найден
- `ASSIGNMENT_NOT_FOUND` - Задание не найдено
- `LESSON_NOT_FOUND` - Занятие не найдено
- `ACCESS_DENIED` - Доступ запрещен
- `INVALID_DATE_FORMAT` - Неверный формат даты
- `MISSING_REQUIRED_FIELD` - Отсутствует обязательное поле
- `INVALID_GRADE_VALUE` - Некорректное значение оценки
- `DUPLICATE_RECORD` - Дублирующаяся запись
- `INTERNAL_SERVER_ERROR` - Внутренняя ошибка сервера

### **Форматы данных:**
- **Даты:** YYYY-MM-DD
- **Дата-время:** ISO 8601 (YYYY-MM-DDTHH:MM:SS)
- **Оценки:** Число от 0 до 100
- **Идентификаторы:** Строка, уникальная в рамках системы

### **Пагинация (где применимо):**
```json
{
  "page": 1,
  "per_page": 20,
  "total": 100,
  "total_pages": 5
}
```

### **Сортировка (где применимо):**
```
?sort=last_name&order=asc
```

---

## **Интеграционные сценарии:**

### **Сценарий 1: Создание задания в системе TaskFlow**
1. TaskFlow вызывает `POST /api/v1/assignments`
2. Колледж создает задание и возвращает ID
3. TaskFlow сохраняет ID задания для дальнейшей синхронизации

### **Сценарий 2: Синхронизация оценок**
1. Преподаватель выставляет оценку в TaskFlow
2. TaskFlow вызывает `PUT /api/v1/assignments/{id}/grades`
3. Колледж обновляет оценку в журнале успеваемости

### **Сценарий 3: Ежедневная синхронизация данных**
1. TaskFlow загружает список студентов через `GET /api/v1/groups/{id}/students`
2. Загружает расписание через `GET /api/v1/groups/{id}/schedule`
3. Обновляет успеваемость через `GET /api/v1/students/{id}/performance`

### **Сценарий 4: Отчетность для куратора**
1. TaskFlow запрашивает статистику через `GET /api/v1/groups/{id}/statistics`
2. Формирует расширенный отчет на основе данных колледжа
3. Предоставляет куратору комплексную аналитику

---

Эта система API обеспечивает полную интеграцию TaskFlow с существующей инфраструктурой колледжа, позволяя синхронизировать данные в реальном времени и избегать дублирования информации.

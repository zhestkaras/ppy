


### **1. StudentsController.cs**
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using System.Collections.Generic;
using CollegeSystem.API.Services;
using CollegeSystem.API.Models;

namespace CollegeSystem.API.Controllers
{
    [ApiController]
    [Route("api/v1/[controller]")]
    public class StudentsController : ControllerBase
    {
        private readonly IStudentService _studentService;
        private readonly IAuthorizationService _authService;
        private readonly ILogger<StudentsController> _logger;

        public StudentsController(
            IStudentService studentService,
            IAuthorizationService authService,
            ILogger<StudentsController> logger)
        {
            _studentService = studentService;
            _authService = authService;
            _logger = logger;
        }

        [HttpGet("groups/{groupId}/students")]
        public async Task<IActionResult> GetGroupStudents(string groupId)
        {
            try
            {
                // Проверка авторизации
                var token = Request.Headers["Authorization"].ToString().Replace("Bearer ", "");
                if (!await _authService.ValidateTokenAsync(token))
                {
                    return Unauthorized(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "UNAUTHORIZED",
                        Message = "Требуется авторизация"
                    });
                }

                // Проверка доступа к группе
                if (!await _authService.HasAccessToGroupAsync(token, groupId))
                {
                    return StatusCode(403, new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "ACCESS_DENIED",
                        Message = "Доступ запрещен"
                    });
                }

                // Получение студентов
                var students = await _studentService.GetStudentsByGroupAsync(groupId);
                if (students == null || students.Count == 0)
                {
                    return NotFound(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "GROUP_NOT_FOUND",
                        Message = "Группа не найдена"
                    });
                }

                // Формирование ответа
                var response = new GroupStudentsResponse
                {
                    GroupId = groupId,
                    GroupName = students[0].GroupName,
                    Students = students,
                    Total = students.Count,
                    Curator = await _studentService.GetGroupCuratorAsync(groupId)
                };

                return Ok(new ApiResponse<GroupStudentsResponse>
                {
                    Success = true,
                    Data = response
                });
            }
            catch (System.Exception ex)
            {
                _logger.LogError(ex, "Ошибка при получении студентов группы {GroupId}", groupId);
                return StatusCode(500, new ApiResponse<object>
                {
                    Success = false,
                    ErrorCode = "INTERNAL_SERVER_ERROR",
                    Message = "Внутренняя ошибка сервера"
                });
            }
        }
    }
}
```

### **2. AssignmentsController.cs**
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using CollegeSystem.API.Services;
using CollegeSystem.API.Models;
using System;

namespace CollegeSystem.API.Controllers
{
    [ApiController]
    [Route("api/v1/[controller]")]
    public class AssignmentsController : ControllerBase
    {
        private readonly IAssignmentService _assignmentService;
        private readonly IAuthorizationService _authService;
        private readonly IValidationService _validationService;
        private readonly ILogger<AssignmentsController> _logger;

        public AssignmentsController(
            IAssignmentService assignmentService,
            IAuthorizationService authService,
            IValidationService validationService,
            ILogger<AssignmentsController> logger)
        {
            _assignmentService = assignmentService;
            _authService = authService;
            _validationService = validationService;
            _logger = logger;
        }

        [HttpPost]
        public async Task<IActionResult> CreateAssignment([FromBody] CreateAssignmentRequest request)
        {
            try
            {
                // Проверка авторизации
                var token = Request.Headers["Authorization"].ToString().Replace("Bearer ", "");
                if (!await _authService.ValidateTokenAsync(token))
                {
                    return Unauthorized(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "UNAUTHORIZED",
                        Message = "Требуется авторизация"
                    });
                }

                // Проверка прав (только преподаватели)
                if (!await _authService.IsTeacherAsync(token))
                {
                    return StatusCode(403, new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "FORBIDDEN",
                        Message = "Только преподаватели могут создавать задания"
                    });
                }

                // Валидация запроса
                var (isValid, errorMessage) = _validationService.ValidateAssignmentRequest(request);
                if (!isValid)
                {
                    return BadRequest(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "INVALID_REQUEST",
                        Message = errorMessage
                    });
                }

                // Создание задания
                var assignment = await _assignmentService.CreateAssignmentAsync(request);

                return CreatedAtAction(
                    nameof(CreateAssignment),
                    new { id = assignment.AssignmentId },
                    new ApiResponse<AssignmentResponse>
                    {
                        Success = true,
                        Data = assignment
                    });
            }
            catch (NotFoundException ex)
            {
                return NotFound(new ApiResponse<object>
                {
                    Success = false,
                    ErrorCode = ex.ErrorCode,
                    Message = ex.Message
                });
            }
            catch (System.Exception ex)
            {
                _logger.LogError(ex, "Ошибка при создании задания");
                return StatusCode(500, new ApiResponse<object>
                {
                    Success = false,
                    ErrorCode = "INTERNAL_SERVER_ERROR",
                    Message = "Внутренняя ошибка сервера"
                });
            }
        }

        [HttpPut("{assignmentId}/grades")]
        public async Task<IActionResult> UpdateGrade(string assignmentId, [FromBody] UpdateGradeRequest request)
        {
            try
            {
                // Проверка авторизации
                var token = Request.Headers["Authorization"].ToString().Replace("Bearer ", "");
                if (!await _authService.ValidateTokenAsync(token))
                {
                    return Unauthorized(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "UNAUTHORIZED",
                        Message = "Требуется авторизация"
                    });
                }

                // Проверка прав (только преподаватели)
                if (!await _authService.IsTeacherAsync(token))
                {
                    return StatusCode(403, new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "FORBIDDEN",
                        Message = "Только преподаватели могут выставлять оценки"
                    });
                }

                // Валидация оценки
                if (request.Grade < 0 || request.Grade > 100)
                {
                    return BadRequest(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "INVALID_GRADE_VALUE",
                        Message = "Оценка должна быть от 0 до 100"
                    });
                }

                // Обновление оценки
                var result = await _assignmentService.UpdateGradeAsync(assignmentId, request);

                return Ok(new ApiResponse<GradeUpdateResponse>
                {
                    Success = true,
                    Data = result
                });
            }
            catch (NotFoundException ex)
            {
                return NotFound(new ApiResponse<object>
                {
                    Success = false,
                    ErrorCode = ex.ErrorCode,
                    Message = ex.Message
                });
            }
            catch (System.Exception ex)
            {
                _logger.LogError(ex, "Ошибка при обновлении оценки для задания {AssignmentId}", assignmentId);
                return StatusCode(500, new ApiResponse<object>
                {
                    Success = false,
                    ErrorCode = "INTERNAL_SERVER_ERROR",
                    Message = "Внутренняя ошибка сервера"
                });
            }
        }
    }
}
```

### **3. ScheduleController.cs**
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using CollegeSystem.API.Services;
using CollegeSystem.API.Models;

namespace CollegeSystem.API.Controllers
{
    [ApiController]
    [Route("api/v1/[controller]")]
    public class ScheduleController : ControllerBase
    {
        private readonly IScheduleService _scheduleService;
        private readonly IAuthorizationService _authService;
        private readonly ILogger<ScheduleController> _logger;

        public ScheduleController(
            IScheduleService scheduleService,
            IAuthorizationService authService,
            ILogger<ScheduleController> logger)
        {
            _scheduleService = scheduleService;
            _authService = authService;
            _logger = logger;
        }

        [HttpGet("groups/{groupId}/schedule")]
        public async Task<IActionResult> GetGroupSchedule(string groupId, [FromQuery] string week = null)
        {
            try
            {
                // Проверка авторизации
                var token = Request.Headers["Authorization"].ToString().Replace("Bearer ", "");
                if (!await _authService.ValidateTokenAsync(token))
                {
                    return Unauthorized(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "UNAUTHORIZED",
                        Message = "Требуется авторизация"
                    });
                }

                // Валидация даты (если указана)
                if (!string.IsNullOrEmpty(week))
                {
                    var (isValid, errorMessage) = _scheduleService.ValidateDate(week);
                    if (!isValid)
                    {
                        return BadRequest(new ApiResponse<object>
                        {
                            Success = false,
                            ErrorCode = "INVALID_DATE_FORMAT",
                            Message = errorMessage
                        });
                    }
                }

                // Получение расписания
                var schedule = await _scheduleService.GetGroupScheduleAsync(groupId, week);
                if (schedule == null)
                {
                    return NotFound(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "SCHEDULE_NOT_FOUND",
                        Message = "Расписание не найдено"
                    });
                }

                return Ok(new ApiResponse<ScheduleResponse>
                {
                    Success = true,
                    Data = schedule
                });
            }
            catch (System.Exception ex)
            {
                _logger.LogError(ex, "Ошибка при получении расписания группы {GroupId}", groupId);
                return StatusCode(500, new ApiResponse<object>
                {
                    Success = false,
                    ErrorCode = "INTERNAL_SERVER_ERROR",
                    Message = "Внутренняя ошибка сервера"
                });
            }
        }
    }
}
```

### **4. PerformanceController.cs**
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using CollegeSystem.API.Services;
using CollegeSystem.API.Models;

namespace CollegeSystem.API.Controllers
{
    [ApiController]
    [Route("api/v1/[controller]")]
    public class PerformanceController : ControllerBase
    {
        private readonly IPerformanceService _performanceService;
        private readonly IAuthorizationService _authService;
        private readonly ILogger<PerformanceController> _logger;

        public PerformanceController(
            IPerformanceService performanceService,
            IAuthorizationService authService,
            ILogger<PerformanceController> logger)
        {
            _performanceService = performanceService;
            _authService = authService;
            _logger = logger;
        }

        [HttpGet("students/{studentId}/performance")]
        public async Task<IActionResult> GetStudentPerformance(
            string studentId,
            [FromQuery] int? semester = null,
            [FromQuery] string academicYear = null)
        {
            try
            {
                // Проверка авторизации
                var token = Request.Headers["Authorization"].ToString().Replace("Bearer ", "");
                if (!await _authService.ValidateTokenAsync(token))
                {
                    return Unauthorized(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "UNAUTHORIZED",
                        Message = "Требуется авторизация"
                    });
                }

                // Проверка доступа к данным студента
                if (!await _authService.HasAccessToStudentAsync(token, studentId))
                {
                    return StatusCode(403, new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "ACCESS_DENIED",
                        Message = "Доступ запрещен"
                    });
                }

                // Валидация семестра
                if (semester.HasValue && (semester.Value < 1 || semester.Value > 2))
                {
                    return BadRequest(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "INVALID_SEMESTER",
                        Message = "Семестр должен быть 1 или 2"
                    });
                }

                // Получение успеваемости
                var performance = await _performanceService.GetStudentPerformanceAsync(
                    studentId, semester, academicYear);

                if (performance == null)
                {
                    return NotFound(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "STUDENT_NOT_FOUND",
                        Message = "Студент не найден"
                    });
                }

                return Ok(new ApiResponse<StudentPerformanceResponse>
                {
                    Success = true,
                    Data = performance
                });
            }
            catch (System.Exception ex)
            {
                _logger.LogError(ex, "Ошибка при получении успеваемости студента {StudentId}", studentId);
                return StatusCode(500, new ApiResponse<object>
                {
                    Success = false,
                    ErrorCode = "INTERNAL_SERVER_ERROR",
                    Message = "Внутренняя ошибка сервера"
                });
            }
        }
    }
}
```

### **7. StatisticsController.cs**
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using CollegeSystem.API.Services;
using CollegeSystem.API.Models;

namespace CollegeSystem.API.Controllers
{
    [ApiController]
    [Route("api/v1/[controller]")]
    public class StatisticsController : ControllerBase
    {
        private readonly IStatisticsService _statisticsService;
        private readonly IAuthorizationService _authService;
        private readonly ILogger<StatisticsController> _logger;

        public StatisticsController(
            IStatisticsService statisticsService,
            IAuthorizationService authService,
            ILogger<StatisticsController> logger)
        {
            _statisticsService = statisticsService;
            _authService = authService;
            _logger = logger;
        }

        [HttpGet("groups/{groupId}/statistics")]
        public async Task<IActionResult> GetGroupStatistics(
            string groupId,
            [FromQuery] string period = null,
            [FromQuery] string startDate = null,
            [FromQuery] string endDate = null)
        {
            try
            {
                // Проверка авторизации
                var token = Request.Headers["Authorization"].ToString().Replace("Bearer ", "");
                if (!await _authService.ValidateTokenAsync(token))
                {
                    return Unauthorized(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "UNAUTHORIZED",
                        Message = "Требуется авторизация"
                    });
                }

                // Проверка доступа к статистике группы
                if (!await _authService.HasAccessToGroupStatisticsAsync(token, groupId))
                {
                    return StatusCode(403, new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "ACCESS_DENIED",
                        Message = "Доступ к статистике запрещен"
                    });
                }

                // Валидация периода
                if (!string.IsNullOrEmpty(period))
                {
                    var validPeriods = new[] { "week", "month", "semester" };
                    if (!validPeriods.Contains(period))
                    {
                        return BadRequest(new ApiResponse<object>
                        {
                            Success = false,
                            ErrorCode = "INVALID_PERIOD",
                            Message = "Период должен быть: week, month или semester"
                        });
                    }
                }

                // Получение статистики
                var statistics = await _statisticsService.GetGroupStatisticsAsync(
                    groupId, period, startDate, endDate);

                if (statistics == null)
                {
                    return NotFound(new ApiResponse<object>
                    {
                        Success = false,
                        ErrorCode = "GROUP_NOT_FOUND",
                        Message = "Группа не найдена"
                    });
                }

                return Ok(new ApiResponse<GroupStatisticsResponse>
                {
                    Success = true,
                    Data = statistics
                });
            }
            catch (System.Exception ex)
            {
                _logger.LogError(ex, "Ошибка при получении статистики группы {GroupId}", groupId);
                return StatusCode(500, new ApiResponse<object>
                {
                    Success = false,
                    ErrorCode = "INTERNAL_SERVER_ERROR",
                    Message = "Внутренняя ошибка сервера"
                });
            }
        }
    }
}
```

## **ЮНИТ-ТЕСТЫ (xUnit + Moq + FluentAssertions):**

### **1. Создание тестового проекта:**
1. В Solution Explorer: правой кнопкой на Solution → **Добавить → Новый проект**
2. Выберите: **xUnit Test Project**
3. Название: `CollegeSystem.API.Tests`
4. Добавьте NuGet пакеты:
   ```
   Install-Package Moq
   Install-Package FluentAssertions
   Install-Package Microsoft.AspNetCore.Mvc.Testing
   Install-Package Microsoft.NET.Test.Sdk
   ```

### **2. StudentsControllerTests.cs**
```csharp
using Xunit;
using Moq;
using FluentAssertions;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using System.Collections.Generic;
using CollegeSystem.API.Controllers;
using CollegeSystem.API.Services;
using CollegeSystem.API.Models;

namespace CollegeSystem.API.Tests.Controllers
{
    public class StudentsControllerTests
    {
        private readonly Mock<IStudentService> _mockStudentService;
        private readonly Mock<IAuthorizationService> _mockAuthService;
        private readonly Mock<ILogger<StudentsController>> _mockLogger;
        private readonly StudentsController _controller;

        public StudentsControllerTests()
        {
            _mockStudentService = new Mock<IStudentService>();
            _mockAuthService = new Mock<IAuthorizationService>();
            _mockLogger = new Mock<ILogger<StudentsController>>();
            
            _controller = new StudentsController(
                _mockStudentService.Object,
                _mockAuthService.Object,
                _mockLogger.Object);
        }

        [Fact]
        public async Task GetGroupStudents_ValidRequest_ReturnsOkWithStudents()
        {
            // Arrange
            var groupId = "IT-21-1";
            var students = new List<StudentDto>
            {
                new StudentDto { StudentId = "ST001", FirstName = "Иван", LastName = "Иванов" }
            };
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToGroupAsync(It.IsAny<string>(), groupId))
                .ReturnsAsync(true);
            _mockStudentService.Setup(x => x.GetStudentsByGroupAsync(groupId))
                .ReturnsAsync(students);
            _mockStudentService.Setup(x => x.GetGroupCuratorAsync(groupId))
                .ReturnsAsync(new CuratorDto());

            // Act
            var result = await _controller.GetGroupStudents(groupId);

            // Assert
            result.Should().BeOfType<OkObjectResult>();
            var okResult = result as OkObjectResult;
            okResult.Value.Should().BeOfType<ApiResponse<GroupStudentsResponse>>();
            
            var response = okResult.Value as ApiResponse<GroupStudentsResponse>;
            response.Success.Should().BeTrue();
            response.Data.Students.Should().HaveCount(1);
            response.Data.GroupId.Should().Be(groupId);
        }

        [Fact]
        public async Task GetGroupStudents_InvalidToken_ReturnsUnauthorized()
        {
            // Arrange
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(false);

            // Act
            var result = await _controller.GetGroupStudents("IT-21-1");

            // Assert
            result.Should().BeOfType<UnauthorizedObjectResult>();
            var unauthorizedResult = result as UnauthorizedObjectResult;
            unauthorizedResult.Value.Should().BeOfType<ApiResponse<object>>();
            
            var response = unauthorizedResult.Value as ApiResponse<object>;
            response.Success.Should().BeFalse();
            response.ErrorCode.Should().Be("UNAUTHORIZED");
        }

        [Fact]
        public async Task GetGroupStudents_NoAccessToGroup_ReturnsForbidden()
        {
            // Arrange
            var groupId = "IT-21-1";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToGroupAsync(It.IsAny<string>(), groupId))
                .ReturnsAsync(false);

            // Act
            var result = await _controller.GetGroupStudents(groupId);

            // Assert
            result.Should().BeOfType<ObjectResult>();
            var objectResult = result as ObjectResult;
            objectResult.StatusCode.Should().Be(403);
            
            var response = objectResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("ACCESS_DENIED");
        }

        [Fact]
        public async Task GetGroupStudents_GroupNotFound_ReturnsNotFound()
        {
            // Arrange
            var groupId = "NONEXISTENT";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToGroupAsync(It.IsAny<string>(), groupId))
                .ReturnsAsync(true);
            _mockStudentService.Setup(x => x.GetStudentsByGroupAsync(groupId))
                .ReturnsAsync((List<StudentDto>)null);

            // Act
            var result = await _controller.GetGroupStudents(groupId);

            // Assert
            result.Should().BeOfType<NotFoundObjectResult>();
            var notFoundResult = result as NotFoundObjectResult;
            
            var response = notFoundResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("GROUP_NOT_FOUND");
        }

        [Fact]
        public async Task GetGroupStudents_ServiceException_ReturnsInternalServerError()
        {
            // Arrange
            var groupId = "IT-21-1";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToGroupAsync(It.IsAny<string>(), groupId))
                .ReturnsAsync(true);
            _mockStudentService.Setup(x => x.GetStudentsByGroupAsync(groupId))
                .ThrowsAsync(new System.Exception("Database error"));

            // Act
            var result = await _controller.GetGroupStudents(groupId);

            // Assert
            result.Should().BeOfType<ObjectResult>();
            var objectResult = result as ObjectResult;
            objectResult.StatusCode.Should().Be(500);
            
            var response = objectResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("INTERNAL_SERVER_ERROR");
        }
    }
}
```

### **3. AssignmentsControllerTests.cs**
```csharp
using Xunit;
using Moq;
using FluentAssertions;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using System;
using CollegeSystem.API.Controllers;
using CollegeSystem.API.Services;
using CollegeSystem.API.Models;

namespace CollegeSystem.API.Tests.Controllers
{
    public class AssignmentsControllerTests
    {
        private readonly Mock<IAssignmentService> _mockAssignmentService;
        private readonly Mock<IAuthorizationService> _mockAuthService;
        private readonly Mock<IValidationService> _mockValidationService;
        private readonly Mock<ILogger<AssignmentsController>> _mockLogger;
        private readonly AssignmentsController _controller;

        public AssignmentsControllerTests()
        {
            _mockAssignmentService = new Mock<IAssignmentService>();
            _mockAuthService = new Mock<IAuthorizationService>();
            _mockValidationService = new Mock<IValidationService>();
            _mockLogger = new Mock<ILogger<AssignmentsController>>();
            
            _controller = new AssignmentsController(
                _mockAssignmentService.Object,
                _mockAuthService.Object,
                _mockValidationService.Object,
                _mockLogger.Object);
        }

        [Fact]
        public async Task CreateAssignment_ValidRequest_ReturnsCreated()
        {
            // Arrange
            var request = new CreateAssignmentRequest
            {
                Title = "Test Assignment",
                SubjectId = "SUB001",
                TeacherId = "TC001",
                Deadline = DateTime.Now.AddDays(7)
            };
            
            var response = new AssignmentResponse
            {
                AssignmentId = "ASG001",
                Title = request.Title
            };
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.IsTeacherAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockValidationService.Setup(x => x.ValidateAssignmentRequest(request))
                .Returns((true, null));
            _mockAssignmentService.Setup(x => x.CreateAssignmentAsync(request))
                .ReturnsAsync(response);

            // Act
            var result = await _controller.CreateAssignment(request);

            // Assert
            result.Should().BeOfType<CreatedAtActionResult>();
            var createdResult = result as CreatedAtActionResult;
            createdResult.StatusCode.Should().Be(201);
            
            var apiResponse = createdResult.Value as ApiResponse<AssignmentResponse>;
            apiResponse.Success.Should().BeTrue();
            apiResponse.Data.AssignmentId.Should().Be("ASG001");
        }

        [Fact]
        public async Task CreateAssignment_NonTeacherUser_ReturnsForbidden()
        {
            // Arrange
            var request = new CreateAssignmentRequest();
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.IsTeacherAsync(It.IsAny<string>()))
                .ReturnsAsync(false);

            // Act
            var result = await _controller.CreateAssignment(request);

            // Assert
            result.Should().BeOfType<ObjectResult>();
            var objectResult = result as ObjectResult;
            objectResult.StatusCode.Should().Be(403);
            
            var response = objectResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("FORBIDDEN");
        }

        [Fact]
        public async Task CreateAssignment_InvalidRequest_ReturnsBadRequest()
        {
            // Arrange
            var request = new CreateAssignmentRequest();
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.IsTeacherAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockValidationService.Setup(x => x.ValidateAssignmentRequest(request))
                .Returns((false, "Title is required"));

            // Act
            var result = await _controller.CreateAssignment(request);

            // Assert
            result.Should().BeOfType<BadRequestObjectResult>();
            var badRequestResult = result as BadRequestObjectResult;
            
            var response = badRequestResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("INVALID_REQUEST");
        }

        [Fact]
        public async Task UpdateGrade_ValidRequest_ReturnsOk()
        {
            // Arrange
            var assignmentId = "ASG001";
            var request = new UpdateGradeRequest
            {
                StudentId = "ST001",
                Grade = 85,
                TeacherComment = "Good work"
            };
            
            var response = new GradeUpdateResponse
            {
                AssignmentId = assignmentId,
                Grade = request.Grade
            };
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.IsTeacherAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAssignmentService.Setup(x => x.UpdateGradeAsync(assignmentId, request))
                .ReturnsAsync(response);

            // Act
            var result = await _controller.UpdateGrade(assignmentId, request);

            // Assert
            result.Should().BeOfType<OkObjectResult>();
            var okResult = result as OkObjectResult;
            
            var apiResponse = okResult.Value as ApiResponse<GradeUpdateResponse>;
            apiResponse.Success.Should().BeTrue();
            apiResponse.Data.Grade.Should().Be(85);
        }

        [Theory]
        [InlineData(-10)]
        [InlineData(150)]
        public async Task UpdateGrade_InvalidGrade_ReturnsBadRequest(int invalidGrade)
        {
            // Arrange
            var assignmentId = "ASG001";
            var request = new UpdateGradeRequest
            {
                StudentId = "ST001",
                Grade = invalidGrade
            };
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.IsTeacherAsync(It.IsAny<string>()))
                .ReturnsAsync(true);

            // Act
            var result = await _controller.UpdateGrade(assignmentId, request);

            // Assert
            result.Should().BeOfType<BadRequestObjectResult>();
            var badRequestResult = result as BadRequestObjectResult;
            
            var response = badRequestResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("INVALID_GRADE_VALUE");
        }

        [Fact]
        public async Task UpdateGrade_AssignmentNotFound_ReturnsNotFound()
        {
            // Arrange
            var assignmentId = "NONEXISTENT";
            var request = new UpdateGradeRequest
            {
                StudentId = "ST001",
                Grade = 85
            };
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.IsTeacherAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAssignmentService.Setup(x => x.UpdateGradeAsync(assignmentId, request))
                .ThrowsAsync(new NotFoundException("ASSIGNMENT_NOT_FOUND", "Assignment not found"));

            // Act
            var result = await _controller.UpdateGrade(assignmentId, request);

            // Assert
            result.Should().BeOfType<NotFoundObjectResult>();
            var notFoundResult = result as NotFoundObjectResult;
            
            var response = notFoundResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("ASSIGNMENT_NOT_FOUND");
        }
    }
}
```

### **4. ScheduleControllerTests.cs**
```csharp
using Xunit;
using Moq;
using FluentAssertions;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using CollegeSystem.API.Controllers;
using CollegeSystem.API.Services;
using CollegeSystem.API.Models;

namespace CollegeSystem.API.Tests.Controllers
{
    public class ScheduleControllerTests
    {
        private readonly Mock<IScheduleService> _mockScheduleService;
        private readonly Mock<IAuthorizationService> _mockAuthService;
        private readonly Mock<ILogger<ScheduleController>> _mockLogger;
        private readonly ScheduleController _controller;

        public ScheduleControllerTests()
        {
            _mockScheduleService = new Mock<IScheduleService>();
            _mockAuthService = new Mock<IAuthorizationService>();
            _mockLogger = new Mock<ILogger<ScheduleController>>();
            
            _controller = new ScheduleController(
                _mockScheduleService.Object,
                _mockAuthService.Object,
                _mockLogger.Object);
        }

        [Fact]
        public async Task GetGroupSchedule_ValidRequest_ReturnsOk()
        {
            // Arrange
            var groupId = "IT-21-1";
            var week = "2024-12-09";
            var schedule = new ScheduleResponse();
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockScheduleService.Setup(x => x.ValidateDate(week))
                .Returns((true, null));
            _mockScheduleService.Setup(x => x.GetGroupScheduleAsync(groupId, week))
                .ReturnsAsync(schedule);

            // Act
            var result = await _controller.GetGroupSchedule(groupId, week);

            // Assert
            result.Should().BeOfType<OkObjectResult>();
            var okResult = result as OkObjectResult;
            
            var response = okResult.Value as ApiResponse<ScheduleResponse>;
            response.Success.Should().BeTrue();
        }

        [Fact]
        public async Task GetGroupSchedule_InvalidDateFormat_ReturnsBadRequest()
        {
            // Arrange
            var groupId = "IT-21-1";
            var invalidDate = "invalid-date";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockScheduleService.Setup(x => x.ValidateDate(invalidDate))
                .Returns((false, "Invalid date format"));

            // Act
            var result = await _controller.GetGroupSchedule(groupId, invalidDate);

            // Assert
            result.Should().BeOfType<BadRequestObjectResult>();
            var badRequestResult = result as BadRequestObjectResult;
            
            var response = badRequestResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("INVALID_DATE_FORMAT");
        }

        [Fact]
        public async Task GetGroupSchedule_ScheduleNotFound_ReturnsNotFound()
        {
            // Arrange
            var groupId = "IT-21-1";
            var week = "2024-12-09";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockScheduleService.Setup(x => x.ValidateDate(week))
                .Returns((true, null));
            _mockScheduleService.Setup(x => x.GetGroupScheduleAsync(groupId, week))
                .ReturnsAsync((ScheduleResponse)null);

            // Act
            var result = await _controller.GetGroupSchedule(groupId, week);

            // Assert
            result.Should().BeOfType<NotFoundObjectResult>();
            var notFoundResult = result as NotFoundObjectResult;
            
            var response = notFoundResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("SCHEDULE_NOT_FOUND");
        }

        [Fact]
        public async Task GetGroupSchedule_NoWeekDate_UsesCurrentWeek()
        {
            // Arrange
            var groupId = "IT-21-1";
            string week = null;
            var schedule = new ScheduleResponse();
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockScheduleService.Setup(x => x.GetGroupScheduleAsync(groupId, week))
                .ReturnsAsync(schedule);

            // Act
            var result = await _controller.GetGroupSchedule(groupId, week);

            // Assert
            result.Should().BeOfType<OkObjectResult>();
            _mockScheduleService.Verify(x => x.GetGroupScheduleAsync(groupId, null), Times.Once);
        }
    }
}
```

### **5. PerformanceControllerTests.cs**
```csharp
using Xunit;
using Moq;
using FluentAssertions;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using CollegeSystem.API.Controllers;
using CollegeSystem.API.Services;
using CollegeSystem.API.Models;

namespace CollegeSystem.API.Tests.Controllers
{
    public class PerformanceControllerTests
    {
        private readonly Mock<IPerformanceService> _mockPerformanceService;
        private readonly Mock<IAuthorizationService> _mockAuthService;
        private readonly Mock<ILogger<PerformanceController>> _mockLogger;
        private readonly PerformanceController _controller;

        public PerformanceControllerTests()
        {
            _mockPerformanceService = new Mock<IPerformanceService>();
            _mockAuthService = new Mock<IAuthorizationService>();
            _mockLogger = new Mock<ILogger<PerformanceController>>();
            
            _controller = new PerformanceController(
                _mockPerformanceService.Object,
                _mockAuthService.Object,
                _mockLogger.Object);
        }

        [Fact]
        public async Task GetStudentPerformance_ValidRequest_ReturnsOk()
        {
            // Arrange
            var studentId = "ST001";
            var performance = new StudentPerformanceResponse();
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToStudentAsync(It.IsAny<string>(), studentId))
                .ReturnsAsync(true);
            _mockPerformanceService.Setup(x => x.GetStudentPerformanceAsync(studentId, null, null))
                .ReturnsAsync(performance);

            // Act
            var result = await _controller.GetStudentPerformance(studentId);

            // Assert
            result.Should().BeOfType<OkObjectResult>();
            var okResult = result as OkObjectResult;
            
            var response = okResult.Value as ApiResponse<StudentPerformanceResponse>;
            response.Success.Should().BeTrue();
        }

        [Fact]
        public async Task GetStudentPerformance_NoAccessToStudent_ReturnsForbidden()
        {
            // Arrange
            var studentId = "ST001";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToStudentAsync(It.IsAny<string>(), studentId))
                .ReturnsAsync(false);

            // Act
            var result = await _controller.GetStudentPerformance(studentId);

            // Assert
            result.Should().BeOfType<ObjectResult>();
            var objectResult = result as ObjectResult;
            objectResult.StatusCode.Should().Be(403);
            
            var response = objectResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("ACCESS_DENIED");
        }

        [Theory]
        [InlineData(0)]
        [InlineData(3)]
        public async Task GetStudentPerformance_InvalidSemester_ReturnsBadRequest(int invalidSemester)
        {
            // Arrange
            var studentId = "ST001";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToStudentAsync(It.IsAny<string>(), studentId))
                .ReturnsAsync(true);

            // Act
            var result = await _controller.GetStudentPerformance(studentId, invalidSemester);

            // Assert
            result.Should().BeOfType<BadRequestObjectResult>();
            var badRequestResult = result as BadRequestObjectResult;
            
            var response = badRequestResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("INVALID_SEMESTER");
        }

        [Fact]
        public async Task GetStudentPerformance_StudentNotFound_ReturnsNotFound()
        {
            // Arrange
            var studentId = "NONEXISTENT";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToStudentAsync(It.IsAny<string>(), studentId))
                .ReturnsAsync(true);
            _mockPerformanceService.Setup(x => x.GetStudentPerformanceAsync(studentId, null, null))
                .ReturnsAsync((StudentPerformanceResponse)null);

            // Act
            var result = await _controller.GetStudentPerformance(studentId);

            // Assert
            result.Should().BeOfType<NotFoundObjectResult>();
            var notFoundResult = result as NotFoundObjectResult;
            
            var response = notFoundResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("STUDENT_NOT_FOUND");
        }
    }
}
```

### **6. StatisticsControllerTests.cs**
```csharp
using Xunit;
using Moq;
using FluentAssertions;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using CollegeSystem.API.Controllers;
using CollegeSystem.API.Services;
using CollegeSystem.API.Models;
using System.Linq;

namespace CollegeSystem.API.Tests.Controllers
{
    public class StatisticsControllerTests
    {
        private readonly Mock<IStatisticsService> _mockStatisticsService;
        private readonly Mock<IAuthorizationService> _mockAuthService;
        private readonly Mock<ILogger<StatisticsController>> _mockLogger;
        private readonly StatisticsController _controller;

        public StatisticsControllerTests()
        {
            _mockStatisticsService = new Mock<IStatisticsService>();
            _mockAuthService = new Mock<IAuthorizationService>();
            _mockLogger = new Mock<ILogger<StatisticsController>>();
            
            _controller = new StatisticsController(
                _mockStatisticsService.Object,
                _mockAuthService.Object,
                _mockLogger.Object);
        }

        [Fact]
        public async Task GetGroupStatistics_ValidRequest_ReturnsOk()
        {
            // Arrange
            var groupId = "IT-21-1";
            var statistics = new GroupStatisticsResponse();
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToGroupStatisticsAsync(It.IsAny<string>(), groupId))
                .ReturnsAsync(true);
            _mockStatisticsService.Setup(x => x.GetGroupStatisticsAsync(groupId, null, null, null))
                .ReturnsAsync(statistics);

            // Act
            var result = await _controller.GetGroupStatistics(groupId);

            // Assert
            result.Should().BeOfType<OkObjectResult>();
            var okResult = result as OkObjectResult;
            
            var response = okResult.Value as ApiResponse<GroupStatisticsResponse>;
            response.Success.Should().BeTrue();
        }

        [Fact]
        public async Task GetGroupStatistics_NoAccessToStatistics_ReturnsForbidden()
        {
            // Arrange
            var groupId = "IT-21-1";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToGroupStatisticsAsync(It.IsAny<string>(), groupId))
                .ReturnsAsync(false);

            // Act
            var result = await _controller.GetGroupStatistics(groupId);

            // Assert
            result.Should().BeOfType<ObjectResult>();
            var objectResult = result as ObjectResult;
            objectResult.StatusCode.Should().Be(403);
            
            var response = objectResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("ACCESS_DENIED");
        }

        [Theory]
        [InlineData("invalid")]
        [InlineData("year")]
        public async Task GetGroupStatistics_InvalidPeriod_ReturnsBadRequest(string invalidPeriod)
        {
            // Arrange
            var groupId = "IT-21-1";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToGroupStatisticsAsync(It.IsAny<string>(), groupId))
                .ReturnsAsync(true);

            // Act
            var result = await _controller.GetGroupStatistics(groupId, invalidPeriod);

            // Assert
            result.Should().BeOfType<BadRequestObjectResult>();
            var badRequestResult = result as BadRequestObjectResult;
            
            var response = badRequestResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("INVALID_PERIOD");
        }

        [Theory]
        [InlineData("week")]
        [InlineData("month")]
        [InlineData("semester")]
        public async Task GetGroupStatistics_ValidPeriod_ReturnsOk(string validPeriod)
        {
            // Arrange
            var groupId = "IT-21-1";
            var statistics = new GroupStatisticsResponse();
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToGroupStatisticsAsync(It.IsAny<string>(), groupId))
                .ReturnsAsync(true);
            _mockStatisticsService.Setup(x => x.GetGroupStatisticsAsync(groupId, validPeriod, null, null))
                .ReturnsAsync(statistics);

            // Act
            var result = await _controller.GetGroupStatistics(groupId, validPeriod);

            // Assert
            result.Should().BeOfType<OkObjectResult>();
        }

        [Fact]
        public async Task GetGroupStatistics_GroupNotFound_ReturnsNotFound()
        {
            // Arrange
            var groupId = "NONEXISTENT";
            
            _mockAuthService.Setup(x => x.ValidateTokenAsync(It.IsAny<string>()))
                .ReturnsAsync(true);
            _mockAuthService.Setup(x => x.HasAccessToGroupStatisticsAsync(It.IsAny<string>(), groupId))
                .ReturnsAsync(true);
            _mockStatisticsService.Setup(x => x.GetGroupStatisticsAsync(groupId, null, null, null))
                .ReturnsAsync((GroupStatisticsResponse)null);

            // Act
            var result = await _controller.GetGroupStatistics(groupId);

            // Assert
            result.Should().BeOfType<NotFoundObjectResult>();
            var notFoundResult = result as NotFoundObjectResult;
            
            var response = notFoundResult.Value as ApiResponse<object>;
            response.ErrorCode.Should().Be("GROUP_NOT_FOUND");
        }
    }
}
```

## **ВАЖНЫЕ ШАГИ В VISUAL STUDIO:**

### **1. Установка NuGet пакетов для тестового проекта:**
```xml
<!-- CollegeSystem.API.Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.3.2" />
    <PackageReference Include="xunit" Version="2.4.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.4.5">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="coverlet.collector" Version="3.1.2">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Moq" Version="4.18.4" />
    <PackageReference Include="FluentAssertions" Version="6.11.0" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="6.0.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\CollegeSystem.API\CollegeSystem.API.csproj" />
  </ItemGroup>
</Project>
```

### **2. Запуск тестов:**
1. Откройте **Test Explorer** (Тест → Обозреватель тестов)
2. Соберите решение (Build → Build Solution)
3. Все тесты появятся в Test Explorer
4. Нажмите **Run All** для запуска всех тестов

### **3. Структура моделей (пример):**
```csharp
// Models/ApiResponse.cs
namespace CollegeSystem.API.Models
{
    public class ApiResponse<T>
    {
        public bool Success { get; set; }
        public T Data { get; set; }
        public string ErrorCode { get; set; }
        public string Message { get; set; }
    }
}
```

### **4. Настройка Program.cs:**
```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Register your services
builder.Services.AddScoped<IStudentService, StudentService>();
builder.Services.AddScoped<IAssignmentService, AssignmentService>();
builder.Services.AddScoped<IAuthorizationService, AuthorizationService>();
// ... other services

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```


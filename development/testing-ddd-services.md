# Testing Strategies for DDD Services

## Overview

Comprehensive testing is essential for Domain-Driven Design architecture. This guide covers testing strategies for each layer of the architecture, from unit tests for domain logic to end-to-end integration tests.

## Testing Pyramid

```
                    E2E Tests (Few)
                   ▼              ▼
               Integration Tests (Some)
              ▼                        ▼
           Unit Tests (Many)
          ▼                            ▼
     Domain Tests                API Tests
```

## Domain Layer Testing

### 1. Entity Testing

Test core business logic and state transitions.

```python
# tests/unit/domain/entities/test_analysis.py
import pytest
from datetime import datetime, timedelta
from src.domain.entities.analysis import Analysis
from src.domain.value_objects.analysis_status import AnalysisStatus

class TestAnalysisEntity:
    def test_create_new_analysis(self):
        """Test creating a new analysis entity"""
        analysis = Analysis.create_new(
            project_id=1,
            contract_name="TestContract",
            contract_source="contract Test {}",
            compiler_version="0.8.19",
            submitted_by=1
        )

        assert analysis.project_id == 1
        assert analysis.contract_name == "TestContract"
        assert analysis.status == AnalysisStatus.SUBMITTED
        assert analysis.submitted_by == 1
        assert analysis.created_at is not None
        assert analysis.completed_at is None

    def test_start_parsing_transitions_status(self):
        """Test that starting parsing changes status correctly"""
        analysis = Analysis.create_new(
            project_id=1,
            contract_name="Test",
            contract_source="contract Test {}",
            compiler_version="0.8.19",
            submitted_by=1
        )

        analysis.start_parsing()

        assert analysis.status == AnalysisStatus.PARSING
        assert analysis.parsing_started_at is not None

    def test_complete_parsing_with_results(self):
        """Test completing parsing with successful results"""
        analysis = Analysis.create_new(
            project_id=1,
            contract_name="Test",
            contract_source="contract Test {}",
            compiler_version="0.8.19",
            submitted_by=1
        )

        analysis.start_parsing()
        ast_data = {"type": "SourceUnit", "children": []}
        bytecode = "0x608060405234801561001057600080fd5b50"

        analysis.complete_parsing(ast_data, bytecode)

        assert analysis.status == AnalysisStatus.PARSED
        assert analysis.ast_data == ast_data
        assert analysis.bytecode == bytecode
        assert analysis.parsing_completed_at is not None

    def test_fail_parsing_with_error(self):
        """Test failing parsing with error message"""
        analysis = Analysis.create_new(
            project_id=1,
            contract_name="Test",
            contract_source="invalid contract",
            compiler_version="0.8.19",
            submitted_by=1
        )

        analysis.start_parsing()
        error_message = "Syntax error at line 1"

        analysis.fail_parsing(error_message)

        assert analysis.status == AnalysisStatus.FAILED
        assert analysis.error_message == error_message
        assert analysis.parsing_failed_at is not None

    def test_cannot_start_analysis_when_failed(self):
        """Test business rule: cannot start analysis on failed parsing"""
        analysis = Analysis.create_new(
            project_id=1,
            contract_name="Test",
            contract_source="invalid contract",
            compiler_version="0.8.19",
            submitted_by=1
        )

        analysis.start_parsing()
        analysis.fail_parsing("Parse error")

        with pytest.raises(ValueError, match="Cannot start analysis on failed parsing"):
            analysis.start_analysis()

    def test_analysis_duration_calculation(self):
        """Test analysis duration calculation"""
        analysis = Analysis.create_new(
            project_id=1,
            contract_name="Test",
            contract_source="contract Test {}",
            compiler_version="0.8.19",
            submitted_by=1
        )

        # Manually set timestamps for testing
        start_time = datetime.utcnow()
        analysis._created_at = start_time
        analysis._completed_at = start_time + timedelta(minutes=5)

        duration = analysis.analysis_duration_seconds()
        assert duration == 300  # 5 minutes
```

### 2. Value Object Testing

Test immutability and validation rules.

```python
# tests/unit/domain/value_objects/test_email.py
import pytest
from src.domain.value_objects.email import Email

class TestEmailValueObject:
    def test_valid_email_creation(self):
        """Test creating valid email addresses"""
        email = Email("user@example.com")
        assert email.value == "user@example.com"

    def test_invalid_email_raises_error(self):
        """Test that invalid emails raise ValueError"""
        with pytest.raises(ValueError, match="Invalid email format"):
            Email("invalid-email")

        with pytest.raises(ValueError, match="Invalid email format"):
            Email("user@")

        with pytest.raises(ValueError, match="Invalid email format"):
            Email("@example.com")

    def test_email_domain_extraction(self):
        """Test domain extraction from email"""
        email = Email("user@example.com")
        assert email.domain() == "example.com"

    def test_email_immutability(self):
        """Test that email value objects are immutable"""
        email = Email("user@example.com")

        # Should not be able to modify the value
        with pytest.raises(AttributeError):
            email.value = "different@example.com"

    def test_email_equality(self):
        """Test email equality comparison"""
        email1 = Email("user@example.com")
        email2 = Email("user@example.com")
        email3 = Email("other@example.com")

        assert email1 == email2
        assert email1 != email3
        assert hash(email1) == hash(email2)
```

### 3. Domain Service Testing

Test complex business rules that span entities.

```python
# tests/unit/domain/services/test_analysis_workflow_service.py
import pytest
from unittest.mock import Mock
from src.domain.services.analysis_workflow_service import AnalysisWorkflowService
from src.domain.entities.analysis import Analysis
from src.domain.entities.project import Project
from src.domain.value_objects.analysis_status import AnalysisStatus

class TestAnalysisWorkflowService:
    @pytest.fixture
    def workflow_service(self):
        return AnalysisWorkflowService()

    def test_can_start_analysis_when_parsed(self, workflow_service):
        """Test that analysis can start when parsing is complete"""
        analysis = Mock(spec=Analysis)
        analysis.status = AnalysisStatus.PARSED
        analysis.ast_data = {"type": "SourceUnit"}
        analysis.bytecode = "0x123"

        assert workflow_service.can_start_analysis(analysis) is True

    def test_cannot_start_analysis_when_not_parsed(self, workflow_service):
        """Test that analysis cannot start before parsing"""
        analysis = Mock(spec=Analysis)
        analysis.status = AnalysisStatus.SUBMITTED

        assert workflow_service.can_start_analysis(analysis) is False

    def test_calculate_analysis_priority(self, workflow_service):
        """Test analysis priority calculation"""
        project = Mock(spec=Project)
        project.security_level = "HIGH"
        project.created_at = Mock()

        analysis = Mock(spec=Analysis)
        analysis.submitted_at = Mock()

        priority = workflow_service.calculate_analysis_priority(project, analysis)
        assert isinstance(priority, int)
        assert priority > 0

    def test_estimate_completion_time(self, workflow_service):
        """Test analysis completion time estimation"""
        analysis = Mock(spec=Analysis)
        analysis.contract_source = "contract Test { " + "function test() {}" * 100 + " }"

        estimated_minutes = workflow_service.estimate_completion_time(analysis)
        assert estimated_minutes > 0
        assert isinstance(estimated_minutes, int)
```

## Application Layer Testing

### 1. Command Handler Testing

Test use case orchestration and business logic.

```python
# tests/unit/application/handlers/test_submit_contract_handler.py
import pytest
from unittest.mock import AsyncMock, Mock, patch
from src.application.analysis.commands.submit_contract_command import SubmitContractCommand
from src.application.analysis.handlers.submit_contract_handler import SubmitContractCommandHandler
from src.domain.entities.analysis import Analysis
from src.domain.entities.project import Project

class TestSubmitContractCommandHandler:
    @pytest.fixture
    def mock_dependencies(self):
        return {
            'project_repository': AsyncMock(),
            'analysis_repository': AsyncMock(),
            'workflow_service': Mock(),
            'parser_client': AsyncMock()
        }

    @pytest.fixture
    def handler(self, mock_dependencies):
        return SubmitContractCommandHandler(**mock_dependencies)

    @pytest.fixture
    def valid_command(self):
        return SubmitContractCommand(
            project_id=1,
            contract_source="contract Test { function test() {} }",
            contract_name="TestContract",
            compiler_version="0.8.19",
            optimization_enabled=True,
            optimization_runs=200,
            user_id=1
        )

    @pytest.mark.asyncio
    async def test_successful_contract_submission(self, handler, valid_command, mock_dependencies):
        """Test successful contract submission flow"""
        # Arrange
        mock_project = Mock(spec=Project)
        mock_project.user_can_submit_contracts.return_value = True
        mock_dependencies['project_repository'].find_by_id.return_value = mock_project

        mock_analysis = Mock(spec=Analysis)
        mock_dependencies['analysis_repository'].save.return_value = mock_analysis

        mock_dependencies['workflow_service'].can_start_analysis.return_value = True

        # Act
        result = await handler.handle(valid_command)

        # Assert
        assert result == mock_analysis
        mock_dependencies['project_repository'].find_by_id.assert_called_once_with(1)
        mock_dependencies['analysis_repository'].save.assert_called_once()
        mock_project.user_can_submit_contracts.assert_called_once_with(1)

    @pytest.mark.asyncio
    async def test_project_not_found(self, handler, valid_command, mock_dependencies):
        """Test handling when project doesn't exist"""
        # Arrange
        mock_dependencies['project_repository'].find_by_id.return_value = None

        # Act & Assert
        with pytest.raises(ValueError, match="Project 1 not found"):
            await handler.handle(valid_command)

    @pytest.mark.asyncio
    async def test_permission_denied(self, handler, valid_command, mock_dependencies):
        """Test handling when user lacks permissions"""
        # Arrange
        mock_project = Mock(spec=Project)
        mock_project.user_can_submit_contracts.return_value = False
        mock_dependencies['project_repository'].find_by_id.return_value = mock_project

        # Act & Assert
        with pytest.raises(PermissionError, match="User cannot submit contracts"):
            await handler.handle(valid_command)

    @pytest.mark.asyncio
    async def test_workflow_validation_failure(self, handler, valid_command, mock_dependencies):
        """Test handling when workflow validation fails"""
        # Arrange
        mock_project = Mock(spec=Project)
        mock_project.user_can_submit_contracts.return_value = True
        mock_dependencies['project_repository'].find_by_id.return_value = mock_project

        mock_dependencies['workflow_service'].can_start_analysis.return_value = False

        # Act & Assert
        with pytest.raises(ValueError, match="Analysis cannot be started"):
            await handler.handle(valid_command)

    @pytest.mark.asyncio
    async def test_parsing_workflow_started(self, handler, valid_command, mock_dependencies):
        """Test that parsing workflow is started asynchronously"""
        # Arrange
        mock_project = Mock(spec=Project)
        mock_project.user_can_submit_contracts.return_value = True
        mock_dependencies['project_repository'].find_by_id.return_value = mock_project

        mock_analysis = Mock(spec=Analysis)
        mock_dependencies['analysis_repository'].save.return_value = mock_analysis

        mock_dependencies['workflow_service'].can_start_analysis.return_value = True

        # Mock the async parsing method
        with patch.object(handler, '_start_parsing_workflow') as mock_start_parsing:
            # Act
            await handler.handle(valid_command)

            # Assert
            mock_start_parsing.assert_called_once_with(mock_analysis, valid_command)
```

### 2. Query Handler Testing

Test data retrieval and filtering logic.

```python
# tests/unit/application/handlers/test_analysis_query_handler.py
import pytest
from unittest.mock import AsyncMock, Mock
from src.application.analysis.handlers.analysis_query_handler import AnalysisQueryHandler
from src.application.analysis.queries.get_analysis_results_query import GetAnalysisResultsQuery

class TestAnalysisQueryHandler:
    @pytest.fixture
    def mock_dependencies(self):
        return {
            'analysis_repository': AsyncMock(),
            'project_repository': AsyncMock()
        }

    @pytest.fixture
    def handler(self, mock_dependencies):
        return AnalysisQueryHandler(**mock_dependencies)

    @pytest.mark.asyncio
    async def test_get_analysis_results_success(self, handler, mock_dependencies):
        """Test successful analysis results retrieval"""
        # Arrange
        query = GetAnalysisResultsQuery(analysis_id=1, user_id=1)

        mock_analysis = Mock()
        mock_analysis.project_id = 1
        mock_dependencies['analysis_repository'].find_by_id.return_value = mock_analysis

        mock_project = Mock()
        mock_project.user_can_view_results.return_value = True
        mock_dependencies['project_repository'].find_by_id.return_value = mock_project

        # Act
        result = await handler.handle(query)

        # Assert
        assert result == mock_analysis
        mock_dependencies['analysis_repository'].find_by_id.assert_called_once_with(1)
        mock_dependencies['project_repository'].find_by_id.assert_called_once_with(1)

    @pytest.mark.asyncio
    async def test_analysis_not_found(self, handler, mock_dependencies):
        """Test handling when analysis doesn't exist"""
        # Arrange
        query = GetAnalysisResultsQuery(analysis_id=999, user_id=1)
        mock_dependencies['analysis_repository'].find_by_id.return_value = None

        # Act
        result = await handler.handle(query)

        # Assert
        assert result is None

    @pytest.mark.asyncio
    async def test_permission_denied_for_results(self, handler, mock_dependencies):
        """Test permission denial for viewing results"""
        # Arrange
        query = GetAnalysisResultsQuery(analysis_id=1, user_id=2)

        mock_analysis = Mock()
        mock_analysis.project_id = 1
        mock_dependencies['analysis_repository'].find_by_id.return_value = mock_analysis

        mock_project = Mock()
        mock_project.user_can_view_results.return_value = False
        mock_dependencies['project_repository'].find_by_id.return_value = mock_project

        # Act & Assert
        with pytest.raises(PermissionError):
            await handler.handle(query)
```

## Infrastructure Layer Testing

### 1. Repository Implementation Testing

Test data persistence and retrieval.

```python
# tests/integration/infrastructure/repositories/test_analysis_repository.py
import pytest
from sqlalchemy.orm import Session
from src.infrastructure.database.repositories.analysis_repository_impl import SqlAlchemyAnalysisRepository
from src.infrastructure.database.models.analysis_model import AnalysisModel
from src.domain.entities.analysis import Analysis
from src.domain.value_objects.analysis_status import AnalysisStatus

class TestSqlAlchemyAnalysisRepository:
    @pytest.fixture
    async def repository(self, db_session: Session):
        return SqlAlchemyAnalysisRepository(db_session)

    @pytest.mark.asyncio
    async def test_save_new_analysis(self, repository, db_session):
        """Test saving a new analysis entity"""
        # Arrange
        analysis = Analysis.create_new(
            project_id=1,
            contract_name="TestContract",
            contract_source="contract Test {}",
            compiler_version="0.8.19",
            submitted_by=1
        )

        # Act
        saved_analysis = await repository.save(analysis)

        # Assert
        assert saved_analysis.id is not None
        assert saved_analysis.contract_name == "TestContract"
        assert saved_analysis.status == AnalysisStatus.SUBMITTED

        # Verify in database
        db_analysis = db_session.query(AnalysisModel).filter_by(id=saved_analysis.id).first()
        assert db_analysis is not None
        assert db_analysis.contract_name == "TestContract"

    @pytest.mark.asyncio
    async def test_find_by_id(self, repository, db_session):
        """Test finding analysis by ID"""
        # Arrange - create analysis in database
        db_analysis = AnalysisModel(
            project_id=1,
            contract_name="TestContract",
            contract_source="contract Test {}",
            compiler_version="0.8.19",
            status="SUBMITTED",
            submitted_by=1
        )
        db_session.add(db_analysis)
        await db_session.commit()

        # Act
        found_analysis = await repository.find_by_id(db_analysis.id)

        # Assert
        assert found_analysis is not None
        assert found_analysis.id == db_analysis.id
        assert found_analysis.contract_name == "TestContract"

    @pytest.mark.asyncio
    async def test_find_by_project_id(self, repository, db_session):
        """Test finding analyses by project ID"""
        # Arrange - create multiple analyses
        for i in range(3):
            db_analysis = AnalysisModel(
                project_id=1,
                contract_name=f"Contract{i}",
                contract_source=f"contract Test{i} {{}}",
                compiler_version="0.8.19",
                status="SUBMITTED",
                submitted_by=1
            )
            db_session.add(db_analysis)

        await db_session.commit()

        # Act
        analyses = await repository.find_by_project_id(project_id=1)

        # Assert
        assert len(analyses) == 3
        for analysis in analyses:
            assert analysis.project_id == 1

    @pytest.mark.asyncio
    async def test_update_existing_analysis(self, repository, db_session):
        """Test updating an existing analysis"""
        # Arrange
        analysis = Analysis.create_new(
            project_id=1,
            contract_name="TestContract",
            contract_source="contract Test {}",
            compiler_version="0.8.19",
            submitted_by=1
        )
        saved_analysis = await repository.save(analysis)

        # Modify the analysis
        saved_analysis.start_parsing()

        # Act
        updated_analysis = await repository.save(saved_analysis)

        # Assert
        assert updated_analysis.status == AnalysisStatus.PARSING
        assert updated_analysis.parsing_started_at is not None
```

### 2. External Service Client Testing

Test HTTP client integrations with mocking.

```python
# tests/unit/infrastructure/external/test_contract_parser_client.py
import pytest
from unittest.mock import Mock, patch
import aiohttp
from src.infrastructure.external.contract_parser_client import ContractParserClient

class TestContractParserClient:
    @pytest.fixture
    def client(self):
        return ContractParserClient(base_url="http://parser:8080")

    @pytest.mark.asyncio
    async def test_parse_contract_success(self, client):
        """Test successful contract parsing"""
        # Arrange
        mock_response_data = {
            "success": True,
            "ast": {"type": "SourceUnit", "children": []},
            "bytecode": "0x608060405234801561001057600080fd5b50",
            "warnings": []
        }

        with patch('aiohttp.ClientSession.post') as mock_post:
            mock_response = Mock()
            mock_response.status = 200
            mock_response.json.return_value = mock_response_data
            mock_post.return_value.__aenter__.return_value = mock_response

            # Act
            result = await client.parse_contract(
                contract_source="contract Test {}",
                compiler_version="0.8.19",
                optimization_enabled=True,
                optimization_runs=200
            )

            # Assert
            assert result.success is True
            assert result.ast == mock_response_data["ast"]
            assert result.bytecode == mock_response_data["bytecode"]

    @pytest.mark.asyncio
    async def test_parse_contract_failure(self, client):
        """Test contract parsing failure"""
        # Arrange
        mock_response_data = {
            "success": False,
            "error_message": "Syntax error at line 1",
            "error_details": {
                "line": 1,
                "column": 10,
                "message": "Expected ';'"
            }
        }

        with patch('aiohttp.ClientSession.post') as mock_post:
            mock_response = Mock()
            mock_response.status = 400
            mock_response.json.return_value = mock_response_data
            mock_post.return_value.__aenter__.return_value = mock_response

            # Act
            result = await client.parse_contract(
                contract_source="invalid contract",
                compiler_version="0.8.19"
            )

            # Assert
            assert result.success is False
            assert result.error_message == "Syntax error at line 1"

    @pytest.mark.asyncio
    async def test_network_error_handling(self, client):
        """Test handling of network errors"""
        with patch('aiohttp.ClientSession.post') as mock_post:
            mock_post.side_effect = aiohttp.ClientError("Connection failed")

            # Act & Assert
            with pytest.raises(aiohttp.ClientError):
                await client.parse_contract(
                    contract_source="contract Test {}",
                    compiler_version="0.8.19"
                )
```

## Presentation Layer Testing

### 1. API Endpoint Testing

Test HTTP endpoints with FastAPI TestClient.

```python
# tests/integration/presentation/test_analysis_endpoints.py
import pytest
from fastapi.testclient import TestClient
from unittest.mock import AsyncMock, patch
from src.main import app

class TestAnalysisEndpoints:
    @pytest.fixture
    def client(self):
        return TestClient(app)

    @pytest.fixture
    def auth_headers(self):
        # Mock JWT token for authentication
        return {"Authorization": "Bearer mock-jwt-token"}

    def test_submit_contract_success(self, client, auth_headers):
        """Test successful contract submission"""
        # Arrange
        request_data = {
            "project_id": 1,
            "contract_source": "contract Test { function test() {} }",
            "contract_name": "TestContract",
            "compiler_version": "0.8.19",
            "optimization_enabled": True,
            "optimization_runs": 200
        }

        with patch('src.presentation.api.v1.analysis.router.SubmitContractCommandHandler') as mock_handler_class:
            mock_handler = AsyncMock()
            mock_handler.handle.return_value = Mock(
                id=1,
                project_id=1,
                contract_name="TestContract",
                status="SUBMITTED"
            )
            mock_handler_class.return_value = mock_handler

            # Act
            response = client.post(
                "/api/v1/analysis/submit",
                json=request_data,
                headers=auth_headers
            )

            # Assert
            assert response.status_code == 201
            assert response.json()["id"] == 1
            assert response.json()["status"] == "SUBMITTED"

    def test_submit_contract_validation_error(self, client, auth_headers):
        """Test contract submission with invalid data"""
        # Arrange
        request_data = {
            "project_id": 1,
            "contract_source": "",  # Empty source should fail validation
            "contract_name": "TestContract",
            "compiler_version": "0.8.19"
        }

        # Act
        response = client.post(
            "/api/v1/analysis/submit",
            json=request_data,
            headers=auth_headers
        )

        # Assert
        assert response.status_code == 422
        assert "contract_source" in response.json()["detail"][0]["loc"]

    def test_get_analysis_results(self, client, auth_headers):
        """Test retrieving analysis results"""
        with patch('src.presentation.api.v1.analysis.router.AnalysisQueryHandler') as mock_handler_class:
            mock_handler = AsyncMock()
            mock_handler.handle.return_value = Mock(
                id=1,
                project_id=1,
                contract_name="TestContract",
                status="COMPLETED",
                vulnerabilities=[
                    {
                        "severity": "HIGH",
                        "type": "Reentrancy",
                        "description": "Potential reentrancy vulnerability"
                    }
                ]
            )
            mock_handler_class.return_value = mock_handler

            # Act
            response = client.get(
                "/api/v1/analysis/1/results",
                headers=auth_headers
            )

            # Assert
            assert response.status_code == 200
            assert response.json()["id"] == 1
            assert len(response.json()["vulnerabilities"]) == 1

    def test_unauthorized_access(self, client):
        """Test unauthorized access without JWT token"""
        # Act
        response = client.get("/api/v1/analysis/1/results")

        # Assert
        assert response.status_code == 401
```

## Integration Testing

### 1. Database Integration Tests

Test complete workflows with real database.

```python
# tests/integration/test_analysis_workflow.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from src.infrastructure.database.models.base_model import BaseModel
from src.application.analysis.commands.submit_contract_command import SubmitContractCommand
from src.application.analysis.handlers.submit_contract_handler import SubmitContractCommandHandler
from src.infrastructure.database.repositories.analysis_repository_impl import SqlAlchemyAnalysisRepository
from src.infrastructure.database.repositories.project_repository_impl import SqlAlchemyProjectRepository

@pytest.mark.integration
class TestAnalysisWorkflowIntegration:
    @pytest.fixture(scope="function")
    async def db_session(self):
        """Create test database session"""
        engine = create_engine("sqlite:///:memory:")
        BaseModel.metadata.create_all(engine)
        SessionLocal = sessionmaker(bind=engine)
        session = SessionLocal()
        try:
            yield session
        finally:
            session.close()

    @pytest.mark.asyncio
    async def test_complete_analysis_submission_workflow(self, db_session):
        """Test complete workflow from command to database persistence"""
        # Arrange
        analysis_repo = SqlAlchemyAnalysisRepository(db_session)
        project_repo = SqlAlchemyProjectRepository(db_session)

        # Create test project first
        project = await self._create_test_project(project_repo)

        # Mock external dependencies
        workflow_service = Mock()
        workflow_service.can_start_analysis.return_value = True

        parser_client = AsyncMock()

        handler = SubmitContractCommandHandler(
            project_repository=project_repo,
            analysis_repository=analysis_repo,
            workflow_service=workflow_service,
            parser_client=parser_client
        )

        command = SubmitContractCommand(
            project_id=project.id,
            contract_source="contract Test { function test() {} }",
            contract_name="TestContract",
            compiler_version="0.8.19",
            user_id=1
        )

        # Act
        result = await handler.handle(command)

        # Assert
        assert result.id is not None
        assert result.contract_name == "TestContract"

        # Verify persistence
        saved_analysis = await analysis_repo.find_by_id(result.id)
        assert saved_analysis is not None
        assert saved_analysis.contract_name == "TestContract"
```

### 2. End-to-End API Tests

Test complete API workflows.

```python
# tests/e2e/test_analysis_api.py
import pytest
import asyncio
from fastapi.testclient import TestClient
from src.main import app

@pytest.mark.e2e
class TestAnalysisAPIE2E:
    @pytest.fixture
    def client(self):
        return TestClient(app)

    def test_complete_analysis_workflow(self, client):
        """Test complete analysis workflow end-to-end"""
        # 1. Create user and get token
        register_response = client.post("/api/v1/auth/register", json={
            "email": "test@example.com",
            "username": "testuser",
            "password": "StrongPassword123",
            "full_name": "Test User"
        })
        assert register_response.status_code == 201

        login_response = client.post("/api/v1/auth/login", json={
            "email": "test@example.com",
            "password": "StrongPassword123"
        })
        assert login_response.status_code == 200
        token = login_response.json()["access_token"]
        headers = {"Authorization": f"Bearer {token}"}

        # 2. Create project
        project_response = client.post("/api/v1/projects", json={
            "name": "Test Project",
            "description": "Test project for E2E testing"
        }, headers=headers)
        assert project_response.status_code == 201
        project_id = project_response.json()["id"]

        # 3. Submit contract for analysis
        submit_response = client.post("/api/v1/analysis/submit", json={
            "project_id": project_id,
            "contract_source": "contract Test { function test() {} }",
            "contract_name": "TestContract",
            "compiler_version": "0.8.19"
        }, headers=headers)
        assert submit_response.status_code == 201
        analysis_id = submit_response.json()["id"]

        # 4. Check analysis status
        status_response = client.get(f"/api/v1/analysis/{analysis_id}", headers=headers)
        assert status_response.status_code == 200
        assert status_response.json()["status"] in ["SUBMITTED", "PARSING"]

        # 5. Wait for completion (in real test, would use async polling)
        # For E2E test, we might mock the completion or use test doubles
```

## Test Configuration and Fixtures

### 1. Pytest Configuration

```python
# conftest.py
import pytest
import asyncio
from unittest.mock import Mock
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from src.infrastructure.database.models.base_model import BaseModel

@pytest.fixture(scope="session")
def event_loop():
    """Create event loop for async tests"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture
async def db_session():
    """Create in-memory database for testing"""
    engine = create_engine("sqlite:///:memory:", echo=False)
    BaseModel.metadata.create_all(engine)
    SessionLocal = sessionmaker(bind=engine)
    session = SessionLocal()
    try:
        yield session
    finally:
        session.close()

@pytest.fixture
def mock_external_services():
    """Mock all external service dependencies"""
    return {
        'contract_parser_client': Mock(),
        'intelligence_engine_client': Mock(),
        'notification_service': Mock(),
        'metrics_client': Mock()
    }
```

### 2. Test Data Factories

```python
# tests/factories.py
from datetime import datetime
from src.domain.entities.user import User
from src.domain.entities.project import Project
from src.domain.entities.analysis import Analysis
from src.domain.value_objects.email import Email

class UserFactory:
    @staticmethod
    def create(email="test@example.com", username="testuser", **kwargs):
        return User(
            user_id=kwargs.get('user_id', 1),
            email=Email(email),
            username=username,
            full_name=kwargs.get('full_name', 'Test User')
        )

class ProjectFactory:
    @staticmethod
    def create(name="Test Project", user_id=1, **kwargs):
        return Project(
            project_id=kwargs.get('project_id', 1),
            name=name,
            description=kwargs.get('description', 'Test project'),
            owner_id=user_id,
            created_at=kwargs.get('created_at', datetime.utcnow())
        )

class AnalysisFactory:
    @staticmethod
    def create(project_id=1, **kwargs):
        return Analysis.create_new(
            project_id=project_id,
            contract_name=kwargs.get('contract_name', 'TestContract'),
            contract_source=kwargs.get('contract_source', 'contract Test {}'),
            compiler_version=kwargs.get('compiler_version', '0.8.19'),
            submitted_by=kwargs.get('submitted_by', 1)
        )
```

## Performance Testing

### 1. Load Testing with Locust

```python
# tests/performance/locustfile.py
from locust import HttpUser, task, between

class AnalysisAPIUser(HttpUser):
    wait_time = between(1, 3)

    def on_start(self):
        """Login and get auth token"""
        response = self.client.post("/api/v1/auth/login", json={
            "email": "test@example.com",
            "password": "password123"
        })
        self.token = response.json()["access_token"]
        self.headers = {"Authorization": f"Bearer {self.token}"}

    @task(3)
    def submit_contract(self):
        """Submit contract for analysis"""
        self.client.post("/api/v1/analysis/submit", json={
            "project_id": 1,
            "contract_source": "contract Test { function test() {} }",
            "contract_name": "LoadTestContract",
            "compiler_version": "0.8.19"
        }, headers=self.headers)

    @task(1)
    def get_analysis_results(self):
        """Get analysis results"""
        self.client.get("/api/v1/analysis/1/results", headers=self.headers)
```

## Best Practices

### 1. Test Organization
- Follow the testing pyramid: many unit tests, some integration tests, few E2E tests
- Group tests by architectural layer
- Use consistent naming conventions
- Keep tests focused and independent

### 2. Mocking Strategy
- Mock external dependencies at service boundaries
- Use real implementations for domain logic testing
- Create test doubles for complex external services
- Avoid over-mocking internal components

### 3. Data Management
- Use factories for consistent test data creation
- Clean up test data between tests
- Use separate test databases
- Consider using database transactions for isolation

### 4. Continuous Integration
- Run unit tests on every commit
- Run integration tests on pull requests
- Run E2E tests on deployment candidates
- Monitor test execution time and optimize slow tests

This comprehensive testing strategy ensures reliable, maintainable code across all layers of the DDD architecture.
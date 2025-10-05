# CQRS Patterns and Examples

## Overview

Command Query Responsibility Segregation (CQRS) is a fundamental pattern in our Domain-Driven Design architecture. This guide provides practical examples and best practices for implementing CQRS in the Solidity Security Platform.

## Core Concepts

### Commands vs Queries

**Commands** (Write Operations):
- Change system state
- Return success/failure or created entity
- May trigger side effects (events, notifications)
- Should be validated against business rules

**Queries** (Read Operations):
- Retrieve data without side effects
- Optimized for specific read scenarios
- Can use different data models than write side
- Should be fast and cacheable

## Command Pattern Implementation

### 1. Command Definition

Commands are simple data structures representing user intentions.

```python
# src/application/analysis/commands/submit_contract_command.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class SubmitContractCommand:
    project_id: int
    contract_source: str
    contract_name: str
    compiler_version: str
    optimization_enabled: bool = False
    optimization_runs: Optional[int] = None
    user_id: int = None  # Set by authentication middleware

    def __post_init__(self):
        if self.optimization_enabled and self.optimization_runs is None:
            raise ValueError("optimization_runs required when optimization is enabled")
```

### 2. Command Handler

Handlers contain the business logic for processing commands.

```python
# src/application/analysis/handlers/submit_contract_handler.py
from typing import Optional
from ...domain.entities.analysis import Analysis
from ...domain.entities.project import Project
from ...domain.repositories.project_repository import ProjectRepository
from ...domain.repositories.analysis_repository import AnalysisRepository
from ...domain.services.analysis_workflow_service import AnalysisWorkflowService
from ...domain.value_objects.analysis_status import AnalysisStatus
from ...infrastructure.external.contract_parser_client import ContractParserClient
from ..commands.submit_contract_command import SubmitContractCommand

class SubmitContractCommandHandler:
    def __init__(
        self,
        project_repository: ProjectRepository,
        analysis_repository: AnalysisRepository,
        workflow_service: AnalysisWorkflowService,
        parser_client: ContractParserClient
    ):
        self._project_repository = project_repository
        self._analysis_repository = analysis_repository
        self._workflow_service = workflow_service
        self._parser_client = parser_client

    async def handle(self, command: SubmitContractCommand) -> Analysis:
        # 1. Validate project exists and user has access
        project = await self._project_repository.find_by_id(command.project_id)
        if not project:
            raise ValueError(f"Project {command.project_id} not found")

        if not project.user_can_submit_contracts(command.user_id):
            raise PermissionError("User cannot submit contracts to this project")

        # 2. Create analysis entity
        analysis = Analysis.create_new(
            project_id=command.project_id,
            contract_name=command.contract_name,
            contract_source=command.contract_source,
            compiler_version=command.compiler_version,
            submitted_by=command.user_id
        )

        # 3. Validate analysis can be started
        if not self._workflow_service.can_start_analysis(analysis):
            raise ValueError("Analysis cannot be started")

        # 4. Save analysis in SUBMITTED status
        saved_analysis = await self._analysis_repository.save(analysis)

        # 5. Start async parsing process (fire and forget)
        await self._start_parsing_workflow(saved_analysis, command)

        return saved_analysis

    async def _start_parsing_workflow(self, analysis: Analysis, command: SubmitContractCommand):
        """Start the parsing workflow asynchronously"""
        try:
            # Update status to PARSING
            analysis.start_parsing()
            await self._analysis_repository.save(analysis)

            # Submit to parser service
            parse_result = await self._parser_client.parse_contract(
                contract_source=command.contract_source,
                compiler_version=command.compiler_version,
                optimization_enabled=command.optimization_enabled,
                optimization_runs=command.optimization_runs
            )

            # Handle parse result
            if parse_result.success:
                analysis.complete_parsing(parse_result.ast, parse_result.bytecode)
            else:
                analysis.fail_parsing(parse_result.error_message)

            await self._analysis_repository.save(analysis)

        except Exception as e:
            # Handle parsing failure
            analysis.fail_parsing(f"Parsing failed: {str(e)}")
            await self._analysis_repository.save(analysis)
```

### 3. Command Validation

Use decorators or validation classes for cross-cutting concerns.

```python
# src/application/common/validation.py
from functools import wraps
from typing import Callable, Any

def validate_command(validation_func: Callable):
    """Decorator for command validation"""
    def decorator(handler_method):
        @wraps(handler_method)
        async def wrapper(self, command):
            # Run validation
            validation_result = await validation_func(command)
            if not validation_result.is_valid:
                raise ValueError(f"Command validation failed: {validation_result.errors}")

            # Execute handler
            return await handler_method(self, command)
        return wrapper
    return decorator

# Usage in handler
@validate_command(validate_submit_contract_command)
async def handle(self, command: SubmitContractCommand) -> Analysis:
    # Handler logic here
    pass
```

## Query Pattern Implementation

### 1. Query Definition

Queries specify what data to retrieve and any filtering criteria.

```python
# src/application/analysis/queries/get_analysis_results_query.py
from dataclasses import dataclass
from typing import Optional
from datetime import datetime

@dataclass
class GetAnalysisResultsQuery:
    analysis_id: int
    user_id: int  # For authorization
    include_raw_data: bool = False
    include_recommendations: bool = True

@dataclass
class ListProjectAnalysesQuery:
    project_id: int
    user_id: int
    status_filter: Optional[str] = None
    from_date: Optional[datetime] = None
    to_date: Optional[datetime] = None
    page: int = 1
    page_size: int = 20
```

### 2. Query Handler

Query handlers focus on efficient data retrieval.

```python
# src/application/analysis/handlers/analysis_query_handler.py
from typing import List, Optional
from ...domain.entities.analysis import Analysis
from ...domain.repositories.analysis_repository import AnalysisRepository
from ...domain.repositories.project_repository import ProjectRepository
from ..queries.get_analysis_results_query import GetAnalysisResultsQuery, ListProjectAnalysesQuery

class AnalysisQueryHandler:
    def __init__(
        self,
        analysis_repository: AnalysisRepository,
        project_repository: ProjectRepository
    ):
        self._analysis_repository = analysis_repository
        self._project_repository = project_repository

    async def handle(self, query: GetAnalysisResultsQuery) -> Optional[Analysis]:
        # 1. Get analysis
        analysis = await self._analysis_repository.find_by_id(query.analysis_id)
        if not analysis:
            return None

        # 2. Check user permissions
        project = await self._project_repository.find_by_id(analysis.project_id)
        if not project or not project.user_can_view_results(query.user_id):
            raise PermissionError("User cannot view this analysis")

        # 3. Filter data based on query options
        if not query.include_raw_data:
            analysis.clear_raw_data()

        if not query.include_recommendations:
            analysis.clear_recommendations()

        return analysis

    async def handle_list(self, query: ListProjectAnalysesQuery) -> List[Analysis]:
        # 1. Validate project access
        project = await self._project_repository.find_by_id(query.project_id)
        if not project or not project.user_can_view_analyses(query.user_id):
            raise PermissionError("User cannot view project analyses")

        # 2. Build filters
        filters = {
            'project_id': query.project_id,
            'status': query.status_filter,
            'from_date': query.from_date,
            'to_date': query.to_date
        }

        # 3. Execute query with pagination
        return await self._analysis_repository.find_by_filters(
            filters=filters,
            page=query.page,
            page_size=query.page_size
        )
```

### 3. Read Model Optimization

Create specialized read models for complex queries.

```python
# src/application/analysis/read_models/analysis_summary.py
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional

@dataclass
class AnalysisSummary:
    """Optimized read model for analysis listings"""
    id: int
    project_id: int
    contract_name: str
    status: str
    severity_level: Optional[str]
    vulnerability_count: int
    created_at: datetime
    completed_at: Optional[datetime]

@dataclass
class ProjectAnalyticsSummary:
    """Aggregated analytics for project dashboard"""
    project_id: int
    total_analyses: int
    completed_analyses: int
    failed_analyses: int
    critical_vulnerabilities: int
    high_vulnerabilities: int
    medium_vulnerabilities: int
    low_vulnerabilities: int
    average_analysis_time_minutes: Optional[float]
    recent_analyses: List[AnalysisSummary]
```

## Event-Driven CQRS

### 1. Domain Events

Events represent things that happened in the domain.

```python
# src/domain/events/analysis_events.py
from dataclasses import dataclass
from datetime import datetime
from typing import Dict, Any

@dataclass
class AnalysisSubmittedEvent:
    analysis_id: int
    project_id: int
    user_id: int
    contract_name: str
    occurred_at: datetime

@dataclass
class AnalysisCompletedEvent:
    analysis_id: int
    project_id: int
    severity_level: str
    vulnerability_count: int
    analysis_duration_seconds: int
    occurred_at: datetime

@dataclass
class VulnerabilitiesFoundEvent:
    analysis_id: int
    project_id: int
    vulnerabilities: List[Dict[str, Any]]
    severity_breakdown: Dict[str, int]
    occurred_at: datetime
```

### 2. Event Handlers

Handle events for side effects like notifications or read model updates.

```python
# src/application/analysis/event_handlers/analysis_event_handlers.py
from ...domain.events.analysis_events import AnalysisCompletedEvent, VulnerabilitiesFoundEvent
from ...infrastructure.external.notification_service import NotificationService
from ...infrastructure.cache.analysis_cache import AnalysisCache

class AnalysisEventHandler:
    def __init__(
        self,
        notification_service: NotificationService,
        analysis_cache: AnalysisCache
    ):
        self._notification_service = notification_service
        self._analysis_cache = analysis_cache

    async def handle_analysis_completed(self, event: AnalysisCompletedEvent):
        """Handle analysis completion"""
        # 1. Update cache
        await self._analysis_cache.invalidate_analysis(event.analysis_id)

        # 2. Send notification
        await self._notification_service.notify_analysis_completed(
            analysis_id=event.analysis_id,
            project_id=event.project_id
        )

        # 3. Update project statistics
        await self._update_project_stats(event.project_id)

    async def handle_vulnerabilities_found(self, event: VulnerabilitiesFoundEvent):
        """Handle critical vulnerabilities"""
        if event.severity_breakdown.get('CRITICAL', 0) > 0:
            await self._notification_service.send_critical_vulnerability_alert(
                analysis_id=event.analysis_id,
                project_id=event.project_id,
                critical_count=event.severity_breakdown['CRITICAL']
            )
```

## Advanced CQRS Patterns

### 1. Command Composition

Compose multiple commands for complex workflows.

```python
# src/application/projects/handlers/project_setup_handler.py
class ProjectSetupCommandHandler:
    async def handle(self, command: SetupProjectCommand) -> Project:
        # 1. Create project
        create_project_cmd = CreateProjectCommand(
            name=command.project_name,
            description=command.description,
            user_id=command.user_id
        )
        project = await self._create_project_handler.handle(create_project_cmd)

        # 2. Configure security rules
        if command.security_rules:
            configure_rules_cmd = ConfigureSecurityRulesCommand(
                project_id=project.id,
                rules=command.security_rules
            )
            await self._configure_rules_handler.handle(configure_rules_cmd)

        # 3. Set up monitoring
        setup_monitoring_cmd = SetupMonitoringCommand(
            project_id=project.id,
            alert_email=command.alert_email
        )
        await self._setup_monitoring_handler.handle(setup_monitoring_cmd)

        return project
```

### 2. Query Caching

Implement caching for expensive queries.

```python
# src/application/analysis/handlers/cached_analysis_query_handler.py
from functools import wraps

def cache_query(cache_key_func, ttl_seconds=300):
    """Decorator for query result caching"""
    def decorator(handler_method):
        @wraps(handler_method)
        async def wrapper(self, query):
            cache_key = cache_key_func(query)

            # Try cache first
            cached_result = await self._cache.get(cache_key)
            if cached_result:
                return cached_result

            # Execute query
            result = await handler_method(self, query)

            # Cache result
            await self._cache.set(cache_key, result, ttl_seconds)
            return result
        return wrapper
    return decorator

class CachedAnalysisQueryHandler(AnalysisQueryHandler):
    @cache_query(
        cache_key_func=lambda q: f"analysis:{q.analysis_id}:user:{q.user_id}",
        ttl_seconds=600
    )
    async def handle(self, query: GetAnalysisResultsQuery) -> Optional[Analysis]:
        return await super().handle(query)
```

## Testing CQRS Components

### 1. Command Handler Testing

```python
# tests/unit/application/test_submit_contract_handler.py
import pytest
from unittest.mock import AsyncMock, Mock

from src.application.analysis.commands.submit_contract_command import SubmitContractCommand
from src.application.analysis.handlers.submit_contract_handler import SubmitContractCommandHandler

@pytest.fixture
def mock_repositories():
    return {
        'project_repository': AsyncMock(),
        'analysis_repository': AsyncMock(),
        'workflow_service': Mock(),
        'parser_client': AsyncMock()
    }

@pytest.mark.asyncio
async def test_submit_contract_success(mock_repositories):
    # Arrange
    handler = SubmitContractCommandHandler(**mock_repositories)

    command = SubmitContractCommand(
        project_id=1,
        contract_source="contract Test {}",
        contract_name="Test",
        compiler_version="0.8.19",
        user_id=1
    )

    mock_project = Mock()
    mock_project.user_can_submit_contracts.return_value = True
    mock_repositories['project_repository'].find_by_id.return_value = mock_project

    # Act
    result = await handler.handle(command)

    # Assert
    assert result is not None
    mock_repositories['analysis_repository'].save.assert_called_once()

@pytest.mark.asyncio
async def test_submit_contract_permission_denied(mock_repositories):
    # Arrange
    handler = SubmitContractCommandHandler(**mock_repositories)
    command = SubmitContractCommand(
        project_id=1,
        contract_source="contract Test {}",
        contract_name="Test",
        compiler_version="0.8.19",
        user_id=2
    )

    mock_project = Mock()
    mock_project.user_can_submit_contracts.return_value = False
    mock_repositories['project_repository'].find_by_id.return_value = mock_project

    # Act & Assert
    with pytest.raises(PermissionError):
        await handler.handle(command)
```

### 2. Query Handler Testing

```python
# tests/unit/application/test_analysis_query_handler.py
@pytest.mark.asyncio
async def test_get_analysis_results_success(mock_repositories):
    # Arrange
    handler = AnalysisQueryHandler(**mock_repositories)
    query = GetAnalysisResultsQuery(analysis_id=1, user_id=1)

    mock_analysis = Mock()
    mock_analysis.project_id = 1
    mock_repositories['analysis_repository'].find_by_id.return_value = mock_analysis

    mock_project = Mock()
    mock_project.user_can_view_results.return_value = True
    mock_repositories['project_repository'].find_by_id.return_value = mock_project

    # Act
    result = await handler.handle(query)

    # Assert
    assert result == mock_analysis
```

## Best Practices

### 1. Command Design
- Keep commands simple and focused
- Include all necessary data for the operation
- Validate commands before processing
- Use immutable command objects

### 2. Query Optimization
- Design queries for specific use cases
- Use read models for complex data projections
- Implement caching for expensive queries
- Consider eventual consistency for read models

### 3. Handler Patterns
- One handler per command/query
- Keep handlers focused and testable
- Use dependency injection for external dependencies
- Handle errors gracefully with proper exception types

### 4. Event Integration
- Publish events for significant business occurrences
- Keep event handlers independent and idempotent
- Use events for read model updates and notifications
- Consider event sourcing for audit requirements

This CQRS implementation provides clear separation between read and write operations while maintaining the flexibility to optimize each side independently.
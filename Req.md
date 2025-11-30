# Risk Dashboard - Technical Requirements Document

## 1. Project Overview

### 1.1 Background

We are migrating a complex Excel-based trader risk management system to a web application. The current system is a massive spreadsheet with 50+ tabs, each containing multiple data grids, summary cards, alerts, and action buttons. Traders use it to view risk, open/close intraday positions, trades, PnL, market data, and more.

Current pain points:
- Excel freezes for 1-2 minutes during data load
- No observability or debugging capability when errors occur
- Data comes from multiple disparate sources (SQL Server, file shares, HTTP requests, RICs)
- Support is difficult - cannot reproduce trader issues
- No execution history or audit trail

### 1.2 Goals

1. **Performance**: Progressive loading - traders see first tab in seconds, not minutes
2. **Observability**: Every execution is logged with full context for replay and debugging
3. **Maintainability**: Clean architecture that scales to 50+ tabs without becoming unmaintainable
4. **Consistency**: All actions go through the same patterns - no rogue code paths
5. **Flexibility**: Support what-if scenarios and overrides without breaking the architecture

### 1.3 Non-Goals (Phase 1)

- Real-time streaming (polling/manual refresh is acceptable)
- User authentication (handled by infrastructure)
- Mobile support

---

## 2. Architecture Overview

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              BROWSER                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  Panel UI (served via websocket)                                      │  │
│  │  - Tabs with grids, cards, forms                                      │  │
│  │  - Perspective widgets for data display                               │  │
│  │  - All filtering/sorting happens client-side                          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Websocket (Panel manages this)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PANEL SERVER                                      │
│                           (Tornado)                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  - Maintains UI session per user                                      │  │
│  │  - Handles button clicks, form submissions                            │  │
│  │  - Calls FastAPI backend for data                                     │  │
│  │  - Pushes updates to browser                                          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ HTTP/JSON (async)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          FASTAPI SERVER                                     │
│                          (Uvicorn)                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  - Stateless request handling                                         │  │
│  │  - SQL queries (parallelised)                                         │  │
│  │  - Business logic (migrated Excel formulas)                           │  │
│  │  - Execution context storage (HYDRA)                                  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  SQL Server  │  HYDRA  │ etc │
                    └───────────────────────────────┘
```

### 2.2 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| UI Framework | Panel + Perspective | Python-native, Perspective built for financial grids, no JS required |
| Backend Framework | FastAPI | Modern async, automatic validation, auto-generated docs |
| Data Grid | Perspective | JPM-created, handles 100k+ rows, client-side filtering |
| Communication | HTTP/JSON | Simple, debuggable, stateless |
| State Storage | HYDRA | Internal object store, stores execution contexts for replay |

### 2.3 Separation of Concerns

```
Panel Server (UI):
  - What things look like
  - Where things are positioned
  - Loading states and spinners
  - User interactions (clicks, form inputs)
  - Does NOT contain business logic

FastAPI Server (Backend):
  - Business logic
  - Data fetching (SQL, files, APIs)
  - Calculations (migrated Excel formulas)
  - Execution context management
  - Does NOT know about UI layout
```

---

## 3. Environment Configuration

### 3.1 Overview

The application supports four environments:
- **local**: Developer laptop, everything runs locally
- **dev**: Development servers, unstable/latest code
- **uat**: User acceptance testing, release candidates
- **prod**: Production, trader-facing

Additionally, the Panel UI can point to any backend environment, enabling:
- Local UI development against remote data
- Debugging production issues locally
- Testing UI changes against UAT data

### 3.2 Configuration File

No `.env` files. All configuration in plain Python for clarity and version control.

```python
# config/environments.py

"""
Environment configuration for Risk Dashboard.

To add a new environment:
1. Add entry to ENVIRONMENTS dict
2. Update Environment type hint
"""

from typing import Literal

# Type for environment names
Environment = Literal['local', 'dev', 'uat', 'prod']

# All environment configurations
ENVIRONMENTS: dict[Environment, dict] = {
    'local': {
        'api_base_url': 'http://localhost:8000',
        'display_name': 'Local',
        'colour': 'blue',
        'debug': True,
        'log_level': 'DEBUG',
    },
    'dev': {
        'api_base_url': 'http://dev-risk-api.internal:8000',
        'display_name': 'Development',
        'colour': 'green',
        'debug': True,
        'log_level': 'DEBUG',
    },
    'uat': {
        'api_base_url': 'http://uat-risk-api.internal:8000',
        'display_name': 'UAT',
        'colour': 'orange',
        'debug': False,
        'log_level': 'INFO',
    },
    'prod': {
        'api_base_url': 'http://prod-risk-api.internal:8000',
        'display_name': 'Production',
        'colour': 'red',
        'debug': False,
        'log_level': 'WARNING',
    },
}

# Default environment when app starts
DEFAULT_ENVIRONMENT: Environment = 'local'
DEFAULT_REMOTE: bool = False  # If False, use local backend regardless of environment
```

### 3.3 Runtime Settings

```python
# config/settings.py

"""
Runtime settings management.

Settings can be changed at runtime via the Settings tab in the UI.
Changes take effect on next API call (no restart required).
"""

from dataclasses import dataclass, field
from config.environments import (
    Environment, 
    ENVIRONMENTS, 
    DEFAULT_ENVIRONMENT, 
    DEFAULT_REMOTE,
)


@dataclass
class AppSettings:
    """
    Current application settings.
    
    Attributes:
        environment: Which environment config to use (local/dev/uat/prod)
        remote: If True, use remote backend. If False, use localhost.
    """
    environment: Environment = DEFAULT_ENVIRONMENT
    remote: bool = DEFAULT_REMOTE
    
    @property
    def api_base_url(self) -> str:
        """Get the current API URL based on settings."""
        if not self.remote:
            return 'http://localhost:8000'
        return ENVIRONMENTS[self.environment]['api_base_url']
    
    @property
    def display_name(self) -> str:
        """Human-readable environment name."""
        if not self.remote:
            return f"Local → {self.environment}"
        return ENVIRONMENTS[self.environment]['display_name']
    
    @property
    def colour(self) -> str:
        """Colour for environment badge."""
        if not self.remote:
            return 'blue'
        return ENVIRONMENTS[self.environment]['colour']
    
    @property
    def debug(self) -> bool:
        """Whether debug mode is enabled."""
        return ENVIRONMENTS[self.environment]['debug']


# Global settings instance (singleton)
# Tabs and components access this for current configuration
_settings: AppSettings = None


def get_settings() -> AppSettings:
    """Get the global settings instance."""
    global _settings
    if _settings is None:
        _settings = AppSettings()
    return _settings


def update_settings(environment: Environment = None, remote: bool = None):
    """
    Update settings at runtime.
    
    Called by the Settings tab when user changes environment.
    """
    settings = get_settings()
    if environment is not None:
        settings.environment = environment
    if remote is not None:
        settings.remote = remote
```

---

## 4. Folder Structure

```
risk_dashboard/
│
├── app.py                          # Entry point - wires everything together
│
├── config/
│   ├── __init__.py
│   ├── environments.py             # Environment definitions (the ONLY place to define envs)
│   └── settings.py                 # Runtime settings management
│
├── api/
│   ├── __init__.py
│   └── client.py                   # API client - ALL backend calls go through here
│
├── components/                     # Reusable UI building blocks
│   ├── __init__.py
│   ├── base.py                     # BaseComponent - all components inherit from this
│   ├── grids.py                    # Grid (Perspective wrapper)
│   ├── cards.py                    # SummaryCard, AlertCard, MetricCard
│   ├── actions.py                  # ActionForm - standardised action buttons with params
│   └── indicators.py               # Status badges, loading indicators
│
├── tabs/                           # One file per tab (or folder for related tabs)
│   ├── __init__.py
│   ├── base.py                     # BaseTab - all tabs inherit from this
│   ├── settings.py                 # Settings tab (environment management)
│   ├── summary.py                  # Summary/overview tab
│   ├── irdelta.py
│   ├── vega.py
│   ├── bonds/
│   │   ├── __init__.py
│   │   ├── eur_govt.py
│   │   ├── usd_corp.py
│   │   └── gbp_gilt.py
│   ├── futures/
│   │   ├── __init__.py
│   │   ├── stir.py
│   │   └── bond_futures.py
│   └── swaps/
│       ├── __init__.py
│       └── irs.py
│
├── state/
│   ├── __init__.py
│   └── execution.py                # Execution state management (current execution_id, etc.)
│
└── utils/
    ├── __init__.py
    └── formatting.py               # Number formatting, colour coding, etc.
```

### 4.1 Folder Purposes

| Folder | Purpose | Key Rule |
|--------|---------|----------|
| `config/` | Environment and settings | No business logic, just configuration |
| `api/` | Backend communication | ALL HTTP calls go through `client.py` |
| `components/` | Reusable UI pieces | Dumb components - display data, nothing else |
| `tabs/` | Page-level layouts | One file per tab, inherits from `BaseTab` |
| `state/` | Global state management | Execution tracking, abort flags |
| `utils/` | Shared utilities | Pure functions, no state |

---

## 5. Component System

### 5.1 Base Component

All UI components inherit from `BaseComponent`. This guarantees a consistent interface.

```python
# components/base.py

"""
Base class for all UI components.

Every component must:
- Accept data via update(data) method
- Have a loading property
- Handle None/empty data gracefully

This enables generic code in BaseTab to manage any component type.
"""

import panel as pn
import param


class BaseComponent(pn.viewable.Viewer):
    """
    Base class for all UI components.
    
    Subclasses must implement:
        - __panel__(): Returns the Panel object to render
        - update(data): Updates component with new data
    """
    
    # Reactive properties - changes automatically update UI
    loading = param.Boolean(
        default=False, 
        doc="Whether component is in loading state"
    )
    visible = param.Boolean(
        default=True, 
        doc="Whether component should be displayed"
    )
    error = param.String(
        default=None, 
        doc="Error message to display, if any"
    )
    
    def update(self, data):
        """
        Update component with new data.
        
        Args:
            data: The data to display. Type depends on component.
                  Must handle None gracefully (show empty state).
        """
        raise NotImplementedError("Subclass must implement update()")
    
    def clear(self):
        """Reset component to empty state."""
        self.update(None)
        self.error = None
    
    def set_loading(self, is_loading: bool):
        """Set loading state."""
        self.loading = is_loading
    
    def set_error(self, message: str):
        """Display error state."""
        self.error = message
        self.loading = False
```

### 5.2 Grid Component

Wrapper around Perspective for consistent grid display.

```python
# components/grids.py

"""
Grid components for displaying tabular data.

Uses Perspective under the hood for:
- High performance (100k+ rows)
- Client-side filtering, sorting, grouping
- Financial data formatting
"""

import panel as pn
import param
import pandas as pd
from components.base import BaseComponent


class Grid(BaseComponent):
    """
    A data grid using Perspective.
    
    Usage:
        grid = Grid(title="Positions", height=400)
        grid.update(my_dataframe)
    
    Features:
        - Automatic loading spinner
        - Empty state handling
        - Error display
        - Perspective filtering/sorting built-in
    """
    
    # Configuration (set at creation, don't change)
    title = param.String(default="Data")
    height = param.Integer(default=300)
    
    # Data (changes trigger re-render)
    data = param.DataFrame(default=None)
    
    def __panel__(self):
        """Render the component."""
        
        # Error state
        if self.error:
            return pn.Card(
                pn.pane.Alert(self.error, alert_type="danger"),
                title=self.title,
                height=self.height,
                styles={'border': '2px solid red'},
            )
        
        # Loading state
        if self.loading:
            return pn.Card(
                pn.Column(
                    pn.indicators.LoadingSpinner(value=True, size=50),
                    pn.pane.Markdown("Loading...", align='center'),
                    align='center',
                ),
                title=self.title,
                height=self.height,
            )
        
        # Empty state
        if self.data is None or (isinstance(self.data, pd.DataFrame) and self.data.empty):
            return pn.Card(
                pn.pane.Markdown("No data", align='center'),
                title=self.title,
                height=self.height,
                styles={'opacity': '0.6'},
            )
        
        # Data state
        return pn.Card(
            pn.pane.Perspective(
                self.data,
                height=self.height - 50,  # Account for card header
                sizing_mode='stretch_width',
                theme='pro',  # or 'pro-dark' for dark mode
            ),
            title=self.title,
            height=self.height,
        )
    
    def update(self, data):
        """
        Update grid with new data.
        
        Args:
            data: pandas DataFrame or None
        """
        if data is not None and not isinstance(data, pd.DataFrame):
            data = pd.DataFrame(data)
        self.data = data
        self.loading = False
        self.error = None
```

### 5.3 Card Components

```python
# components/cards.py

"""
Card components for displaying summary information.

Cards are non-tabular displays: metrics, alerts, status indicators.
"""

import panel as pn
import param
from components.base import BaseComponent


class SummaryCard(BaseComponent):
    """
    A card showing key-value metrics.
    
    Usage:
        card = SummaryCard(title="Summary")
        card.update({
            'notional': 45000000,
            'pnl': 234000,
            'dv01': 12445,
            'status': 'ok',
        })
    
    Features:
        - Automatic number formatting
        - PnL colour coding (green/red)
        - Status indicators
    """
    
    title = param.String(default="Summary")
    width = param.Integer(default=300)
    
    # The metrics to display: {name: value}
    metrics = param.Dict(default={})
    
    def __panel__(self):
        if self.error:
            return pn.Card(
                pn.pane.Alert(self.error, alert_type="danger"),
                title=self.title,
                width=self.width,
            )
        
        if self.loading:
            return pn.Card(
                pn.indicators.LoadingSpinner(value=True),
                title=self.title,
                width=self.width,
            )
        
        if not self.metrics:
            return pn.Card(
                pn.pane.Markdown("No data"),
                title=self.title,
                width=self.width,
            )
        
        rows = []
        for name, value in self.metrics.items():
            formatted = self._format_value(name, value)
            rows.append(
                pn.Row(
                    pn.pane.Markdown(f"**{self._format_label(name)}:**", width=100),
                    pn.pane.Markdown(formatted, width=150),
                )
            )
        
        return pn.Card(
            pn.Column(*rows),
            title=self.title,
            width=self.width,
        )
    
    def _format_label(self, name: str) -> str:
        """Convert snake_case to Title Case."""
        return name.replace('_', ' ').title()
    
    def _format_value(self, name: str, value) -> str:
        """Format value based on name/type."""
        if value is None:
            return "—"
        
        # Status field
        if name.lower() == 'status':
            icons = {
                'ok': '🟢 OK',
                'warning': '🟡 Warning', 
                'breach': '🔴 Breach',
                'error': '🔴 Error',
            }
            return icons.get(str(value).lower(), f'⚪ {value}')
        
        # Numeric fields
        if isinstance(value, (int, float)):
            # PnL fields - colour code
            if 'pnl' in name.lower() or 'pl' in name.lower():
                colour = 'green' if value >= 0 else 'red'
                return f"<span style='color:{colour}'>€{value:+,.0f}</span>"
            # Other numbers
            return f"€{value:,.0f}"
        
        return str(value)
    
    def update(self, data):
        """Update with new metrics dict."""
        self.metrics = data or {}
        self.loading = False
        self.error = None


class AlertCard(BaseComponent):
    """
    A card showing alert messages.
    
    Usage:
        card = AlertCard(title="Alerts")
        card.update([
            {'message': 'Limit exceeded BUND', 'severity': 'error'},
            {'message': 'Tenor mismatch', 'severity': 'warning'},
        ])
    """
    
    title = param.String(default="Alerts")
    width = param.Integer(default=300)
    max_display = param.Integer(default=5)
    
    alerts = param.List(default=[])
    
    def __panel__(self):
        if self.loading:
            return pn.Card(
                pn.indicators.LoadingSpinner(value=True),
                title=self.title,
                width=self.width,
            )
        
        if not self.alerts:
            return pn.Card(
                pn.pane.Markdown("✅ No alerts"),
                title=self.title,
                width=self.width,
            )
        
        items = []
        for alert in self.alerts[:self.max_display]:
            icon = self._severity_icon(alert.get('severity', 'info'))
            items.append(pn.pane.Markdown(f"{icon} {alert['message']}"))
        
        if len(self.alerts) > self.max_display:
            items.append(pn.pane.Markdown(
                f"*...and {len(self.alerts) - self.max_display} more*"
            ))
        
        return pn.Card(
            pn.Column(*items),
            title=f"{self.title} ({len(self.alerts)})",
            width=self.width,
        )
    
    def _severity_icon(self, severity: str) -> str:
        icons = {
            'error': '🔴',
            'warning': '🟡',
            'info': '🔵',
        }
        return icons.get(severity.lower(), '⚪')
    
    def update(self, data):
        """Update with new alerts list."""
        self.alerts = data or []
        self.loading = False
        self.error = None
```

### 5.4 Action Form Component

**Critical**: All user actions go through `ActionForm`. No rogue buttons.

We use **Pydantic models** to define action parameters. This gives us:
- Type safety (no string-based type checking)
- Automatic validation
- IDE autocomplete
- Self-documenting schemas
- Reusable parameter models

```python
# components/actions.py

"""
Action form components using Pydantic for type-safe parameter definitions.

ALL user actions must go through ActionForm, even actions with no parameters.
This ensures:
- Consistent UI/UX
- Type-safe parameter handling
- Automatic validation via Pydantic
- All actions can be disabled during global operations
"""

import panel as pn
import param
from typing import Callable, Awaitable, Type, get_type_hints, get_origin, get_args, Literal, Annotated, Union
from datetime import date, time, datetime
from enum import Enum
from pydantic import BaseModel, Field
from pydantic.fields import FieldInfo
from components.base import BaseComponent


# ═══════════════════════════════════════════════════════════════════════════
# CUSTOM FIELD TYPES - For special widget behaviors
# ═══════════════════════════════════════════════════════════════════════════

class TimeStr(str):
    """Marker type for time strings (HH:MM format)."""
    pass


class SelectOption:
    """Marker for select/dropdown fields. Use with Literal types."""
    pass


# ═══════════════════════════════════════════════════════════════════════════
# WIDGET FACTORY - Maps Python types to Panel widgets
# ═══════════════════════════════════════════════════════════════════════════

class WidgetFactory:
    """
    Creates Panel widgets from Pydantic field types.
    
    Supports:
        - bool → Checkbox
        - int → IntInput
        - float → FloatInput
        - str → TextInput
        - TimeStr → TextInput with HH:MM placeholder
        - date → DatePicker
        - datetime → DatetimePicker
        - Literal['a', 'b', 'c'] → Select dropdown
        - Enum → Select dropdown
    """
    
    @classmethod
    def create_widget(
        cls, 
        field_name: str, 
        field_type: Type, 
        field_info: FieldInfo,
    ) -> pn.widgets.Widget:
        """
        Create appropriate widget for a Pydantic field.
        
        Args:
            field_name: Name of the field
            field_type: Python type annotation
            field_info: Pydantic FieldInfo with metadata
        
        Returns:
            Panel widget instance
        """
        # Extract label from Field description or generate from name
        label = field_info.description or field_name.replace('_', ' ').title()
        default = field_info.default if field_info.default is not None else None
        
        # Handle Optional types (Union with None)
        origin = get_origin(field_type)
        if origin is Union:
            args = get_args(field_type)
            # Filter out NoneType to get the actual type
            non_none_args = [a for a in args if a is not type(None)]
            if len(non_none_args) == 1:
                field_type = non_none_args[0]
                origin = get_origin(field_type)
        
        # Handle Literal types → Select dropdown
        if origin is Literal:
            options = list(get_args(field_type))
            return pn.widgets.Select(
                name=label,
                options=options,
                value=default if default in options else options[0],
            )
        
        # Handle Enum types → Select dropdown
        if isinstance(field_type, type) and issubclass(field_type, Enum):
            options = {e.name: e.value for e in field_type}
            return pn.widgets.Select(
                name=label,
                options=options,
                value=default.value if default else list(options.values())[0],
            )
        
        # Handle basic types
        if field_type is bool:
            return pn.widgets.Checkbox(
                name=label,
                value=default if default is not None else False,
            )
        
        if field_type is int:
            return pn.widgets.IntInput(
                name=label,
                value=default if default is not None else 0,
            )
        
        if field_type is float:
            return pn.widgets.FloatInput(
                name=label,
                value=default if default is not None else 0.0,
            )
        
        if field_type is date:
            return pn.widgets.DatePicker(
                name=label,
                value=default,
            )
        
        if field_type is datetime:
            return pn.widgets.DatetimePicker(
                name=label,
                value=default,
            )
        
        if field_type is time or field_type is TimeStr:
            return pn.widgets.TextInput(
                name=label,
                value=str(default) if default else "",
                placeholder="HH:MM",
            )
        
        # Default: string input
        return pn.widgets.TextInput(
            name=label,
            value=default if default is not None else "",
            placeholder=field_info.json_schema_extra.get('placeholder', '') if field_info.json_schema_extra else '',
        )


# ═══════════════════════════════════════════════════════════════════════════
# EMPTY PARAMS MODEL - For actions with no parameters
# ═══════════════════════════════════════════════════════════════════════════

class EmptyParams(BaseModel):
    """Use this for actions that have no parameters."""
    pass


# ═══════════════════════════════════════════════════════════════════════════
# ACTION FORM COMPONENT
# ═══════════════════════════════════════════════════════════════════════════

class ActionForm(BaseComponent):
    """
    A form with Pydantic-typed parameters and a submit button.
    
    Usage:
        # Define parameters as a Pydantic model
        class LoadIntradayParams(BaseModel):
            as_of_time: TimeStr = Field(default="12:00", description="As Of Time")
            include_cancelled: bool = Field(default=False, description="Include Cancelled")
        
        # Create form
        form = ActionForm(
            name="Load Intraday",
            params_model=LoadIntradayParams,
            on_submit=self._on_load_intraday,
        )
        
        # Action with NO parameters (still use ActionForm!)
        form = ActionForm(
            name="Refresh Prices",
            params_model=EmptyParams,
            on_submit=self._on_refresh_prices,
        )
    
    The on_submit handler receives a validated Pydantic model instance:
        async def _on_load_intraday(self, params: LoadIntradayParams):
            # params.as_of_time, params.include_cancelled are typed!
            await self._api.post('/endpoint', params.model_dump())
    """
    
    name = param.String(default="Action")
    button_type = param.String(default="primary")
    collapsed = param.Boolean(default=True)
    
    def __init__(
        self,
        name: str,
        params_model: Type[BaseModel],
        on_submit: Callable[[BaseModel], Awaitable],
        button_type: str = "primary",
        collapsed: bool = True,
        **kwargs
    ):
        super().__init__(**kwargs)
        self.name = name
        self.button_type = button_type
        self.collapsed = collapsed
        self._params_model = params_model
        self._on_submit = on_submit
        
        # Build widgets from Pydantic model
        self._widgets = self._build_widgets_from_model()
        
        # Submit button
        self._submit_btn = pn.widgets.Button(
            name=self.name,
            button_type=self.button_type,
        )
        self._submit_btn.on_click(self._handle_submit)
        
        # Status indicator
        self._status = pn.pane.Markdown("", width=150)
    
    def _build_widgets_from_model(self) -> dict:
        """Create widgets by introspecting the Pydantic model."""
        widgets = {}
        
        # Get field definitions from Pydantic model
        for field_name, field_info in self._params_model.model_fields.items():
            field_type = field_info.annotation
            
            widgets[field_name] = WidgetFactory.create_widget(
                field_name=field_name,
                field_type=field_type,
                field_info=field_info,
            )
        
        return widgets
    
    def __panel__(self):
        """Render the form."""
        button_row = pn.Row(self._submit_btn, self._status)
        
        # No params - just show button
        if not self._widgets:
            return button_row
        
        # Has params - show in collapsible card
        return pn.Card(
            pn.Column(
                *self._widgets.values(),
                button_row,
            ),
            title=self.name,
            collapsed=self.collapsed,
        )
    
    async def _handle_submit(self, event):
        """Handle form submission with Pydantic validation."""
        
        # Disable during execution
        self._submit_btn.disabled = True
        for widget in self._widgets.values():
            widget.disabled = True
        self._status.object = "⏳"
        
        try:
            # Collect values from widgets
            raw_values = {
                name: widget.value
                for name, widget in self._widgets.items()
            }
            
            # Validate through Pydantic model
            # This raises ValidationError if invalid
            params = self._params_model(**raw_values)
            
            # Call handler with validated model
            await self._on_submit(params)
            
            self._status.object = "✅"
        
        except ValueError as e:
            # Pydantic validation error
            self._status.object = f"❌ {e}"
        
        except Exception as e:
            self._status.object = "❌"
            self.error = str(e)
        
        finally:
            # Re-enable
            self._submit_btn.disabled = False
            for widget in self._widgets.values():
                widget.disabled = False
    
    def set_disabled(self, disabled: bool):
        """Disable/enable the entire form."""
        self._submit_btn.disabled = disabled
        for widget in self._widgets.values():
            widget.disabled = disabled
    
    def update(self, data):
        """ActionForm doesn't receive data updates."""
        pass
    
    def get_current_values(self) -> BaseModel:
        """Get current values as a validated Pydantic model."""
        raw_values = {
            name: widget.value
            for name, widget in self._widgets.items()
        }
        return self._params_model(**raw_values)
    
    def get_current_values_dict(self) -> dict:
        """Get current values as a dict."""
        return self.get_current_values().model_dump()
```

---

## 6. Tab System

### 6.1 Base Tab

All tabs inherit from `BaseTab`. This enforces the pattern.

```python
# tabs/base.py

"""
Base class for all tabs.

Every tab MUST inherit from BaseTab and implement three methods:
- define_components(): What widgets exist in this tab
- define_layout(): How the widgets are arranged
- map_data(): How API response maps to widgets

This pattern ensures:
- Consistent structure across 50+ tabs
- Automatic loading state management
- Automatic error handling
- Easy to find and modify any tab
"""

import panel as pn
from abc import ABC, abstractmethod
from api.client import get_api_client


class BaseTab(pn.viewable.Viewer, ABC):
    """
    Base class for all tabs.
    
    Subclasses must implement:
        - define_components() -> dict[str, BaseComponent]
        - define_layout(components: dict) -> pn.viewable.Viewable
        - map_data(api_response: dict) -> dict[str, Any]
    
    Subclasses must set:
        - name: str - Display name for tab
        - api_endpoint: str - Backend endpoint (e.g., '/risk/eur_govt')
    
    Example:
        class MyTab(BaseTab):
            name = "My Tab"
            api_endpoint = "/risk/my_tab"
            
            def define_components(self):
                return {
                    'summary': SummaryCard(),
                    'grid': Grid(title="Data"),
                }
            
            def define_layout(self, c):
                return pn.Column(c['summary'], c['grid'])
            
            def map_data(self, api_response):
                return {
                    'summary': api_response['summary'],
                    'grid': api_response['data'],
                }
    """
    
    # Subclass must set these
    name: str = "Unnamed Tab"
    api_endpoint: str = None
    
    def __init__(self):
        super().__init__()
        
        # Get API client
        self._api = get_api_client()
        
        # Create components
        self._components = self.define_components()
        
        # Create layout
        self._layout = self.define_layout(self._components)
        
        # Error display (shown at top of tab if error occurs)
        self._error_pane = pn.pane.Alert(
            "", 
            alert_type="danger", 
            visible=False,
        )
        
        # Optional: custom behaviour hook
        self.define_custom_behaviour()
    
    def __panel__(self):
        """Render the tab."""
        return pn.Column(
            self._error_pane,
            self._layout,
            sizing_mode='stretch_width',
        )
    
    # ═══════════════════════════════════════════════════════════════
    # SUBCLASS MUST IMPLEMENT THESE
    # ═══════════════════════════════════════════════════════════════
    
    @abstractmethod
    def define_components(self) -> dict:
        """
        Define all components in this tab.
        
        Returns:
            Dict mapping component names to component instances.
            Names are used in define_layout() and map_data().
        
        Example:
            return {
                'summary': SummaryCard(title="Summary"),
                'positions': Grid(title="Positions", height=400),
                'refresh_btn': ActionForm(name="Refresh", params={}, on_submit=self._on_refresh),
            }
        """
        pass
    
    @abstractmethod
    def define_layout(self, c: dict):
        """
        Arrange components into a layout.
        
        Args:
            c: Dict of components from define_components().
               Use c['name'] to reference components.
        
        Returns:
            A Panel layout (Column, Row, Card, etc.)
        
        Example:
            return pn.Column(
                pn.Row(c['summary'], c['alerts']),
                c['positions'],
                c['refresh_btn'],
            )
        """
        pass
    
    @abstractmethod
    def map_data(self, api_response: dict) -> dict:
        """
        Map API response to components.
        
        Args:
            api_response: JSON response from backend API
        
        Returns:
            Dict mapping component names to their data.
            Keys must match keys from define_components().
            Values are passed to each component's update() method.
        
        Example:
            return {
                'summary': api_response['summary'],
                'positions': api_response['positions_df'],
                # Note: action forms typically not in map_data
            }
        """
        pass
    
    # ═══════════════════════════════════════════════════════════════
    # OPTIONAL HOOK FOR CUSTOM BEHAVIOUR
    # ═══════════════════════════════════════════════════════════════
    
    def define_custom_behaviour(self):
        """
        Optional hook for tab-specific behaviour.
        
        Override this for things that don't fit the standard pattern.
        Document WHY in the docstring.
        
        Examples:
            - Cross-tab communication
            - Custom keyboard shortcuts
            - Conditional component visibility
        
        If you use this often, consider whether it should be a pattern
        in BaseTab or a new component type.
        """
        pass
    
    # ═══════════════════════════════════════════════════════════════
    # PROVIDED BY BASETAB - SUBCLASS TYPICALLY DOESN'T OVERRIDE
    # ═══════════════════════════════════════════════════════════════
    
    def set_loading(self, is_loading: bool):
        """
        Set loading state on all components.
        
        Called by app.py during Load All.
        """
        for component in self._components.values():
            if hasattr(component, 'set_loading'):
                component.set_loading(is_loading)
            elif hasattr(component, 'loading'):
                component.loading = is_loading
    
    def set_disabled(self, disabled: bool):
        """
        Disable all action forms.
        
        Called by app.py during Load All to prevent conflicting actions.
        """
        for component in self._components.values():
            if hasattr(component, 'set_disabled'):
                component.set_disabled(disabled)
    
    def update(self, api_response: dict):
        """
        Update all components from API response.
        
        Called by app.py when data arrives for this tab.
        """
        try:
            # Get mapping from subclass
            data_mapping = self.map_data(api_response)
            
            # Update each component
            for component_name, data in data_mapping.items():
                if component_name not in self._components:
                    raise KeyError(f"map_data returned unknown component: {component_name}")
                
                component = self._components[component_name]
                component.update(data)
            
            # Clear any previous error
            self._error_pane.visible = False
            
        except Exception as e:
            self._error_pane.object = f"Error updating tab: {e}"
            self._error_pane.visible = True
    
    def set_error(self, message: str):
        """Display an error message at the top of the tab."""
        self._error_pane.object = message
        self._error_pane.visible = True
    
    def clear_error(self):
        """Clear any displayed error."""
        self._error_pane.visible = False
    
    def get_component(self, name: str):
        """
        Get a component by name.
        
        Useful in define_custom_behaviour() or action handlers.
        """
        return self._components.get(name)
```

### 6.2 Settings Tab

Special tab for environment management.

```python
# tabs/settings.py

"""
Settings tab for environment management.

This tab allows runtime configuration of:
- Backend environment (local/dev/uat/prod)
- Remote vs local backend
- Displays current API URL

Future additions:
- User preferences
- Display settings
- Debug options
"""

import panel as pn
from tabs.base import BaseTab
from config.environments import ENVIRONMENTS, Environment
from config.settings import get_settings, update_settings
from components.base import BaseComponent


class EnvironmentSelector(BaseComponent):
    """
    Environment selection component.
    
    Two dropdowns:
    - Environment: local/dev/uat/prod
    - Remote: True/False
    
    Plus display of resulting API URL.
    """
    
    def __init__(self):
        super().__init__()
        settings = get_settings()
        
        # Environment dropdown
        self._env_select = pn.widgets.Select(
            name="Backend Environment",
            options=list(ENVIRONMENTS.keys()),
            value=settings.environment,
        )
        self._env_select.param.watch(self._on_env_change, 'value')
        
        # Remote toggle
        self._remote_select = pn.widgets.Select(
            name="Use Remote Backend",
            options={'Yes (use remote server)': True, 'No (use localhost)': False},
            value=settings.remote,
        )
        self._remote_select.param.watch(self._on_remote_change, 'value')
        
        # URL display
        self._url_display = pn.pane.Markdown(self._format_url_display())
    
    def _on_env_change(self, event):
        """Handle environment change."""
        update_settings(environment=event.new)
        self._url_display.object = self._format_url_display()
    
    def _on_remote_change(self, event):
        """Handle remote toggle change."""
        update_settings(remote=event.new)
        self._url_display.object = self._format_url_display()
    
    def _format_url_display(self) -> str:
        """Format the current URL for display."""
        settings = get_settings()
        colour = settings.colour
        return f"""
### Current Configuration

**Environment:** <span style='color:{colour}'>{settings.display_name}</span>

**API URL:** `{settings.api_base_url}`

**Debug Mode:** {'Enabled' if settings.debug else 'Disabled'}
"""
    
    def __panel__(self):
        return pn.Card(
            pn.Column(
                self._env_select,
                self._remote_select,
                pn.layout.Divider(),
                self._url_display,
            ),
            title="Environment Configuration",
        )
    
    def update(self, data):
        """Not used - settings are interactive."""
        pass


class SettingsTab(BaseTab):
    """
    Settings tab.
    
    Contains:
    - Environment configuration
    - (Future) User preferences
    - (Future) Debug options
    """
    
    name = "⚙️ Settings"
    api_endpoint = None  # Settings tab doesn't call API
    
    def define_components(self):
        return {
            'environment': EnvironmentSelector(),
        }
    
    def define_layout(self, c):
        return pn.Column(
            pn.pane.Markdown("# Settings"),
            pn.pane.Markdown("Configure application settings. Changes take effect immediately."),
            c['environment'],
            sizing_mode='stretch_width',
        )
    
    def map_data(self, api_response):
        # Settings tab doesn't receive data from API
        return {}
```

### 6.3 Example Business Tab

```python
# tabs/bonds/eur_govt.py

"""
EUR Government Bonds tab.

Displays:
- Summary metrics (notional, PnL, DV01, status)
- Alerts
- Positions grid
- Risk by tenor grid
- Risk by CCY grid
- Action buttons (Refresh Prices, Load Intraday, Override Spread)

Data contract from API:
    GET /risk/{execution_id}/eur_govt
    Response:
    {
        "summary": {
            "notional": 45000000,
            "pnl": 234000,
            "dv01": 12445,
            "status": "ok"
        },
        "alerts": [
            {"message": "Limit exceeded BUND", "severity": "error"},
            {"message": "Tenor mismatch", "severity": "warning"}
        ],
        "positions": [...],       # List of dicts, becomes DataFrame
        "risk_by_tenor": [...],   # List of dicts, becomes DataFrame
        "risk_by_ccy": [...]      # List of dicts, becomes DataFrame
    }
"""

import panel as pn
from typing import Literal
from pydantic import BaseModel, Field

from tabs.base import BaseTab
from components.cards import SummaryCard, AlertCard
from components.grids import Grid
from components.actions import ActionForm, EmptyParams, TimeStr
from styles.grid_styles import GridPresets, ColumnPresets


# ═══════════════════════════════════════════════════════════════════════════
# ACTION PARAMETER MODELS - Pydantic models for each action
# ═══════════════════════════════════════════════════════════════════════════

class LoadIntradayParams(BaseModel):
    """Parameters for Load Intraday action."""
    as_of_time: TimeStr = Field(
        default="12:00", 
        description="As Of Time",
    )
    include_cancelled: bool = Field(
        default=False, 
        description="Include Cancelled Trades",
    )


class OverrideSpreadParams(BaseModel):
    """Parameters for Override Spread action."""
    instrument: str = Field(
        ...,  # Required (no default)
        description="Instrument",
        json_schema_extra={'placeholder': 'e.g., BUND'},
    )
    spread_bps: float = Field(
        default=0.0, 
        description="Spread (bps)",
    )


class ExportParams(BaseModel):
    """Parameters for Export action."""
    format: Literal['csv', 'xlsx', 'json'] = Field(
        default='xlsx',
        description="Export Format",
    )
    include_hidden: bool = Field(
        default=False,
        description="Include Hidden Columns",
    )


# ═══════════════════════════════════════════════════════════════════════════
# TAB DEFINITION
# ═══════════════════════════════════════════════════════════════════════════

class EurGovtTab(BaseTab):
    """
    EUR Government Bonds tab.
    
    Layout:
    ┌─────────────────────────────────────────────────────────────────┐
    │  ┌─────────────────────────┐  ┌─────────────────────────────┐  │
    │  │  Summary                │  │  Alerts                     │  │
    │  └─────────────────────────┘  └─────────────────────────────┘  │
    ├─────────────────────────────────────────────────────────────────┤
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │  Positions (grid)                                       │   │
    │  └─────────────────────────────────────────────────────────┘   │
    ├─────────────────────────────────────────────────────────────────┤
    │  ┌─────────────────────────┐  ┌─────────────────────────────┐  │
    │  │  Risk by Tenor          │  │  Risk by CCY                │  │
    │  └─────────────────────────┘  └─────────────────────────────┘  │
    ├─────────────────────────────────────────────────────────────────┤
    │  [Refresh Prices]  [Load Intraday]  [Override Spread]  [Export]│
    └─────────────────────────────────────────────────────────────────┘
    """
    
    name = "EUR Govt"
    api_endpoint = "/eur_govt"
    
    def define_components(self):
        return {
            # Data displays
            'summary': SummaryCard(title="Summary"),
            'alerts': AlertCard(title="Alerts"),
            'positions': Grid(
                title="Positions",
                **GridPresets.POSITIONS_GRID,
            ),
            'risk_by_tenor': Grid(
                title="Risk by Tenor",
                height=250,
                columns_config={
                    'dv01': ColumnPresets.RISK_HEATMAP,
                    'pnl': ColumnPresets.PNL,
                },
            ),
            'risk_by_ccy': Grid(
                title="Risk by CCY",
                height=250,
                columns_config={
                    'dv01': ColumnPresets.RISK_HEATMAP,
                    'pnl': ColumnPresets.PNL,
                },
            ),
            
            # Actions - ALL use Pydantic models for params
            'refresh_prices': ActionForm(
                name="Refresh Prices",
                params_model=EmptyParams,  # No params, but still typed
                on_submit=self._on_refresh_prices,
            ),
            'load_intraday': ActionForm(
                name="Load Intraday",
                params_model=LoadIntradayParams,
                on_submit=self._on_load_intraday,
            ),
            'override_spread': ActionForm(
                name="Override Spread",
                params_model=OverrideSpreadParams,
                on_submit=self._on_override_spread,
            ),
            'export': ActionForm(
                name="Export",
                params_model=ExportParams,
                on_submit=self._on_export,
            ),
        }
    
    def define_layout(self, c):
        return pn.Column(
            # Row 1: Summary cards
            pn.Row(
                c['summary'],
                c['alerts'],
                sizing_mode='stretch_width',
            ),
            
            # Row 2: Main positions grid
            c['positions'],
            
            # Row 3: Secondary grids
            pn.Row(
                c['risk_by_tenor'],
                c['risk_by_ccy'],
                sizing_mode='stretch_width',
            ),
            
            # Row 4: Actions
            pn.Row(
                c['refresh_prices'],
                c['load_intraday'],
                c['override_spread'],
                c['export'],
            ),
            
            sizing_mode='stretch_width',
        )
    
    def map_data(self, api_response):
        """Map API response to components."""
        return {
            'summary': api_response.get('summary', {}),
            'alerts': api_response.get('alerts', []),
            'positions': api_response.get('positions'),
            'risk_by_tenor': api_response.get('risk_by_tenor'),
            'risk_by_ccy': api_response.get('risk_by_ccy'),
            # Note: action forms not in map_data - they don't receive data
        }
    
    # ═══════════════════════════════════════════════════════════════
    # ACTION HANDLERS - All receive typed Pydantic models
    # ═══════════════════════════════════════════════════════════════
    
    async def _on_refresh_prices(self, params: EmptyParams):
        """Handle Refresh Prices action."""
        # params is EmptyParams (no fields)
        await self._api.post(f"{self.api_endpoint}/refresh_prices", params.model_dump())
    
    async def _on_load_intraday(self, params: LoadIntradayParams):
        """Handle Load Intraday action."""
        # params.as_of_time and params.include_cancelled are typed!
        await self._api.post(f"{self.api_endpoint}/load_intraday", params.model_dump())
    
    async def _on_override_spread(self, params: OverrideSpreadParams):
        """Handle Override Spread action."""
        # params.instrument (str, required) and params.spread_bps (float)
        await self._api.post(f"{self.api_endpoint}/override_spread", params.model_dump())
    
    async def _on_export(self, params: ExportParams):
        """Handle Export action."""
        # params.format is Literal['csv', 'xlsx', 'json'] - IDE knows the options!
        await self._api.post(f"{self.api_endpoint}/export", params.model_dump())
```

---

## 7. API Client

### 7.1 Client Implementation

```python
# api/client.py

"""
API client for communicating with FastAPI backend.

ALL backend calls MUST go through this client.
No direct HTTP calls anywhere else in the codebase.

This ensures:
- Consistent error handling
- Centralized logging
- Easy to mock for testing
- Configuration managed in one place
"""

import httpx
from typing import Optional
from config.settings import get_settings


class RiskAPIClient:
    """
    Client for the Risk Dashboard FastAPI backend.
    
    Usage:
        client = get_api_client()
        
        # Start a new execution
        exec_id = await client.start_execution({
            'books': ['DESK_A'],
            'overrides': {},
        })
        
        # Fetch data for a tab
        data = await client.fetch_tab(exec_id, '/eur_govt')
        
        # Post an action
        await client.post('/eur_govt/refresh_prices', {})
    """
    
    def __init__(self):
        self._client: Optional[httpx.AsyncClient] = None
    
    @property
    def client(self) -> httpx.AsyncClient:
        """Lazy-init HTTP client with current settings."""
        settings = get_settings()
        
        # Recreate client if settings changed
        if self._client is None or self._client.base_url != settings.api_base_url:
            if self._client:
                # Close old client (fire and forget)
                pass
            self._client = httpx.AsyncClient(
                base_url=settings.api_base_url,
                timeout=120.0,  # Long timeout for big queries
            )
        
        return self._client
    
    async def start_execution(self, request: dict) -> str:
        """
        Start a new execution.
        
        Args:
            request: Execution parameters
                {
                    'user': str,
                    'books': list[str],
                    'overrides': dict,
                    ...
                }
        
        Returns:
            execution_id: Unique identifier for this execution
        """
        response = await self.client.post("/risk/compute", json=request)
        response.raise_for_status()
        return response.json()["execution_id"]
    
    async def fetch_tab(self, execution_id: str, endpoint: str) -> dict:
        """
        Fetch data for a single tab.
        
        Args:
            execution_id: From start_execution()
            endpoint: Tab's API endpoint (e.g., '/eur_govt')
        
        Returns:
            Tab data as dict (matches tab's map_data expectations)
        """
        url = f"/risk/{execution_id}{endpoint}"
        response = await self.client.get(url)
        response.raise_for_status()
        return response.json()
    
    async def post(self, endpoint: str, params: dict) -> dict:
        """
        Post an action.
        
        Args:
            endpoint: Full endpoint path (e.g., '/eur_govt/refresh_prices')
            params: Action parameters
        
        Returns:
            Response data as dict
        """
        response = await self.client.post(endpoint, json=params)
        response.raise_for_status()
        return response.json()
    
    async def abort(self, execution_id: str) -> None:
        """
        Abort a running execution.
        
        Args:
            execution_id: The execution to abort
        """
        await self.client.post(f"/risk/{execution_id}/abort")
    
    async def health_check(self) -> bool:
        """
        Check if backend is reachable.
        
        Returns:
            True if healthy, False otherwise
        """
        try:
            response = await self.client.get("/health")
            return response.status_code == 200
        except Exception:
            return False


# Singleton instance
_api_client: Optional[RiskAPIClient] = None


def get_api_client() -> RiskAPIClient:
    """Get the global API client instance."""
    global _api_client
    if _api_client is None:
        _api_client = RiskAPIClient()
    return _api_client
```

---

## 8. Execution State Management

```python
# state/execution.py

"""
Global execution state management.

Tracks:
- Current execution ID
- Running/aborted status
- Which tabs have loaded

Used by app.py to coordinate Load All behaviour.
"""

from dataclasses import dataclass, field
from typing import Optional, Set


@dataclass
class ExecutionState:
    """
    Current execution state.
    
    Attributes:
        execution_id: Current execution ID (None if no execution)
        is_running: Whether an execution is in progress
        loaded_tabs: Set of tab names that have successfully loaded
        failed_tabs: Set of tab names that failed to load
    """
    execution_id: Optional[str] = None
    is_running: bool = False
    loaded_tabs: Set[str] = field(default_factory=set)
    failed_tabs: Set[str] = field(default_factory=set)
    
    def start(self, execution_id: str):
        """Start a new execution."""
        self.execution_id = execution_id
        self.is_running = True
        self.loaded_tabs = set()
        self.failed_tabs = set()
    
    def tab_loaded(self, tab_name: str):
        """Mark a tab as successfully loaded."""
        self.loaded_tabs.add(tab_name)
    
    def tab_failed(self, tab_name: str):
        """Mark a tab as failed."""
        self.failed_tabs.add(tab_name)
    
    def abort(self):
        """Abort the current execution."""
        self.is_running = False
    
    def complete(self):
        """Mark execution as complete."""
        self.is_running = False
    
    def reset(self):
        """Reset to initial state."""
        self.execution_id = None
        self.is_running = False
        self.loaded_tabs = set()
        self.failed_tabs = set()


# Global state instance
_execution_state: Optional[ExecutionState] = None


def get_execution_state() -> ExecutionState:
    """Get the global execution state."""
    global _execution_state
    if _execution_state is None:
        _execution_state = ExecutionState()
    return _execution_state
```

---

## 9. Main Application

```python
# app.py

"""
Risk Dashboard - Main Application Entry Point

This file:
- Initializes Panel
- Creates all tabs
- Wires up global controls (Load All, Abort)
- Defines the main layout
- Serves the application

To run:
    panel serve app.py --show
    
With environment override:
    API_BASE_URL=http://dev:8000 panel serve app.py --show
"""

import panel as pn
import asyncio

# Initialize Panel with Perspective extension
pn.extension('perspective')

# Imports
from config.settings import get_settings
from api.client import get_api_client
from state.execution import get_execution_state

# Import all tabs
from tabs.settings import SettingsTab
from tabs.summary import SummaryTab
from tabs.irdelta import IRDeltaTab
from tabs.vega import VegaTab
from tabs.bonds.eur_govt import EurGovtTab
from tabs.bonds.usd_corp import UsdCorpTab
# ... import all other tabs


# ═══════════════════════════════════════════════════════════════════════════
# TAB REGISTRY
# ═══════════════════════════════════════════════════════════════════════════

# All tabs in display order
# Settings tab is special - always last
TAB_CLASSES = [
    SummaryTab,
    IRDeltaTab,
    VegaTab,
    EurGovtTab,
    UsdCorpTab,
    # ... all other tab classes
]


def create_tabs():
    """Instantiate all tabs."""
    return [TabClass() for TabClass in TAB_CLASSES]


# ═══════════════════════════════════════════════════════════════════════════
# GLOBAL STATE
# ═══════════════════════════════════════════════════════════════════════════

all_tabs = create_tabs()
settings_tab = SettingsTab()  # Settings tab is separate
api = get_api_client()
execution = get_execution_state()


# ═══════════════════════════════════════════════════════════════════════════
# GLOBAL CONTROLS
# ═══════════════════════════════════════════════════════════════════════════

load_all_btn = pn.widgets.Button(
    name="Load All",
    button_type="primary",
    width=100,
)

abort_btn = pn.widgets.Button(
    name="Abort",
    button_type="danger",
    width=100,
    disabled=True,
)

status_text = pn.pane.Markdown("Ready", width=300)


def get_env_badge():
    """Create environment badge with current settings."""
    settings = get_settings()
    return pn.pane.Markdown(
        f"**{settings.display_name}** | `{settings.api_base_url}`",
        styles={
            'background-color': settings.colour,
            'color': 'white',
            'padding': '5px 10px',
            'border-radius': '3px',
        }
    )


env_badge = get_env_badge()


# ═══════════════════════════════════════════════════════════════════════════
# LOAD ALL LOGIC
# ═══════════════════════════════════════════════════════════════════════════

async def on_load_all(event):
    """
    Handle Load All button click.
    
    Flow:
    1. Disable all buttons except Abort
    2. Set all tabs to loading state
    3. Start execution on backend
    4. Fetch each tab's data sequentially
    5. Update each tab as data arrives
    6. Re-enable buttons when complete
    """
    
    # Lock UI
    load_all_btn.disabled = True
    abort_btn.disabled = False
    for tab in all_tabs:
        tab.set_loading(True)
        tab.set_disabled(True)
    
    try:
        # Start execution
        status_text.object = "🔄 Starting execution..."
        execution.start(
            await api.start_execution({
                'user': 'current_user',  # TODO: Get actual user
                'books': [],  # TODO: Get from UI
                'overrides': {},  # TODO: Get from UI
            })
        )
        
        # Fetch each tab
        for tab in all_tabs:
            # Check for abort
            if not execution.is_running:
                status_text.object = "🛑 Aborted"
                break
            
            # Skip tabs without API endpoint (like Settings)
            if not tab.api_endpoint:
                continue
            
            status_text.object = f"🔄 Loading {tab.name}..."
            
            try:
                # Fetch data
                data = await api.fetch_tab(execution.execution_id, tab.api_endpoint)
                
                # Update tab
                tab.update(data)
                execution.tab_loaded(tab.name)
                
            except Exception as e:
                tab.set_error(f"Failed to load: {e}")
                execution.tab_failed(tab.name)
        
        # Complete
        if execution.is_running:
            execution.complete()
            failed_count = len(execution.failed_tabs)
            if failed_count > 0:
                status_text.object = f"⚠️ Complete with {failed_count} errors"
            else:
                status_text.object = "✅ Complete"
    
    except Exception as e:
        status_text.object = f"❌ Error: {e}"
    
    finally:
        # Unlock UI
        load_all_btn.disabled = False
        abort_btn.disabled = True
        for tab in all_tabs:
            tab.set_disabled(False)


def on_abort(event):
    """Handle Abort button click."""
    execution.abort()
    # The running load loop will see is_running=False and stop


# Wire up buttons
load_all_btn.on_click(on_load_all)
abort_btn.on_click(on_abort)


# ═══════════════════════════════════════════════════════════════════════════
# LAYOUT
# ═══════════════════════════════════════════════════════════════════════════

# Header
header = pn.Row(
    pn.pane.Markdown("# 📊 Risk Dashboard", width=200),
    pn.Spacer(width=50),
    load_all_btn,
    abort_btn,
    pn.Spacer(width=20),
    status_text,
    pn.Spacer(),
    env_badge,
    sizing_mode='stretch_width',
)

# Tab container
tabs = pn.Tabs(
    *[(tab.name, tab) for tab in all_tabs],
    (settings_tab.name, settings_tab),  # Settings always last
    sizing_mode='stretch_width',
    dynamic=True,  # Only render active tab
)

# Main app
app = pn.Column(
    header,
    pn.layout.Divider(),
    tabs,
    sizing_mode='stretch_both',
)

# Serve
app.servable()
```

---

## 10. Architectural Discipline

### 10.1 Patterns to Follow

| Pattern | Rule |
|---------|------|
| One tab = one file | Every tab in its own file in `tabs/` |
| Tabs inherit from BaseTab | No direct Panel usage in tab files |
| Three methods per tab | `define_components`, `define_layout`, `map_data` |
| Components inherit from BaseComponent | Consistent interface |
| All actions through ActionForm | Even no-param actions |
| All API calls through client.py | No direct httpx/requests |
| Config in environments.py | No .env files |

### 10.2 Escape Hatches

When something doesn't fit the pattern:

1. **Use `define_custom_behaviour()`** in tabs for tab-specific logic
2. **Document with `# WEIRD:` comment** explaining why
3. **Log in architecture decision doc** if it's a new pattern

```python
def define_custom_behaviour(self):
    """
    WEIRD: EUR Govt tab needs cross-tab highlighting.
    Traders want to click a bond and see related futures highlighted.
    If more tabs need this, consider a CrossTabLink component.
    """
    self._components['positions'].on_click(self._handle_position_click)
```

### 10.3 Adding a New Tab Checklist

```
□ Create file in tabs/ (e.g., tabs/bonds/new_bond.py)
□ Inherit from BaseTab
□ Set name and api_endpoint
□ Implement define_components()
□ Implement define_layout()
□ Implement map_data()
□ Add import to app.py
□ Add to TAB_CLASSES list
□ Test locally
```

### 10.4 Monitoring Architectural Drift

Periodically run:

```bash
# How many tabs use custom behaviour?
grep -r "def define_custom_behaviour" tabs/ --include="*.py" | grep -v "pass"

# How many WEIRD comments?
grep -r "# WEIRD:" . --include="*.py"

# Any raw pn.widgets in tabs?
grep -r "pn.widgets\." tabs/ --include="*.py"
```

---

## 11. Data Flow Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  USER CLICKS [LOAD ALL]                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  app.py: on_load_all()                                                      │
│                                                                             │
│  1. Disable buttons                                                         │
│  2. For each tab: tab.set_loading(True)                                    │
│  3. api.start_execution() → execution_id                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ For each tab...
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  api.fetch_tab(execution_id, tab.api_endpoint)                              │
│                                                                             │
│  HTTP GET → FastAPI backend → SQL queries + calculations → JSON response   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  tab.update(api_response)                                                   │
│                                                                             │
│  1. Calls tab.map_data(api_response) → {'summary': {...}, 'grid': [...]}   │
│  2. For each component_name, data:                                          │
│       component.update(data)                                                │
│  3. Panel detects param changes → re-renders UI                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Browser shows updated data in Perspective grids                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Next Steps

### Phase 1 (MVP)
1. Set up project structure
2. Implement BaseComponent and core components (Grid, SummaryCard, ActionForm)
3. Implement BaseTab
4. Create Settings tab
5. Create 1-2 business tabs as proof of concept
6. Set up FastAPI backend skeleton with mock data
7. Test end-to-end flow

### Phase 2 (Core Features)
1. Implement all 50+ tabs
2. Connect to real data sources (SQL Server, etc.)
3. Implement execution context storage (HYDRA)
4. Add error handling and retry logic
5. Performance testing with realistic data volumes

### Phase 3 (Polish)
1. Add user preferences persistence
2. Implement export functionality
3. Add keyboard shortcuts
4. Consider real-time updates (polling/SSE)
5. Documentation for traders

---

## 13. Styling System

### 13.1 Philosophy

Styling must be:
- **Consistent**: Same data type looks the same everywhere
- **Centralized**: One place to change "how PnL looks"
- **Semantic**: Style by meaning (PnL, status, risk) not by column name

### 13.2 Folder Addition

```
risk_dashboard/
├── ...
├── styles/
│   ├── __init__.py
│   ├── themes.py          # Global theme settings
│   ├── grid_styles.py     # Perspective column configurations
│   ├── card_styles.py     # Card/component CSS
│   └── presets.py         # Pre-built column configs for common patterns
```

### 13.3 Grid Styling (Perspective)

Perspective supports several styling modes. We centralize these into reusable presets.

```python
# styles/grid_styles.py

"""
Grid styling presets for Perspective.

USE THESE PRESETS. Do not define one-off column configs in tabs.
If you need a new style, add it here so others can reuse it.

Perspective column_config options:
- number_color_mode: 'foreground', 'background', 'gradient', 'bar'
- gradient: Scale for gradient mode
- pos_fg_color / neg_fg_color: Colors for positive/negative
- pos_bg_color / neg_bg_color: Background colors
- string_color_mode: 'foreground', 'background', 'series'
"""

from typing import Dict, Any


# ═══════════════════════════════════════════════════════════════════════════
# COLOR PALETTE - Single source of truth for colors
# ═══════════════════════════════════════════════════════════════════════════

class Colors:
    """Standard colors used throughout the app."""
    
    # PnL colors
    PROFIT_GREEN = "#00C853"
    LOSS_RED = "#FF1744"
    NEUTRAL = "#FFFFFF"
    
    # Status colors
    STATUS_OK = "#00C853"
    STATUS_WARNING = "#FFB300"
    STATUS_ERROR = "#FF1744"
    STATUS_UNKNOWN = "#9E9E9E"
    
    # Risk colors (heat map)
    RISK_LOW = "#E8F5E9"
    RISK_MEDIUM = "#FFF3E0"
    RISK_HIGH = "#FFEBEE"
    RISK_CRITICAL = "#FF1744"
    
    # Highlight
    HIGHLIGHT_ROW = "#FFF9C4"
    SELECTED_ROW = "#BBDEFB"


# ═══════════════════════════════════════════════════════════════════════════
# COLUMN PRESETS - Reusable column configurations
# ═══════════════════════════════════════════════════════════════════════════

class ColumnPresets:
    """
    Pre-built column configurations.
    
    Usage in tabs:
        from styles.grid_styles import ColumnPresets
        
        Grid(
            columns_config={
                'pnl': ColumnPresets.PNL,
                'notional': ColumnPresets.CURRENCY_BAR,
                'status': ColumnPresets.STATUS,
            }
        )
    """
    
    # ─────────────────────────────────────────────────────────────────────
    # NUMERIC STYLES
    # ─────────────────────────────────────────────────────────────────────
    
    PNL: Dict[str, Any] = {
        'number_color_mode': 'foreground',
        'pos_fg_color': Colors.PROFIT_GREEN,
        'neg_fg_color': Colors.LOSS_RED,
    }
    
    PNL_GRADIENT: Dict[str, Any] = {
        'number_color_mode': 'gradient',
        'gradient': 100000,  # Adjust based on typical PnL magnitude
        'pos_fg_color': Colors.PROFIT_GREEN,
        'neg_fg_color': Colors.LOSS_RED,
    }
    
    PNL_BACKGROUND: Dict[str, Any] = {
        'number_color_mode': 'background',
        'pos_bg_color': '#E8F5E9',  # Light green
        'neg_bg_color': '#FFEBEE',  # Light red
    }
    
    CURRENCY: Dict[str, Any] = {
        # Plain formatted number, no color
    }
    
    CURRENCY_BAR: Dict[str, Any] = {
        'number_color_mode': 'bar',
    }
    
    PERCENTAGE: Dict[str, Any] = {
        'number_color_mode': 'foreground',
        'pos_fg_color': Colors.PROFIT_GREEN,
        'neg_fg_color': Colors.LOSS_RED,
    }
    
    RISK_HEATMAP: Dict[str, Any] = {
        'number_color_mode': 'gradient',
        'gradient': 1000000,  # Adjust based on risk magnitude
    }
    
    # ─────────────────────────────────────────────────────────────────────
    # TEXT STYLES
    # ─────────────────────────────────────────────────────────────────────
    
    STATUS: Dict[str, Any] = {
        'string_color_mode': 'series',
    }
    
    INSTRUMENT: Dict[str, Any] = {
        # Plain text, maybe bold
    }
    
    # ─────────────────────────────────────────────────────────────────────
    # SPECIAL STYLES
    # ─────────────────────────────────────────────────────────────────────
    
    DELTA: Dict[str, Any] = {
        'number_color_mode': 'foreground',
        'pos_fg_color': Colors.PROFIT_GREEN,
        'neg_fg_color': Colors.LOSS_RED,
    }
    
    GREEKS: Dict[str, Any] = {
        'number_color_mode': 'gradient',
        'gradient': 10000,
    }


# ═══════════════════════════════════════════════════════════════════════════
# GRID PRESETS - Full grid configurations for common grid types
# ═══════════════════════════════════════════════════════════════════════════

class GridPresets:
    """
    Full grid configurations for common patterns.
    
    Usage:
        Grid(
            title="Positions",
            **GridPresets.POSITIONS_GRID
        )
    """
    
    POSITIONS_GRID: Dict[str, Any] = {
        'columns_config': {
            'pnl': ColumnPresets.PNL,
            'daily_pnl': ColumnPresets.PNL,
            'mtd_pnl': ColumnPresets.PNL,
            'ytd_pnl': ColumnPresets.PNL,
            'notional': ColumnPresets.CURRENCY_BAR,
            'delta': ColumnPresets.DELTA,
            'gamma': ColumnPresets.GREEKS,
            'vega': ColumnPresets.GREEKS,
            'theta': ColumnPresets.GREEKS,
            'status': ColumnPresets.STATUS,
        },
        'height': 400,
    }
    
    RISK_GRID: Dict[str, Any] = {
        'columns_config': {
            'dv01': ColumnPresets.RISK_HEATMAP,
            'cs01': ColumnPresets.RISK_HEATMAP,
            'vega': ColumnPresets.RISK_HEATMAP,
            'pnl': ColumnPresets.PNL,
        },
        'height': 300,
    }
    
    TRADES_GRID: Dict[str, Any] = {
        'columns_config': {
            'notional': ColumnPresets.CURRENCY_BAR,
            'price': ColumnPresets.CURRENCY,
            'pnl': ColumnPresets.PNL,
            'status': ColumnPresets.STATUS,
        },
        'height': 350,
    }


# ═══════════════════════════════════════════════════════════════════════════
# HELPER FUNCTIONS
# ═══════════════════════════════════════════════════════════════════════════

def merge_column_config(base: Dict, overrides: Dict) -> Dict:
    """
    Merge column configs, with overrides taking precedence.
    
    Usage:
        columns_config = merge_column_config(
            GridPresets.POSITIONS_GRID['columns_config'],
            {'special_column': ColumnPresets.PNL_GRADIENT}
        )
    """
    result = base.copy()
    result.update(overrides)
    return result


def make_pnl_gradient(scale: float) -> Dict[str, Any]:
    """
    Create a PnL gradient config with custom scale.
    
    Usage:
        columns_config = {
            'pnl': make_pnl_gradient(500000),  # For larger PnL values
        }
    """
    config = ColumnPresets.PNL_GRADIENT.copy()
    config['gradient'] = scale
    return config
```

### 13.4 Using Styles in Tabs

```python
# tabs/bonds/eur_govt.py

from styles.grid_styles import ColumnPresets, GridPresets, merge_column_config

class EurGovtTab(BaseTab):
    
    def define_components(self):
        return {
            # Use a full preset
            'positions': Grid(
                title="Positions",
                **GridPresets.POSITIONS_GRID,
            ),
            
            # Use preset with overrides
            'risk_by_tenor': Grid(
                title="Risk by Tenor",
                height=250,
                columns_config=merge_column_config(
                    GridPresets.RISK_GRID['columns_config'],
                    {'special_risk': ColumnPresets.PNL_GRADIENT},
                ),
            ),
            
            # Build custom config from presets
            'custom_grid': Grid(
                title="Custom",
                height=300,
                columns_config={
                    'pnl': ColumnPresets.PNL,
                    'weird_column': ColumnPresets.PERCENTAGE,
                },
            ),
        }
```

### 13.5 Card Styling

```python
# styles/card_styles.py

"""
CSS styles for cards and non-grid components.
"""


class CardStyles:
    """
    CSS style dicts for Panel components.
    
    Usage:
        pn.Card(
            content,
            styles=CardStyles.SUMMARY_CARD,
        )
    """
    
    SUMMARY_CARD = {
        'background': '#FAFAFA',
        'border-radius': '8px',
        'box-shadow': '0 2px 4px rgba(0,0,0,0.1)',
    }
    
    ALERT_CARD = {
        'background': '#FFF3E0',
        'border-left': '4px solid #FFB300',
    }
    
    ERROR_CARD = {
        'background': '#FFEBEE',
        'border-left': '4px solid #FF1744',
    }
    
    SUCCESS_CARD = {
        'background': '#E8F5E9',
        'border-left': '4px solid #00C853',
    }


class TextStyles:
    """
    CSS style dicts for text elements.
    """
    
    METRIC_VALUE = {
        'font-size': '1.5em',
        'font-weight': 'bold',
    }
    
    METRIC_LABEL = {
        'font-size': '0.9em',
        'color': '#666666',
    }
    
    PROFIT = {
        'color': '#00C853',
        'font-weight': 'bold',
    }
    
    LOSS = {
        'color': '#FF1744',
        'font-weight': 'bold',
    }
```

### 13.6 Global Theme

```python
# styles/themes.py

"""
Global theme configuration.

Sets Panel/Perspective defaults that apply everywhere.
"""

import panel as pn


def apply_theme():
    """
    Apply global theme settings.
    
    Call this once at app startup.
    """
    
    # Panel raw CSS
    pn.config.raw_css.append('''
        /* Global font */
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        
        /* Card styling */
        .card {
            border-radius: 8px;
        }
        
        /* Tab styling */
        .bk-tab {
            font-weight: 500;
        }
        
        .bk-tab.bk-active {
            border-bottom: 3px solid #1976D2;
        }
        
        /* Loading spinner color */
        .pn-loading .bk-canvas-events {
            background-color: rgba(25, 118, 210, 0.1);
        }
        
        /* Positive/negative values */
        .value-positive {
            color: #00C853;
        }
        
        .value-negative {
            color: #FF1744;
        }
    ''')


# Perspective theme options
PERSPECTIVE_THEME = 'pro'  # or 'pro-dark' for dark mode
```

### 13.7 Putting It Together

Update app.py:

```python
# app.py

import panel as pn
from styles.themes import apply_theme, PERSPECTIVE_THEME

# Apply global theme before creating any components
pn.extension('perspective')
apply_theme()

# ... rest of app.py
```

Update Grid component:

```python
# components/grids.py

from styles.themes import PERSPECTIVE_THEME

class Grid(BaseComponent):
    
    # ...
    
    def __panel__(self):
        # ...
        
        return pn.Card(
            pn.pane.Perspective(
                self.data,
                height=self.height - 50,
                sizing_mode='stretch_width',
                theme=PERSPECTIVE_THEME,  # Use global theme
                columns_config=self.columns_config,
            ),
            title=self.title,
            height=self.height,
        )
```

### 13.8 Styling Rules

| Rule | Rationale |
|------|-----------|
| Colors defined in `Colors` class only | Single source of truth |
| Use `ColumnPresets` for column styling | Consistency across tabs |
| Use `GridPresets` when a full pattern applies | Less boilerplate |
| New styles go in `styles/` folder | Discoverability |
| Never hardcode colors in tabs | Maintainability |

### 13.9 Adding a New Style

```
□ Is this a new color? Add to Colors class
□ Is this a new column pattern? Add to ColumnPresets
□ Is this a new grid pattern? Add to GridPresets
□ Is this a new card style? Add to CardStyles
□ Document what it's for in the docstring
□ Use it via import, not copy-paste
```

---

## Appendix A: Technology Choices Rationale

| Choice | Alternatives Considered | Why This Choice |
|--------|------------------------|-----------------|
| Panel | Dash, Streamlit, React | Python-native, Perspective integration, websocket handled automatically |
| Perspective | AG Grid, DataTables | JPM-created, 100k+ row performance, client-side filtering |
| FastAPI | Flask, Tornado | Modern async, automatic validation, auto docs |
| httpx | requests, aiohttp | Async support, similar API to requests |
| No .env files | python-dotenv | Explicit configuration, version controlled, no magic |

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| Tab | A page in the dashboard (e.g., EUR Govt, IRDelta) |
| Component | A reusable UI element (Grid, SummaryCard, ActionForm) |
| Execution | One "Load All" operation, identified by execution_id |
| Override | User-specified value that replaces computed/database value |
| HYDRA | Internal object store for storing execution contexts |

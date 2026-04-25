<python_coding_standards version="3.13+">
  <language_baseline>
    <rule>Target Python 3.13+. Use all modern syntax. Never write backwards-compatible code for older versions.</rule>
    <rule>Do not use `from __future__ import annotations`. It is unnecessary on 3.13+.</rule>
  </language_baseline>
  <typing>
    <modern_syntax_rules>
      <rule id="pep_695_generics">
        <description>Use PEP 695 inline type parameters, not `TypeVar`.</description>
        <example type="correct">
          ```python
          def first[T](iterable: Iterable[T]) -> T: ...
          class Repository[TModel]: ...
          ```
        </example>
        <example type="incorrect">
          ```python
          T = TypeVar('T')
          def first(iterable: Iterable[T]) -> T: ...
          ```
        </example>
      </rule>
      <rule id="pep_695_type_aliases">
        <description>Use the `type` statement for aliases, not `TypeAlias`.</description>
        <example type="correct">
          ```python
          type JsonData = int | float | str | dict[str, JsonData] | list[JsonData] | None
          type State = Literal['pending', 'active', 'closed']
          ```
        </example>
        <example type="incorrect">
          ```python
          JsonData: TypeAlias = Union[int, float, str, ...]
          ```
        </example>
      </rule>
      <rule id="pep_604_unions">
        <description>Always use `X | Y | None`. Never use `Optional[X]` or `Union[X, Y]`.</description>
      </rule>
      <rule id="builtin_generics">
        <description>Use built-in generics: `dict[K, V]`, `list[T]`, `tuple[T, ...]`, `set[T]`, `frozenset[T]`. Never import `Dict`, `List`, `Tuple`, `Set`, `FrozenSet` from `typing`.</description>
      </rule>
      <rule id="collections_abc">
        <description>Import abstract types from `collections.abc`, not `typing`.</description>
        <applies_to>`Callable`, `Iterable`, `Iterator`, `Mapping`, `Sequence`, `Collection`, `Generator`, `Awaitable`, `Coroutine`, `MutableMapping`, `MutableSequence`</applies_to>
      </rule>
    </modern_syntax_rules>
    <typing_module_imports>
      <description>Only import from `typing` what does not exist elsewhere.</description>
      <allowed>`Any`, `ClassVar`, `Final`, `Literal`, `Self`, `Protocol`, `TypeGuard`, `TypeIs`, `Unpack`, `TypedDict`, `NamedTuple`, `overload`, `cast`, `TYPE_CHECKING`, `runtime_checkable`, `Never`, `Concatenate`, `ParamSpec`, `assert_type`</allowed>
    </typing_module_imports>
    <annotation_conventions>
      <rule>Annotate all public function signatures — parameters and return types.</rule>
      <rule>Annotate private/internal functions when it adds clarity; skip when obvious.</rule>
      <rule id="overload_usage">
        <description>Use `overload` when return type depends on input values or types. Prefer precise overloads over a loose union return type.</description>
      </rule>
      <rule id="protocol_usage">
        <description>Use `Protocol` for structural typing — especially for callback/handler signatures.</description>
        <example type="correct">
          ```python
          class EventHandler(Protocol):
            def __call__(self, event: Event, *, context: Context = ...) -> Response: ...
          ```
        </example>
      </rule>
      <rule id="type_checking_guard">
        <description>Use `TYPE_CHECKING` for imports only needed by type checkers or to break circular imports.</description>
        <example type="correct">
          ```python
          from typing import TYPE_CHECKING
          if TYPE_CHECKING:
            from myproject.services import UserService
          ```
        </example>
      </rule>
      <rule id="classvar_final">
        <description>Use `ClassVar` for class-level state. Use `Final` for constants that must not be reassigned.</description>
      </rule>
      <rule id="any_acceptable">
        <description>`Any` is acceptable for genuinely dynamic data (serialization, JSON, reflection). Don't build over-complex generics to avoid it.</description>
      </rule>
      <rule id="literal_for_restricted_values">
        <description>Use `Literal` for restricted values instead of bare `str` or `int`.</description>
        <example type="correct">
          ```python
          type OutputFormat = Literal['json', 'csv', 'xml']
          ```
        </example>
      </rule>
    </annotation_conventions>
  </typing>
  <module_organization>
    <package_structure>
      <rule>Organize by concern, not by technical layer. Group related functionality into subpackages (`auth/`, `cache/`, `events/`, `billing/`).</rule>
      <rule>Use `_components/` subdirectories to decompose complex modules into small, focused files while keeping the public API at the parent level.</rule>
      <rule>`_components/__init__.py` files should be empty — they are implementation namespaces, not re-export surfaces.</rule>
    </package_structure>
    <public_vs_private>
      <convention scope="private_modules">Leading underscore (`_utils.py`, `_types.py`, `_internal.py`).</convention>
      <convention scope="private_classes">Leading underscore (`class _InternalParser`).</convention>
      <convention scope="private_instance_attributes">Double underscore for name mangling (`self.__state`).</convention>
      <convention scope="protected_methods">Single underscore (`def _on_connect(self)`) for subclass API.</convention>
      <convention scope="public_api">No underscore. Use `__all__` only when the module exports a lot and you need to be explicit.</convention>
    </public_vs_private>
    <init_files>
      <rule>Keep `__init__.py` files minimal or empty. Don't aggregate and re-export everything.</rule>
      <rule>Users should import from the specific module: `from mypackage.auth.tokens import TokenService`, not `from mypackage.auth import TokenService` via a fat `__init__.py`.</rule>
    </init_files>
    <imports>
      <rule id="absolute_imports">
        <description>Absolute imports everywhere. No relative imports.</description>
      </rule>
      <rule id="import_order">
        <description>Import order, enforced with isort or ruff:</description>
        <order>
          1. Standard library
          2. Third-party
          3. Project-local
        </order>
      </rule>
      <rule id="lazy_imports">
        <description>Use lazy imports inside functions for optional or heavy dependencies.</description>
        <example type="correct">
          ```python
          def render_html():
            from markdown import Markdown  # optional dep
            ...
          ```
        </example>
      </rule>
      <rule id="optional_extras">
        <description>Use `try/except ImportError` for optional extras.</description>
        <example type="correct">
          ```python
          try:
            from redis import Redis
          except ImportError:
            Redis = None
          ```
        </example>
      </rule>
    </imports>
  </module_organization>
  <coding_style>
    <naming_conventions>

      | Element | Convention | Example |
      |---------|-----------|---------|
      | Functions, methods, variables | `snake_case` | `parse_token`, `retry_count` |
      | Classes | `PascalCase` | `ConnectionPool`, `RetryPolicy` |
      | Constants | `UPPER_SNAKE_CASE` | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
      | Type aliases | `PascalCase` | `type RequestData = ...` |
      | Private type aliases | `_PascalCase` | `type _InternalState = ...` |

    </naming_conventions>
    <sentinel_values>
      <description>Use `object()` for "not provided" vs `None` disambiguation.</description>
      <example type="correct">
        ```python
        _GUARD: Final[Any] = object()
        _MISSING: Final[Any] = object()

        def get(key: str, default: str = _GUARD) -> str:
          ...
        ```
      </example>
    </sentinel_values>
    <data_containers>
      <rule id="frozen_dataclass">
        <description>Use `@dataclass(frozen=True)` for value objects, configuration, and DTOs.</description>
        <example type="correct">
          ```python
          @dataclass(frozen=True)
          class RetryConfig:
            max_attempts: int = 3
            backoff_factor: float = 0.5
          ```
        </example>
      </rule>
      <rule id="namedtuple">
        <description>Use `NamedTuple` for lightweight structured return values.</description>
        <example type="correct">
          ```python
          class ParseResult(NamedTuple):
            value: str
            remaining: int
          ```
        </example>
      </rule>
      <rule id="typeddict">
        <description>Use `TypedDict` for typed dict shapes (API payloads, config dicts).</description>
      </rule>
      <rule id="slots">
        <description>Don't add `__slots__` unless profiling proves a memory problem.</description>
      </rule>
    </data_containers>
    <constants>
      <rule>
        <description>Always annotate constants with `Final`.</description>
        <example type="correct">
          ```python
          MAX_RETRIES: Final = 3
          DEFAULT_ENCODING: Final[str] = 'utf-8'
          ```
        </example>
      </rule>
      <rule>
        <description>Provide immutable empty collection singletons rather than mutable defaults.</description>
        <example type="correct">
          ```python
          EMPTY_MAP: Final[Mapping[Any, Any]] = MappingProxyType({})
          EMPTY_SET: Final[frozenset[Any]] = frozenset()
          ```
        </example>
      </rule>
    </constants>
    <classes>
      <rule id="abc_with_init_subclass">
        <description>Use ABC + `__init_subclass__` for abstract bases with subclass validation.</description>
        <example type="correct">
          ```python
          class Plugin(ABC):
            def __init_subclass__(cls, **kwargs):
              super().__init_subclass__(**kwargs)
              if not hasattr(cls, 'name'):
                raise TypeError(f'{cls.__name__} must define a "name" attribute')
          ```
        </example>
      </rule>
      <rule id="protocol_over_abc">
        <description>Prefer `Protocol` over ABC when you only need structural compatibility.</description>
      </rule>
      <rule id="classmethod_staticmethod">
        <description>Use `@classmethod` for factory/alternate constructors. Use `@staticmethod` for pure utility functions scoped to the class.</description>
      </rule>
      <rule id="self_return">
        <description>Use `Self` return type for builder patterns and fluent APIs.</description>
      </rule>
    </classes>
    <enums>
      <rule>Use `StrEnum` for string enums (3.11+).</rule>
      <rule>Use `Enum` for non-string value enums.</rule>
      <rule>Prefer enums over bare string/int constants for any value with a fixed set of options.</rule>
    </enums>
    <exceptions>
      <rule>
        <description>Custom exceptions should carry structured metadata (codes, context), not just messages.</description>
        <example type="correct">
          ```python
          class ValidationError(ValueError):
            def __init__(self, field: str, message: str):
              self.field = field
              super().__init__(f'{field}: {message}')
          ```
        </example>
      </rule>
      <rule>Use stable identifiers (codes, UUIDs) for error types that cross API boundaries.</rule>
    </exceptions>
  </coding_style>
  <async_patterns>
    <rule>Use `async`/`await` consistently. Don't mix sync blocking calls into async code.</rule>
    <rule>Use async context managers (`__aenter__`/`__aexit__`) for resource lifecycle.</rule>
    <rule id="typed_async_callbacks">
      <description>Type async callbacks precisely.</description>
      <example type="correct">
        ```python
        type ErrorHandler = Callable[
          [Exception],
          Awaitable[bool] | bool
        ]
        ```
      </example>
    </rule>
    <rule>When both sync and async variants exist, the async variant is the primary API.</rule>
  </async_patterns>
  <testing>
    <conventions>
      <rule id="test_structure">
        <description>Test structure mirrors source structure — `tests/auth/test_tokens.py` tests `mypackage/auth/tokens.py`.</description>
      </rule>
      <rule id="test_naming">
        <description>Use `test_<behavior_under_test>`, not `test_method_name`.</description>
      </rule>
      <rule id="subtest_matrices">
        <description>Use `subTest` for test matrices in unittest.</description>
        <example type="correct">
          ```python
          for value, expected in cases:
            with self.subTest(value=value):
              self.assertEqual(process(value), expected)
          ```
        </example>
      </rule>
      <rule id="mocking">
        <description>Use `unittest.mock.patch` / `patch.object`. Mock at boundaries, not internals.</description>
      </rule>
      <rule id="async_tests">
        <description>Use an async test mixin or `unittest.IsolatedAsyncioTestCase` (3.11+).</description>
      </rule>
      <rule id="integration_test_tagging">
        <description>Tag or mark integration tests (database, network, external services) so they can be excluded in fast runs.</description>
      </rule>
    </conventions>
    <unit_testing_principles>
      <description>How to decide what to test and how to write tests that catch real regressions.</description>
      <rule id="test_observable_behavior">
        <heading>Test observable behavior, not internal mechanism.</heading>
        <description>Assert on what callers see: return values, emitted events, mock calls to external boundaries. Don't assert on private attributes, internal call ordering, or implementation steps that exist only because of how you happened to write the function. If the implementation is rewritten and the behavior is preserved, the test should still pass.</description>
      </rule>
      <rule id="no_tautological_tests">
        <heading>Don't test what the structure already guarantees.</heading>
        <description>Before writing a test, check whether the assertion can fail given the code under test. If the function's only job is `await x()` and the test asserts `x` was awaited, the test cannot fail without the function being deleted — that's a tautology, not a test.</description>
        <example type="incorrect">
          ```python
          # function is `await publish()`; test asserts publish was awaited
          async def fn():
            await publish()

          async def test_fn_publishes():
            await fn()
            publish.assert_awaited_once()
          ```
        </example>
      </rule>
      <rule id="no_decorator_bypass">
        <heading>Don't bypass decorators or wrappers to reach the target.</heading>
        <description>If you reach for `tool.__wrapped__`, `func.__func__`, `cls.__dict__["m"]`, or similar to dodge a decorator, you are testing a function the production code never calls. Test the public surface the framework actually invokes, or test the underlying helper directly as its own unit.</description>
        <example type="incorrect">
          ```python
          # tool is exposed via @function_tool; test calls the raw inner fn
          await my_tool.__wrapped__(ctx, ...)
          ```
        </example>
      </rule>
      <rule id="no_plumbing_tests">
        <heading>Don't unit test plumbing.</heading>
        <description>Wiring code (DI registration, decorator application, framework hookup) is verified by the framework itself or by integration tests. A unit test that asserts "this function is registered" or "this decorator was applied" tests the framework, not your code.</description>
      </rule>
      <rule id="regression_check">
        <heading>Before writing a test, ask: what regression would this catch?</heading>
        <description>If you cannot describe a realistic code change that would break this test and represent a real bug, don't write it. "Someone might delete the function" is not a realistic regression.</description>
      </rule>
      <rule id="test_longest_path">
        <heading>Test the longest path; don't write subset tests of it.</heading>
        <description>If test A asserts `[mute, say]` and test B asserts `[mute, ring, say]` on the same code path with a different input, B already covers A's invariant. Keep B, drop A. Two tests where one is a strict subset of the other is duplication, not coverage.</description>
      </rule>
      <rule id="parametrize_correctly">
        <heading>Parametrize on same invariant + different input; split on branched assertions.</heading>
        <description>Use `@pytest.mark.parametrize` only when every case asserts the same thing. The moment the test body needs `if param == "x": assert A else: assert B`, split into separate functions — the parametrize is hiding two different tests.</description>
        <example type="incorrect">
          ```python
          # branched assertions inside a parametrized test
          @pytest.mark.parametrize("mode", ["sync", "async"])
          def test_run(mode):
            result = run(mode)
            if mode == "sync":
              assert result.value == 1
            else:
              assert result.awaited
          ```
        </example>
      </rule>
    </unit_testing_principles>
  </testing>
  <docstrings>
    <rule id="sphinx_style">
      <description>Use Sphinx-style `:param:` / `:return:` on public APIs.</description>
      <example type="correct">
        ```python
        def retry[T](
          fn: Callable[[], Awaitable[T]],
          *,
          max_attempts: int = 3,
        ) -> T:
          """
          Retries an async callable on failure.
          :param fn: The callable to retry.
          :param max_attempts: Maximum number of attempts before raising.
          :return: The value returned by the callable.
          """
        ```
      </example>
    </rule>
    <rule>No docstrings on private/obvious methods. Comments only where logic isn't self-evident.</rule>
    <rule>No trailing summaries. Don't explain what you just did — the code and diff speak for themselves.</rule>
  </docstrings>
  <key_patterns>
    <description>Summary callouts of the most important patterns to apply consistently.</description>
    <pattern id="1">**`_components/` decomposition** — split complex modules into focused files under `_components/`, keep the public surface at the parent.</pattern>
    <pattern id="2">**Sentinels over `None`** — `_GUARD = object()` to distinguish "not provided" from `None`.</pattern>
    <pattern id="3">**`overload` for precision** — when return type varies by input, provide typed overloads.</pattern>
    <pattern id="4">**`Protocol` for contracts** — define callback/handler shapes as Protocols, not ABCs.</pattern>
    <pattern id="5">**Frozen dataclasses** — `@dataclass(frozen=True)` for config, value objects, DTOs.</pattern>
    <pattern id="6">**Lazy imports** — import optional or heavy deps inside the function that needs them.</pattern>
    <pattern id="7">**`TYPE_CHECKING`** — for type-only imports and circular dependency breaking.</pattern>
    <pattern id="8">**Immutable defaults** — never use mutable default arguments. Use `Final` singletons.</pattern>
    <pattern id="9">**Absolute imports** — no relative imports, ever.</pattern>
    <pattern id="10">**Minimal `__init__.py`** — don't re-export the world. Let users import from the source module.</pattern>
    <pattern id="11">**Test observable behavior** — assert on what callers see, not internal mechanism. Tests should survive implementation rewrites that preserve behavior.</pattern>
    <pattern id="12">**No tautological tests** — if the assertion can't fail given the code under test, it's not catching a regression. Don't write it.</pattern>
    <pattern id="13">**Test the longest path** — drop subset tests when a superset test on the same code path already covers the invariant.</pattern>
  </key_patterns>
</python_coding_standards>

# Pydantic v2 Cheat Sheets (2026)

A practical reference collection for current Pydantic v2 workflows. All cheat sheets are PNG images and are embedded below so they display directly on GitHub.

## 1. Core Pydantic v2

![Pydantic v2 core cheat sheet](./pydantic-v2-cheatsheet-2026.png)

[Open / view full-size PNG](./pydantic-v2-cheatsheet-2026.png)

Covers BaseModel, Field, ConfigDict, validation, custom validators, serialization, TypeAdapter, nested models, settings and v1-to-v2 replacements.

## 2. Pydantic v2 Across Frameworks

![Pydantic v2 integrations cheat sheet](./pydantic-v2-integrations-cheatsheet-2026.png)

[Open / view full-size PNG](./pydantic-v2-integrations-cheatsheet-2026.png)

Covers:
- LangChain structured model output, agents and tool input schemas
- LangGraph StateGraph and state-schema choices
- FastAPI request validation and response models
- SQLAlchemy 2.x ORM-to-DTO conversion
- MongoDB document aliases and serialization
- Pandas / ETL validation with TypeAdapter

## 3. All Pydantic v2 Decorators

![Pydantic v2 all decorators cheat sheet](./pydantic-v2-all-decorators-2026.png)

[Open / view full-size PNG](./pydantic-v2-all-decorators-2026.png)

Covers `@field_validator`, `@model_validator`, `@field_serializer`, `@model_serializer`, `@computed_field`, `@validate_call`, and Pydantic `@dataclass`. It also explains the `Annotated` validator and serializer helpers and highlights deprecated v1 names.

## Important framework distinctions

- LangGraph supports Pydantic state schemas, but its documentation recommends `TypedDict` or dataclasses when performance is the priority. LangChain `create_agent` state schemas are TypedDict-based.
- Use a Pydantic model for an agent's structured response via `response_format`; the parsed response is returned in `structured_response`.
- Use separate request and response DTOs in APIs. Do not return credentials or private fields from an input schema.
- Pydantic validation does not replace database constraints, transactions, authorization, or business logic.

## Official documentation

- [Pydantic models](https://pydantic.dev/docs/validation/latest/concepts/models/)
- [Pydantic validators](https://pydantic.dev/docs/validation/latest/concepts/validators/)
- [Pydantic serialization](https://pydantic.dev/docs/validation/latest/concepts/serialization/)
- [Pydantic migration guide](https://pydantic.dev/docs/validation/latest/get-started/migration/)
- [LangChain structured output](https://docs.langchain.com/oss/python/langchain/structured-output)
- [LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [FastAPI response models](https://fastapi.tiangolo.com/tutorial/response-model/)
- [SQLAlchemy ORM](https://docs.sqlalchemy.org/en/20/orm/)

Examples target current Pydantic v2 patterns. Confirm installed package versions and provider/framework support before relying on version-specific behavior.

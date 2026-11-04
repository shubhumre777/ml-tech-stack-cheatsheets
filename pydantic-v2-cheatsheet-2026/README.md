# Pydantic v2 Cheat Sheet (2026)

A practical, white-background reference for current Pydantic v2 workflows.

## Topics
- BaseModel, Field, ConfigDict, typed constraints and reusable Annotated types
- Input validation: model_validate, model_validate_json, model_validate_strings
- field_validator and model_validator
- Serialization: model_dump, model_dump_json, field_serializer, computed_field
- TypeAdapter and validate_call
- Nested models and discriminated unions
- Environment configuration with the separate pydantic-settings package
- V1-to-V2 API replacements to avoid in new projects

## Files
- [Open the scalable SVG cheat sheet](./pydantic-v2-cheatsheet-2026.svg)

## Official references
- [Pydantic documentation](https://pydantic.dev/docs/validation/latest/)
- [Models](https://pydantic.dev/docs/validation/latest/concepts/models/)
- [Validators](https://pydantic.dev/docs/validation/latest/concepts/validators/)
- [Serialization](https://pydantic.dev/docs/validation/latest/concepts/serialization/)
- [Configuration](https://pydantic.dev/docs/validation/latest/concepts/config/)
- [Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)

The cheat sheet focuses on Pydantic v2 APIs. Check your installed package version before relying on version-specific behavior.

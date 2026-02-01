# Custom Target Pattern

Complete pattern for creating a Pants plugin with custom target types.

## Overview

Custom targets allow you to:
1. Define new BUILD file syntax for your domain
2. Attach metadata to files and directories
3. Enable rules to process your specific target types
4. Integrate with the Pants dependency graph

## Complete Implementation

### targets.py

```python
"""Custom target types for a documentation plugin."""

from pants.engine.target import (
    COMMON_TARGET_FIELDS,
    StringField,
    StringSequenceField,
    BoolField,
    IntField,
    DictStringToStringField,
    Target,
    Dependencies,
)


# =============================================================================
# Field Definitions
# =============================================================================

class DocSourcesField(StringSequenceField):
    """Markdown source files for documentation."""

    alias = "sources"
    help = "Markdown files to include in documentation"
    default = ("*.md",)


class DocTitleField(StringField):
    """Title for the documentation section."""

    alias = "title"
    help = "Title displayed in the documentation"
    required = True  # User must provide this


class DocOutputFormatField(StringField):
    """Output format for generated documentation."""

    alias = "output_format"
    help = "Output format: html, pdf, or epub"
    default = "html"

    # Validate allowed values
    valid_choices = ("html", "pdf", "epub")


class DocThemeField(StringField):
    """Theme for the documentation."""

    alias = "theme"
    help = "Visual theme for the documentation"
    default = "default"


class DocIncludeTocField(BoolField):
    """Whether to include a table of contents."""

    alias = "include_toc"
    help = "Generate a table of contents"
    default = True


class DocMaxDepthField(IntField):
    """Maximum depth for table of contents."""

    alias = "toc_depth"
    help = "Maximum heading depth for TOC (1-6)"
    default = 3


class DocMetadataField(DictStringToStringField):
    """Metadata for the documentation."""

    alias = "metadata"
    help = "Key-value metadata: author, version, etc."


class DocDependenciesField(Dependencies):
    """Dependencies on other documentation targets."""

    # Uses Pants' built-in dependency resolution


# =============================================================================
# Target Definitions
# =============================================================================

class DocumentationTarget(Target):
    """A documentation target built from markdown sources.

    Example:
        documentation(
            name="user-guide",
            title="User Guide",
            sources=["*.md"],
            output_format="html",
            theme="modern",
            include_toc=True,
            metadata={
                "author": "Documentation Team",
                "version": "1.0",
            },
            dependencies=[":api-docs"],
        )
    """

    alias = "documentation"
    help = "Documentation built from markdown sources"

    core_fields = (
        *COMMON_TARGET_FIELDS,
        DocSourcesField,
        DocTitleField,
        DocOutputFormatField,
        DocThemeField,
        DocIncludeTocField,
        DocMaxDepthField,
        DocMetadataField,
        DocDependenciesField,
    )


class DocChapterTarget(Target):
    """A single chapter within documentation.

    Example:
        doc_chapter(
            name="getting-started",
            title="Getting Started",
            sources=["getting-started.md"],
        )
    """

    alias = "doc_chapter"
    help = "A single documentation chapter"

    core_fields = (
        *COMMON_TARGET_FIELDS,
        DocSourcesField,
        DocTitleField,
    )


class DocImageTarget(Target):
    """Image assets for documentation.

    Example:
        doc_images(
            name="screenshots",
            sources=["*.png", "*.jpg"],
        )
    """

    alias = "doc_images"
    help = "Image assets for documentation"

    core_fields = (
        *COMMON_TARGET_FIELDS,
        StringSequenceField.mixin(alias="sources", default=("*.png", "*.jpg", "*.svg")),
    )
```

### rules.py

```python
"""Rules for processing custom targets."""

from dataclasses import dataclass
from typing import Sequence

from pants.engine.rules import collect_rules, rule, Get
from pants.engine.target import (
    FieldSet,
    Target,
    Targets,
    TransitiveTargets,
    TransitiveTargetsRequest,
)
from pants.engine.fs import Digest, PathGlobs, MergeDigests

from my_docs.targets import (
    DocumentationTarget,
    DocSourcesField,
    DocTitleField,
    DocOutputFormatField,
    DocThemeField,
    DocIncludeTocField,
    DocMetadataField,
)


@dataclass(frozen=True)
class DocFieldSet(FieldSet):
    """FieldSet for documentation targets."""

    required_fields = (DocSourcesField, DocTitleField)

    sources: DocSourcesField
    title: DocTitleField
    output_format: DocOutputFormatField
    theme: DocThemeField
    include_toc: DocIncludeTocField
    metadata: DocMetadataField


@dataclass(frozen=True)
class BuiltDocumentation:
    """Result of building documentation."""

    title: str
    output_format: str
    output_digest: Digest
    output_files: tuple[str, ...]


@rule
async def build_documentation(
    fieldset: DocFieldSet,
) -> BuiltDocumentation:
    """Build documentation from a target."""

    # Get source files
    sources_digest = await Get(
        Digest,
        PathGlobs(fieldset.sources.value),
    )

    # In a real implementation, you would:
    # 1. Run a documentation generator (mdbook, sphinx, etc.)
    # 2. Apply the theme
    # 3. Generate output in the specified format

    return BuiltDocumentation(
        title=fieldset.title.value,
        output_format=fieldset.output_format.value,
        output_digest=sources_digest,
        output_files=tuple(fieldset.sources.value),
    )


@rule
async def get_all_doc_sources(
    target: DocumentationTarget,
) -> Digest:
    """Get all sources including transitive dependencies."""

    # Get transitive dependencies
    transitive = await Get(
        TransitiveTargets,
        TransitiveTargetsRequest([target.address]),
    )

    # Collect all source digests
    source_digests = []
    for dep in transitive.closure:
        if dep.has_field(DocSourcesField):
            digest = await Get(
                Digest,
                PathGlobs(dep[DocSourcesField].value),
            )
            source_digests.append(digest)

    # Merge all sources
    merged = await Get(Digest, MergeDigests(source_digests))

    return merged


def rules() -> Sequence:
    return collect_rules()
```

### Using Targets in Rules

```python
"""Accessing target fields in rules."""

from pants.engine.target import WrappedTarget, Target

@rule
async def process_any_target(wrapped: WrappedTarget) -> Output:
    """Process any target that has certain fields."""

    target = wrapped.target

    # Check target type
    if isinstance(target, DocumentationTarget):
        # Access fields directly
        title = target[DocTitleField].value
        sources = target[DocSourcesField].value

    # Check if target has a field
    if target.has_field(DocThemeField):
        theme = target[DocThemeField].value

    # Get field with fallback
    metadata = target.get(DocMetadataField)
    if metadata:
        author = metadata.value.get("author", "Unknown")

    return Output(...)
```

### register.py

```python
"""Register custom targets."""

from typing import Sequence

from pants.engine.rules import collect_rules

from my_docs import rules as doc_rules
from my_docs.targets import (
    DocumentationTarget,
    DocChapterTarget,
    DocImageTarget,
)


def rules() -> Sequence:
    return collect_rules(doc_rules)


def target_types() -> Sequence:
    return [
        DocumentationTarget,
        DocChapterTarget,
        DocImageTarget,
    ]
```

## Usage

### BUILD file

```python
# Main documentation
documentation(
    name="user-guide",
    title="User Guide",
    sources=["*.md"],
    output_format="html",
    theme="modern",
    include_toc=True,
    toc_depth=3,
    metadata={
        "author": "Docs Team",
        "version": "2.0",
    },
    dependencies=[
        ":getting-started",
        ":api-reference",
        ":images",
    ],
)

# Chapter
doc_chapter(
    name="getting-started",
    title="Getting Started",
    sources=["getting-started.md"],
)

# Images
doc_images(
    name="images",
    sources=["screenshots/*.png"],
)
```

## Field Types Reference

| Field Type | Description | Example |
|------------|-------------|---------|
| `StringField` | Single string | `"value"` |
| `StringSequenceField` | List of strings | `["a", "b"]` |
| `BoolField` | Boolean | `True` |
| `IntField` | Integer | `42` |
| `DictStringToStringField` | String dict | `{"key": "val"}` |
| `Dependencies` | Target refs | `[":other"]` |

## Key Points

1. **Fields are typed** - Use appropriate field type for your data
2. **required=True** - Make fields mandatory when needed
3. **default** - Provide sensible defaults for optional fields
4. **valid_choices** - Restrict values to valid options
5. **Dependencies** - Use for target relationships
6. **COMMON_TARGET_FIELDS** - Always include for name, tags, etc.

## Testing

```python
from pants.testutil.rule_runner import RuleRunner

def test_custom_target(rule_runner):
    rule_runner.write_files({
        "BUILD": '''
documentation(
    name="test-docs",
    title="Test Documentation",
    sources=["README.md"],
)
        ''',
        "README.md": "# Test",
    })

    target = rule_runner.get_target(Address("", target_name="test-docs"))

    assert isinstance(target, DocumentationTarget)
    assert target[DocTitleField].value == "Test Documentation"
```

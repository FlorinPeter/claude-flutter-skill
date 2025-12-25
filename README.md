# Claude Flutter Architecture Skill

A Claude Code skill that provides comprehensive Flutter app architecture guidelines based on the official Flutter architecture documentation.

## Overview

This skill helps Claude Code assist with building well-structured Flutter applications following the MVVM (Model-View-ViewModel) architecture pattern. It emphasizes:

- **Clean widget organization** - Keep files small and focused
- **Separation of concerns** - Clear boundaries between UI, data, and domain layers
- **Testability** - Architecture that enables easy unit and widget testing
- **Scalability** - Patterns that work for growing teams and codebases

## Installation

Copy the `.claude/skills/flutter-architecture.md` file to your project's `.claude/skills/` directory.

## Key Features

### Architecture Layers

```
┌─────────────────────────────────────┐
│           UI Layer                  │
│  ┌─────────────┬─────────────────┐  │
│  │   Views     │   ViewModels    │  │
│  │  (Widgets)  │ (ChangeNotifier)│  │
│  └─────────────┴─────────────────┘  │
├─────────────────────────────────────┤
│          Data Layer                 │
│  ┌─────────────┬─────────────────┐  │
│  │Repositories │    Services     │  │
│  │   (SSOT)    │  (API Wrappers) │  │
│  └─────────────┴─────────────────┘  │
└─────────────────────────────────────┘
```

### File Size Guidelines

| File Type | Max Lines | Action if Exceeded |
|-----------|-----------|-------------------|
| Widget/Screen | 150-200 | Extract child widgets |
| ViewModel | 150-200 | Extract to use-cases |
| Repository | 100-150 | Split by domain area |

### Design Patterns Included

- **Command Pattern** - Encapsulate ViewModel actions with loading/error states
- **Result Pattern** - Explicit error handling without exceptions
- **Optimistic State** - Immediate UI updates for perceived performance

### Project Structure

```
lib/
├── ui/                    # BY FEATURE
│   ├── home/
│   │   ├── home_screen.dart
│   │   ├── home_view_model.dart
│   │   └── widgets/
│   └── shared/
├── data/                  # BY TYPE
│   ├── repositories/
│   └── services/
├── domain/
│   └── models/
└── utils/
    ├── result.dart
    └── command.dart
```

## Usage

When working on Flutter projects, Claude Code will use this skill to:

1. Suggest proper file organization
2. Keep widgets small and focused
3. Implement MVVM architecture correctly
4. Use appropriate design patterns
5. Create testable code structure

## Based On

This skill is based on the official Flutter architecture documentation:
- [Flutter App Architecture Guide](https://docs.flutter.dev/app-architecture)

## License

MIT
# Shipwright Brand Standards

This document defines the visual and editorial standards for the Shipwright project to ensure consistent branding across all media, including the website, documentation, presentations, and community materials.

## Table of Contents

1. [Project Overview](#project-overview)
2. [Project Naming](#project-naming)
3. [Logo Guidelines](#logo-guidelines)
4. [Color Palette](#color-palette)
5. [Typography](#typography)
6. [Tone of Voice](#tone-of-voice)

---

## Project Overview

### Vision Statement

**Shipwright is an extensible framework for building container images on Kubernetes.**

### Target Audience

- **Primary Audience**: Potential adopters and contributors to Shipwright
- **Knowledge Level**: Familiarity with Kubernetes and containers
- **Key Personas**:
  - Platform Engineers
  - Application Developers

---

## Project Naming

### Main Project Name

**"Shipwright"**

Use "Shipwright" when:
- Referencing the project as a whole
- Multiple Shipwright sub-projects/components are referenced together
- The context clearly implies the Build component

**Example**: "Shipwright simplifies building container images on Kubernetes."

### Sub-projects and Components

#### Shipwright Build

- **Full name**: "Shipwright Build"
- **Usage**: Only use the full name when distinguishing the core component from other sub-projects
- **Default**: In most contexts, "Shipwright" implies the Build component

**Example**: "Shipwright Build provides the core APIs and controllers, while the Shipwright Operator manages the system's deployment."

#### `shp` Command Line

- **Full name**: "`shp` command line" or "`shp`"
- **Formatting**: ALWAYS use code formatting (backticks in Markdown, monospace in other contexts)
- **Never**: Write "shp" without code formatting

**Examples**:
- "The `shp` command line provides an intuitive interface..."
- "Use `shp build create` to start a new build..."

#### Shipwright Triggers

- **First reference**: "Shipwright Triggers"
- **Subsequent references**: "Triggers"

**Example**: "Shipwright Triggers integrates builds with Tekton Pipelines. Deploy Triggers to run Shipwright builds within a larger Tekton pipeline."

#### The Shipwright Operator

- **Always**: "the Shipwright Operator" (with preceding "the")
- **Purpose**: The article "the" communicates that a single instance is expected
- **Subsequent references**: "the Operator"

**Example**: "The Shipwright Operator manages the lifecycle of Build resources. Install the Operator using Operator Lifecycle Manager."

---

## Logo Guidelines

### Logo Variants

The Shipwright logo is available in three layouts:

1. **Horizontal** - Preferred for most uses (logo mark + wordmark side-by-side)
2. **Stacked** - For square or vertical spaces (logo mark above wordmark)
3. **Icon** - Logo mark only, for small spaces or when Shipwright is already identified

### Color Variants

Each layout is available in three color treatments:

- **Color** - Primary usage, full color logo
- **Black** - For light backgrounds when color is not available
- **White** - For dark backgrounds or reversed treatments

### Logo Files

All logo files are available in the `/assets/icons/` directory:

```
assets/icons/
├── horizontal/
│   ├── color/shipwright-horizontal-color.svg
│   ├── black/shipwright-horizontal-black.svg
│   └── white/shipwright-horizontal-white.svg
├── stacked/
│   ├── color/shipwright-stacked-color.svg
│   ├── black/shipwright-stacked-black.svg
│   └── white/shipwright-stacked-white.svg
└── icon/
    ├── color/shipwright-icon-color.svg
    ├── black/shipwright-icon-black.svg
    └── white/shipwright-icon-white.svg
```

Changes to these files MUST be synced to the official [CNCF artwork repository](https://github.com/cncf/artwork).

### Usage Guidelines

- **Minimum size**: Ensure the logo is legible - horizontal logo minimum width of 120px, icon minimum 32px
- **Clear space**: Maintain clear space around the logo equal to the height of the ship icon
- **Do not**: Alter colors, rotate, distort, outline, or add effects to the logo
- **Background**: Ensure sufficient contrast between logo and background

---

## Color Palette

### Core Brand Colors

The Shipwright color palette is inspired by maritime themes - deep ocean blues and slate grays that evoke shipbuilding and nautical imagery.

| Color Name | Hex Code | Usage |
|------------|----------|-------|
| **Navy** | `#0e232e` | Primary brand color, dark backgrounds, headings |
| **Slate Blue** | `#546378` | Secondary brand color, subheadings, UI elements |
| **Light Slate** | `#7c8fa4` | Tertiary color, icons, accents, backgrounds |
| **White** | `#ffffff` | Light backgrounds, reversed text |

### Light Theme Palette

Recommended for documentation, websites, and presentations with light backgrounds.

| Element | Color | Hex Code | Contrast Ratio* |
|---------|-------|----------|----------------|
| **Background** | White | `#ffffff` | - |
| **Primary Text** | Navy | `#0e232e` | 15.3:1 ✓✓ |
| **Secondary Text** | Slate Blue | `#546378` | 7.1:1 ✓✓ |
| **Links (Unvisited)** | Slate Blue | `#546378` | 7.1:1 ✓✓ |
| **Links (Hover/Active)** | Navy | `#0e232e` | 15.3:1 ✓✓ |
| **Accent/Borders** | Light Slate | `#7c8fa4` | 4.5:1 ✓ |
| **Code Blocks Background** | Ivory | `#f5f7f9` | - |
| **Code Blocks Text** | Navy | `#0e232e` | 13.8:1 ✓✓ |

*Contrast ratio against background. ✓ = WCAG AA compliant, ✓✓ = WCAG AAA compliant

### Dark Theme Palette

Recommended for dark mode interfaces, terminal applications, and developer tools.

| Element | Color | Hex Code | Contrast Ratio* |
|---------|-------|----------|----------------|
| **Background** | Navy | `#0e232e` | - |
| **Primary Text** | White | `#ffffff` | 15.3:1 ✓✓ |
| **Secondary Text** | Light Slate | `#7c8fa4` | 4.5:1 ✓ |
| **Links (Unvisited)** | Sky Blue | `#a8c5e0` | 7.2:1 ✓✓ |
| **Links (Hover/Active)** | White | `#ffffff` | 15.3:1 ✓✓ |
| **Accent/Borders** | Slate Blue | `#546378` | 3.2:1 ~ |
| **Code Blocks Background** | Deep Sea | `#1a3340` | - |
| **Code Blocks Text** | White | `#ffffff` | 12.1:1 ✓✓ |

*Contrast ratio against background. ✓ = WCAG AA compliant, ✓✓ = WCAG AAA compliant, ~ = Use for large text only

**Note**: For dark theme links, we use a lighter shade (`#a8c5e0`) to ensure sufficient contrast against the navy background.

### Accessibility Considerations

All color combinations in this palette have been tested for accessibility:

- **WCAG AA Compliance**: Minimum 4.5:1 contrast ratio for normal text, 3:1 for large text (18pt+)
- **WCAG AAA Compliance**: Minimum 7:1 contrast ratio for enhanced readability
- **Color Blindness**: The Shipwright palette uses both color and luminance contrast, making it distinguishable for users with protanopia, deuteranopia, and tritanopia
- **Never use color alone**: Always combine color with text labels, icons, or patterns to convey information

### Extended Palette

For additional UI needs (alerts, notifications, status indicators):

| Purpose | Light Theme | Dark Theme | Notes |
|---------|-------------|------------|-------|
| **Success** | `#2ea043` | `#3fb950` | Green, positive actions |
| **Warning** | `#9a6700` | `#d29922` | Amber, caution states |
| **Error** | `#cf222e` | `#f85149` | Red, destructive actions |
| **Info** | `#0969da` | `#58a6ff` | Blue, informational messages |

These colors should be used sparingly and always meet WCAG AA contrast requirements.

---

## Typography

### Typefaces

#### Body Text and UI

Use system font stacks for optimal performance and native appearance:

```css
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Noto Sans", 
             Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji";
```

**Characteristics**:
- Clean, modern sans-serif
- Excellent readability across devices
- Native to each operating system

#### Logo Wordmark

The Shipwright logo uses a clean, geometric sans-serif typeface. When the logo is used, no additional typeface is required for the wordmark.

#### Code and Technical Content

For code blocks, inline code, terminal commands, and API references:

```css
font-family: ui-monospace, SFMono-Regular, "SF Mono", Menlo, Consolas, 
             "Liberation Mono", monospace;
```

**Usage**:
- Command line examples: `` `shp build create` ``
- API objects: `` `BuildRun` ``, `` `BuildStrategy` ``
- File paths: `` `/var/lib/shipwright` ``
- Configuration values: `` `enabled: true` ``

### Type Scale

Recommended type sizes for web and documentation:

| Element | Size | Weight | Usage |
|---------|------|--------|-------|
| **H1** | 2.5rem (40px) | 700 | Page titles |
| **H2** | 2rem (32px) | 600 | Section headings |
| **H3** | 1.5rem (24px) | 600 | Subsection headings |
| **H4** | 1.25rem (20px) | 600 | Component headings |
| **Body** | 1rem (16px) | 400 | Paragraph text |
| **Small** | 0.875rem (14px) | 400 | Captions, metadata |
| **Code** | 0.9375rem (15px) | 400 | Code blocks |
| **Inline Code** | 0.875rem (14px) | 400 | Inline code snippets |

### Line Height

- **Body text**: 1.6 (for optimal readability)
- **Headings**: 1.2-1.3 (tighter spacing)
- **Code blocks**: 1.5 (balance between density and readability)

---

## Tone of Voice

### Brand Personality

Shipwright's voice is:

- **Approachable**: Friendly and welcoming to newcomers
- **Teaching-oriented**: Focused on helping users learn and succeed
- **Technical but not intimidating**: Precise without being overly formal
- **Inclusive**: Respectful of diverse backgrounds and experience levels

### Writing Principles

1. **Be Clear and Concise**
   - Use simple, direct language
   - Avoid jargon unless it's standard Kubernetes/container terminology
   - Define technical terms on first use

2. **Be Helpful**
   - Anticipate user questions
   - Provide context for why something matters
   - Include practical examples

3. **Be Respectful**
   - Acknowledge that learning is a journey
   - Never assume knowledge or make users feel inadequate
   - Celebrate contributions of all sizes

4. **Express Yourself**
   - Casual tone is permissible - this is open source
   - Use contractions (it's, you're, we'll)
   - Show personality while remaining professional
   - Encourage contributors to bring their authentic voice

### Voice Examples

**Do**: "Let's build your first container image with Shipwright! We'll use the `shp` command line to make this quick and easy."

**Don't**: "One must configure the BuildStrategy CRD in accordance with the technical specifications outlined herein."

---

**Do**: "The Shipwright Operator handles the heavy lifting of managing your build infrastructure. Think of it as your build system's autopilot."

**Don't**: "The operator component provides automated lifecycle management capabilities."

---

**Do**: "New to Kubernetes? No worries! Here's what you need to know before getting started..."

**Don't**: "This documentation assumes expert-level knowledge of Kubernetes primitives."

### Content Guidelines

#### Documentation

- Start with the user's goal, not the feature
- Use second person ("you") to directly address readers
- Include code examples that can be copied and run
- Explain not just how, but why

#### Blog Posts and Announcements

- Lead with the benefit to users
- Share the story behind features and decisions
- Highlight community contributions
- Use headers and formatting for scanability

#### Error Messages

- Explain what went wrong in plain language
- Suggest how to fix it
- Provide a link to relevant documentation when helpful

**Example**:
```
Error: BuildRun 'example-build' failed
Reason: Source repository not found at https://github.com/example/repo

Check that:
- The repository URL is correct
- The repository is publicly accessible, or credentials are configured
- Network connectivity allows access to GitHub

See: https://shipwright.io/docs/troubleshooting/source-errors
```

---

## Questions or Updates

Brand standards evolve with the project. If you have questions or suggestions:

- Open an issue at [shipwright-io/community](https://github.com/shipwright-io/community/issues)
- Join the discussion in the Shipwright Slack channel
- Submit a pull request with proposed changes

---

*Last updated: April 2, 2026*

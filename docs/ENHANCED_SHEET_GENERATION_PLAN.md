# Enhanced Sheet Generation with LLM-Powered Pose Variants

## Overview

This document outlines the plan for enhancing the sheet generation logic to support user-provided pose descriptions with LLM-generated variants.

## Current State

The existing sheet generation logic:
1. Accepts a set of default poses
2. Generates a sheet image containing multiple poses
3. Processes with green removal (for Gemini-generated images)
4. Cuts individual stickers from the sheet

## Proposed Enhancement

Add a new workflow that allows users to provide custom pose descriptions, which are then expanded into 9 total poses (1 original + 8 variants) via LLM.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER INPUT                                    │
│                                                                      │
│   "A cat sitting with one paw raised, looking curious"              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   POSE VARIANT GENERATOR                             │
│                                                                      │
│   LLM Service (OpenAI/Claude/Gemini)                                │
│   Prompt: "Generate 8 subtle variations of this pose description"   │
│                                                                      │
│   Output: 8 variant descriptions                                    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    POSE COLLECTION (9 total)                         │
│                                                                      │
│   [0] Original: "A cat sitting with one paw raised, looking curious"│
│   [1] Variant:  "A cat sitting with one paw slightly lifted..."     │
│   [2] Variant:  "A cat perched upright, paw gently extended..."     │
│   [3] Variant:  "A seated cat with paw hovering mid-air..."         │
│   [4] Variant:  "A cat in sitting pose, one front paw raised..."    │
│   [5] Variant:  "A curious cat sitting, delicately lifting paw..."  │
│   [6] Variant:  "A cat seated attentively, paw poised upward..."    │
│   [7] Variant:  "A sitting feline with raised paw, head tilted..."  │
│   [8] Variant:  "A cat in upright sit, one paw extended forward..." │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    SHEET GENERATOR                                   │
│                                                                      │
│   Input: 9 pose descriptions + character/style context              │
│   Image Gen: Gemini Imagen / DALL-E / Stable Diffusion              │
│   Output: Single sheet image with 9 poses arranged in 3x3 grid      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    POST-PROCESSING PIPELINE                          │
│                                                                      │
│   ┌─────────────────┐    ┌─────────────────┐    ┌────────────────┐ │
│   │ Green Removal   │ -> │ Sheet Cutting   │ -> │ Individual     │ │
│   │ (if Gemini)     │    │ (9 segments)    │    │ Stickers       │ │
│   └─────────────────┘    └─────────────────┘    └────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         OUTPUT                                       │
│                                                                      │
│   - Full processed sheet image                                      │
│   - 9 individual sticker images                                     │
│   - Metadata (pose descriptions, generation params)                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Component Design

### 1. Pose Variant Generator Service

**Purpose:** Generate subtle variations of user-provided pose descriptions using an LLM.

**Interface:**
```typescript
interface PoseVariantGeneratorConfig {
  llmProvider: 'openai' | 'anthropic' | 'gemini';
  model: string;
  temperature: number;  // Recommend 0.7-0.9 for creative variation
  variantCount: number; // Default: 8
}

interface PoseVariantRequest {
  originalPose: string;
  characterContext?: string;  // e.g., "cute cartoon cat"
  styleGuidance?: string;     // e.g., "kawaii style, simple shapes"
}

interface PoseVariantResponse {
  original: string;
  variants: string[];
  allPoses: string[];  // original + variants combined
}
```

**LLM Prompt Template:**
```
You are a creative assistant helping generate pose variations for character art.

Given the following pose description, generate {variantCount} subtle variations.
Each variation should:
- Maintain the core essence and action of the original pose
- Introduce small, natural modifications (angle, expression, limb position)
- Be distinct enough to create visual variety
- Remain feasible and natural for the character type

Original pose: "{originalPose}"
{characterContext ? `Character: ${characterContext}` : ''}
{styleGuidance ? `Style: ${styleGuidance}` : ''}

Respond with exactly {variantCount} variations, one per line, numbered 1-{variantCount}.
Keep each description concise (1-2 sentences).
```

**Implementation Notes:**
- Use structured output (JSON mode) if available for reliability
- Implement retry logic for malformed responses
- Cache results for identical inputs to reduce API costs
- Consider adding a "variation intensity" parameter (subtle vs. dramatic)

---

### 2. Enhanced Sheet Generation Service

**Purpose:** Generate a single sheet image containing multiple poses.

**Modifications to Existing Logic:**

```typescript
interface SheetGenerationRequest {
  // Existing parameters
  characterDescription: string;
  stylePrompt: string;
  imageProvider: 'gemini' | 'dalle' | 'stable-diffusion';

  // NEW: Custom pose mode
  poseMode: 'default' | 'custom';

  // For custom mode
  customPoseInput?: {
    userPoseDescription: string;
    generateVariants: boolean;  // If true, generates 8 variants
    variantConfig?: PoseVariantGeneratorConfig;
  };

  // For default mode (existing)
  defaultPoses?: string[];

  // Sheet layout
  gridLayout: { rows: number; cols: number };  // Default: 3x3 for 9 poses
}

interface SheetGenerationResponse {
  sheetImageUrl: string;
  sheetImageBase64?: string;
  poses: {
    index: number;
    description: string;
    isOriginal: boolean;  // true for user's original pose
  }[];
  processingMetadata: {
    greenRemovalApplied: boolean;
    provider: string;
    generationTime: number;
  };
}
```

**Sheet Composition Prompt Template:**
```
Create a character sheet with exactly 9 poses arranged in a 3x3 grid.

Character: {characterDescription}
Style: {stylePrompt}

Poses (left to right, top to bottom):
1. {poses[0]}
2. {poses[1]}
3. {poses[2]}
4. {poses[3]}
5. {poses[4]}
6. {poses[5]}
7. {poses[6]}
8. {poses[7]}
9. {poses[8]}

Requirements:
- Each pose should be clearly separated with consistent spacing
- Maintain consistent character appearance across all poses
- Use a solid green background (#00FF00) for easy removal
- Each pose should fit within its grid cell with padding
- Character should be centered within each cell
```

---

### 3. Processing Pipeline

**Step 1: Green Removal (Conditional)**

```typescript
interface GreenRemovalConfig {
  enabled: boolean;
  tolerance: number;        // Color matching tolerance (0-255)
  edgeSmoothing: boolean;   // Anti-alias edges
  spillCorrection: boolean; // Remove green color spill on edges
}

function shouldApplyGreenRemoval(provider: string): boolean {
  // Gemini tends to generate better with green screen approach
  return provider === 'gemini';
}
```

**Step 2: Sheet Cutting**

```typescript
interface CuttingConfig {
  gridLayout: { rows: number; cols: number };
  padding: number;           // Pixels to trim from each cell edge
  outputFormat: 'png' | 'webp';
  transparentBackground: boolean;
}

interface CutResult {
  index: number;
  imageData: Buffer;
  poseDescription: string;
  bounds: { x: number; y: number; width: number; height: number };
}

function cutSheet(
  sheetImage: Buffer,
  config: CuttingConfig
): CutResult[] {
  // Calculate cell dimensions
  // Extract each cell
  // Apply any per-cell processing
  // Return array of individual stickers
}
```

---

## Data Flow

### Request Flow

```
1. User submits pose description
   └─> Validate input (length, content safety)

2. Generate pose variants (if enabled)
   └─> Call LLM service
   └─> Parse and validate 8 variants
   └─> Combine with original (total: 9)

3. Build sheet generation prompt
   └─> Inject poses into template
   └─> Add character/style context

4. Generate sheet image
   └─> Call image generation API
   └─> Receive raw sheet image

5. Post-process sheet
   └─> Green removal (if applicable)
   └─> Quality validation

6. Cut into individual stickers
   └─> Calculate grid positions
   └─> Extract 9 individual images
   └─> Apply per-sticker processing

7. Return results
   └─> Sheet image
   └─> Individual stickers
   └─> Metadata
```

---

## API Design

### Endpoint: Generate Custom Pose Sheet

```
POST /api/v1/sheets/generate-custom
```

**Request Body:**
```json
{
  "characterDescription": "A cute orange tabby cat with big eyes",
  "stylePrompt": "kawaii anime style, soft colors, simple shapes",
  "poseDescription": "sitting with one paw raised, looking curious",
  "generateVariants": true,
  "variantConfig": {
    "llmProvider": "anthropic",
    "model": "claude-3-haiku-20240307",
    "temperature": 0.8
  },
  "imageProvider": "gemini",
  "outputFormat": "png",
  "includeSheet": true,
  "includeIndividual": true
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "sheetId": "sheet_abc123",
    "sheetImage": {
      "url": "https://storage.example.com/sheets/sheet_abc123.png",
      "width": 1536,
      "height": 1536
    },
    "stickers": [
      {
        "index": 0,
        "isOriginal": true,
        "poseDescription": "sitting with one paw raised, looking curious",
        "url": "https://storage.example.com/stickers/sheet_abc123_0.png"
      },
      {
        "index": 1,
        "isOriginal": false,
        "poseDescription": "sitting with one paw slightly lifted, ears perked",
        "url": "https://storage.example.com/stickers/sheet_abc123_1.png"
      }
      // ... 7 more stickers
    ],
    "metadata": {
      "generationTime": 12500,
      "llmTokensUsed": 450,
      "imageProvider": "gemini",
      "greenRemovalApplied": true
    }
  }
}
```

---

## Implementation Plan

### Phase 1: Pose Variant Generator (Foundation)

**Tasks:**
1. Create `PoseVariantGenerator` service class
2. Implement LLM provider abstraction (support OpenAI, Anthropic, Gemini)
3. Design and test prompt templates for consistent variant generation
4. Add input validation and content safety checks
5. Implement response parsing with fallback handling
6. Add caching layer for repeated inputs
7. Write unit tests with mocked LLM responses

**Files to Create/Modify:**
```
src/
  services/
    pose-variant-generator/
      index.ts
      types.ts
      prompts.ts
      providers/
        openai.ts
        anthropic.ts
        gemini.ts
      __tests__/
        pose-variant-generator.test.ts
```

### Phase 2: Sheet Generator Enhancement

**Tasks:**
1. Extend `SheetGenerator` interface to accept custom poses
2. Add `poseMode` routing logic (default vs custom)
3. Modify prompt builder to handle 9 custom poses
4. Update grid layout logic for 3x3 arrangement
5. Ensure backward compatibility with existing default pose flow
6. Add validation for pose count matching grid layout
7. Write integration tests

**Files to Create/Modify:**
```
src/
  services/
    sheet-generator/
      index.ts           # Modify
      types.ts           # Modify
      prompt-builder.ts  # Modify
      custom-pose-handler.ts  # New
```

### Phase 3: Processing Pipeline Updates

**Tasks:**
1. Ensure green removal handles 3x3 grid correctly
2. Update cutting logic for configurable grid sizes
3. Add pose metadata to cut results
4. Implement quality checks per sticker
5. Add retry logic for failed cuts
6. Performance optimization for batch processing

**Files to Create/Modify:**
```
src/
  services/
    image-processor/
      green-removal.ts   # Modify if needed
      sheet-cutter.ts    # Modify
      types.ts           # Modify
```

### Phase 4: API Integration

**Tasks:**
1. Create new endpoint for custom pose generation
2. Add request validation middleware
3. Implement async job handling for long-running generations
4. Add progress reporting (WebSocket or polling)
5. Update API documentation
6. Rate limiting and quota management

### Phase 5: Testing & Refinement

**Tasks:**
1. End-to-end testing with various pose inputs
2. LLM output quality validation
3. Edge case handling (inappropriate content, malformed responses)
4. Performance benchmarking
5. Cost analysis and optimization
6. User feedback integration

---

## Configuration

```typescript
// config/sheet-generation.ts

export const sheetGenerationConfig = {
  poseVariants: {
    defaultCount: 8,
    maxCount: 12,
    llm: {
      defaultProvider: 'anthropic',
      defaultModel: 'claude-3-haiku-20240307',
      defaultTemperature: 0.8,
      maxRetries: 3,
      timeoutMs: 30000
    }
  },

  sheetGeneration: {
    defaultGridLayout: { rows: 3, cols: 3 },
    supportedLayouts: [
      { rows: 2, cols: 2 },  // 4 poses
      { rows: 3, cols: 3 },  // 9 poses
      { rows: 3, cols: 4 },  // 12 poses
    ],
    defaultImageSize: 1536,
    cellPadding: 20
  },

  processing: {
    greenRemoval: {
      defaultTolerance: 30,
      edgeSmoothing: true,
      spillCorrection: true
    },
    output: {
      format: 'png',
      quality: 95,
      transparentBackground: true
    }
  }
};
```

---

## Error Handling

| Error Type | Handling Strategy |
|------------|-------------------|
| LLM rate limit | Exponential backoff, queue requests |
| LLM malformed response | Retry up to 3 times, then use fallback generic variants |
| Image generation failure | Retry with simplified prompt |
| Green removal failure | Return unprocessed image with warning |
| Cutting detection failure | Use fixed grid positions as fallback |
| Content policy violation | Return error with guidance |

---

## Monitoring & Observability

**Metrics to Track:**
- Variant generation latency (p50, p95, p99)
- LLM token usage per request
- Image generation success rate by provider
- Green removal effectiveness score
- Cut accuracy validation
- End-to-end generation time
- Error rates by type

**Logging:**
- Request/response payloads (sanitized)
- LLM prompts and responses
- Processing step timings
- Error details with stack traces

---

## Future Enhancements

1. **Variation Intensity Control**: Allow users to specify how different variants should be
2. **Pose Categories**: Pre-defined categories (action, emotion, interaction)
3. **Style Transfer**: Apply consistent style across variant poses
4. **Batch Generation**: Generate multiple characters with same pose set
5. **Interactive Refinement**: Allow users to regenerate specific variants
6. **A/B Testing**: Compare different LLM providers for variant quality
7. **User Favorites**: Save and reuse successful pose sets

---

## Dependencies

**New Dependencies:**
- LLM SDK (anthropic, openai, or @google/generative-ai)
- Image processing library (sharp, jimp, or canvas)

**Existing Dependencies to Leverage:**
- Current image generation integration
- Current green removal implementation
- Current sheet cutting logic

---

## Summary

This enhancement introduces a user-driven pose customization workflow that:

1. **Accepts** a single pose description from the user
2. **Generates** 8 creative variants via LLM
3. **Produces** a 9-pose sheet (original + 8 variants)
4. **Processes** with existing green removal and cutting pipeline
5. **Returns** both the full sheet and individual stickers

The design maintains backward compatibility with existing default pose functionality while adding powerful customization capabilities.

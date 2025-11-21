# @x402/extensions

x402 Payment Protocol Extensions

## Overview

Extensions allow the x402 payment protocol to be extended with additional functionality and metadata. They provide a standardized way for resource servers to communicate capabilities, requirements, and metadata to facilitators and clients.

## What Are Extensions?

Extensions in x402 v2 follow a consistent pattern with two components:

- **`info`**: The actual data values you want to communicate
- **`schema`**: A JSON Schema that validates the structure of `info`

This pattern ensures extensions are self-describing, validatable, and interoperable across the x402 ecosystem.

## Core Extension Functions

Every extension provides three types of functions that serve different roles in the payment flow:

### Declaration Functions (Resource Servers)

**Purpose**: Create properly structured extensions to include in `PaymentRequired` responses.

**Used by**: Resource servers that want to communicate capabilities, requirements, or metadata to clients and facilitators.

**Example**: `declareDiscoveryExtension()` - Creates a Bazaar extension that describes how to call an endpoint.

**What they do**:
- Accept human-friendly parameters
- Generate both the `info` (data) and `schema` (validation rules)
- Return a properly formatted extension object ready to include in responses
- Ensure consistency between info and schema

### Validation Functions (Facilitators)

**Purpose**: Verify that extension data conforms to its declared schema.

**Used by**: Facilitators and clients that receive payment payloads with extensions.

**Example**: `validateDiscoveryExtension()` - Validates that a Bazaar extension is well-formed.

**What they do**:
- Use JSON Schema validation (via AJV) to check extension structure
- Verify that `info` matches the rules defined in `schema`
- Return validation results with detailed error messages
- Protect against malformed or malicious extension data

### Extraction Functions (Facilitators)

**Purpose**: Safely extract and use extension data from payment payloads.

**Used by**: Facilitators that need to process extension information (e.g., catalog resources, enforce policies).

**Example**: `extractDiscoveryInfo()` - Extracts discovery metadata from a payment payload.

**What they do**:
- Locate extensions within payment payloads
- Optionally validate before extraction
- Transform extension data into usable types
- Handle missing or invalid extensions gracefully
- Often combine validation and extraction in one step

## Installation

```bash
npm install @x402/extensions
```

## Usage

### For Resource Servers

Resource servers use extensions to declare capabilities in their `PaymentRequired` responses:

```typescript
import { declareDiscoveryExtension, BAZAAR } from '@x402/extensions/bazaar';

const extensions = declareDiscoveryExtension({
  input: { query: "example", limit: 10 },
  inputSchema: {
    properties: {
      query: { type: "string" },
      limit: { type: "number" }
    },
    required: ["query"]
  },
  output: {
    example: { results: [], total: 0 }
  }
});

const paymentRequired = {
  x402Version: 2,
  resource: {
    url: "/api/search",
    description: "Search endpoint",
    mimeType: "application/json"
  },
  accepts: [/* payment methods */],
  extensions
};
```

### For Facilitators

Facilitators validate and extract extension data from payment payloads:

```typescript
import { extractDiscoveryInfo, BAZAAR } from '@x402/extensions/bazaar';

const discovered = extractDiscoveryInfo(paymentPayload, paymentRequirements);

if (discovered) {
  console.log(`Discovered: ${discovered.method} ${discovered.resourceUrl}`);
  console.log('Discovery info:', discovered.discoveryInfo);
}
```

## Implementing a Custom Extension

### Step 1: Define Your Extension Structure

```typescript
// types.ts
export const MY_EXTENSION = "my-extension";

export interface MyExtensionInfo {
  feature: string;
  options?: {
    maxRetries?: number;
    timeout?: number;
  };
}

export interface MyExtension {
  info: MyExtensionInfo;
  schema: {
    $schema: "https://json-schema.org/draft/2020-12/schema";
    type: "object";
    properties: {
      feature: { type: "string" };
      options?: {
        type: "object";
        properties?: {
          maxRetries?: { type: "number" };
          timeout?: { type: "number" };
        };
      };
    };
    required: string[];
  };
}
```

### Step 2: Create Declaration Function (Resource Server)

```typescript
// resource-service.ts
export function declareMyExtension(
  feature: string,
  options?: { maxRetries?: number; timeout?: number }
): Record<string, MyExtension> {
  return {
    [MY_EXTENSION]: {
      info: {
        feature,
        ...(options ? { options } : {}),
      },
      schema: {
        $schema: "https://json-schema.org/draft/2020-12/schema",
        type: "object",
        properties: {
          feature: { type: "string" },
          ...(options ? {
            options: {
              type: "object",
              properties: {
                maxRetries: { type: "number" },
                timeout: { type: "number" },
              },
            },
          } : {}),
        },
        required: ["feature"],
      },
    },
  };
}
```

### Step 3: Create Validation Function (Facilitator)

```typescript
// facilitator.ts
import Ajv from "ajv/dist/2020";

export interface ValidationResult {
  valid: boolean;
  errors?: string[];
}

export function validateMyExtension(
  extension: MyExtension
): ValidationResult {
  try {
    const ajv = new Ajv({ strict: false, allErrors: true });
    const validate = ajv.compile(extension.schema);
    const valid = validate(extension.info);

    if (valid) {
      return { valid: true };
    }

    const errors = validate.errors?.map(err => {
      const path = err.instancePath || "(root)";
      return `${path}: ${err.message}`;
    }) || ["Unknown validation error"];

    return { valid: false, errors };
  } catch (error) {
    return {
      valid: false,
      errors: [`Schema validation failed: ${error}`],
    };
  }
}
```

### Step 4: Create Extraction Function (Facilitator)

```typescript
export function extractMyExtensionInfo(
  paymentPayload: PaymentPayload,
  validate: boolean = true
): MyExtensionInfo | null {
  if (paymentPayload.x402Version !== 2 || !paymentPayload.extensions) {
    return null;
  }

  const extension = paymentPayload.extensions[MY_EXTENSION];
  if (!extension) {
    return null;
  }

  try {
    const typedExtension = extension as MyExtension;

    if (validate) {
      const result = validateMyExtension(typedExtension);
      if (!result.valid) {
        console.warn(`Validation failed: ${result.errors?.join(", ")}`);
        return null;
      }
    }

    return typedExtension.info;
  } catch (error) {
    console.warn(`Extension extraction failed: ${error}`);
    return null;
  }
}
```

### Step 5: Export Your Extension

```typescript
// index.ts
export { MY_EXTENSION } from "./types";
export type { MyExtensionInfo, MyExtension } from "./types";
export { declareMyExtension } from "./resource-service";
export { validateMyExtension, extractMyExtensionInfo } from "./facilitator";
```

## Extension Best Practices

1. **Use clear, descriptive extension keys**: Choose unique identifiers like `"bazaar"` or `"rate-limit"`
2. **Validate on both sides**: Resource servers should create valid extensions, facilitators should validate before using
3. **Provide examples**: Include example usage in documentation
4. **Version your schemas**: Consider how your extension might evolve over time
5. **Keep info and schema in sync**: The schema should accurately describe the info structure

## Development

### Build

```bash
npm run build
```

### Test

```bash
npm run test
```

### Lint

```bash
npm run lint
```

### Format

```bash
npm run format
```

## Learn More

- [x402 Protocol Specification](https://github.com/coinbase/x402)
- [JSON Schema Documentation](https://json-schema.org/)
- [Bazaar Extension Example](./src/bazaar/)

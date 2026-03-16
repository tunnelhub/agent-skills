# Smart Logging Strategy

⚠️ **WARNING**: Customizing the logging strategy is a **very advanced** operation that should only be done under direct
supervision of a 4success TunnelHub specialist. Incorrect configuration can lead to severe performance issues, excessive
infrastructure costs, and DynamoDB throttling. Most integrations should use the default behavior without modification.

This reference explains the intelligent logging strategy in TunnelHub SDK v3.0, which automatically chooses between
real-time (DynamoDB) and batch (Firehose) logging based on data volume and integration characteristics.

## Overview

The SDK analyzes each integration execution and selects the optimal logging strategy:

- **Real-time (DynamoDB)**: For small volumes or fast-processing integrations
- **Batch (Firehose)**: For large volumes or slower integrations

This approach provides:

- **Cost Optimization**: Significant infrastructure cost reduction
- **Performance**: Eliminates 70s overhead for fast integrations
- **Reliability**: 99.9% log durability with Firehose
- **Intelligent Selection**: Adapts to integration characteristics

## Default Behavior

### Delta Flows (DeltaIntegrationFlow, BatchDeltaIntegrationFlow)

```typescript
// Default configurations
protected realtimeLoggingThreshold: number = 100;
protected maxRealtimeItems: number = 1000;
protected highNoDeltaRatioThreshold: number = 0.8; // 80%
```

**Decision Logic:**

1. **≤100 items** → Realtime logging
2. **>1000 items** → Batch logging (DynamoDB protection)
3. **100-500 items** with **≥80% noDelta ratio** → Realtime (fast integration optimization)
4. **Other cases** → Batch logging

### NoDelta Flows (NoDeltaIntegrationFlow, NoDeltaBatchIntegrationFlow)

```typescript
// Default configurations
protected realtimeLoggingThreshold: number = 100;
protected maxRealtimeItems: number = 1000;
```

**Decision Logic:**

1. **≤100 items** → Realtime logging
2. **>1000 items** → Batch logging (DynamoDB protection)
3. **100-500 items** → Realtime (assuming fast processing)
4. **>500 items** → Batch logging

## Configuration Options

> **Note**: The configurations below are provided for extreme cases. Only modify these parameters under direct
> supervision of a 4success TunnelHub specialist.

### Basic Thresholds

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected realtimeLoggingThreshold: number = 100; // Items threshold
  protected maxRealtimeItems: number = 1000; // Safety limit for DynamoDB
  protected maxRealtimeDuration: number = 30; // Estimated max seconds
}
```

**realtimeLoggingThreshold**

- **Default**: 100
- **Description**: Minimum number of items to consider batch mode
- **Lower values**: Prefer batch (cost optimization)
- **Higher values**: Prefer realtime (latency optimization)

**maxRealtimeItems**

- **Default**: 1000
- **Description**: Safety limit to protect DynamoDB from excessive writes
- **Never** set above 5000 without careful testing
- **Reason**: Prevents WCU (Write Capacity Units) exhaustion

**maxRealtimeDuration**

- **Default**: 30 seconds
- **Description**: Estimated maximum execution duration for realtime logging
- **Used for**: Decision optimization (future enhancements)

### Advanced Configuration (Delta Flows)

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected highNoDeltaRatioThreshold: number = 0.8; // 80%

  protected isKnownFastIntegration(): boolean {
    // Return true if this integration is known to be fast
    return this.executionEvent.metadata?.some(m => m.key === 'processing_speed' && m.value === 'fast');
  }
}
```

**highNoDeltaRatioThreshold**

- **Default**: 0.8 (80%)
- **Description**: Percentage of items with no changes that qualifies as "fast integration"
- **Higher values**: More restrictive (only very fast integrations get realtime)
- **Lower values**: More permissive (more integrations get realtime)

**isKnownFastIntegration()**

- **Override**: Optional
- **Purpose**: Force realtime logging even with high volumes
- **Use case**: Known fast integrations that process >1000 items quickly
- **Returns**: `true` to always use realtime, `false` to use default logic

## Customization Examples

> **Note**: The examples below are provided for extreme cases. Only apply these customizations under direct supervision
> of a 4success TunnelHub specialist.

### 1. Known Fast Integration

For integrations that are known to be fast regardless of volume:

```typescript
class FastSyncIntegration extends DeltaIntegrationFlow<Employee> {
  protected isKnownFastIntegration(): boolean {
    return true; // This integration is always fast
  }

  // Alternative: Check metadata
  protected isKnownFastIntegration(): boolean {
    return this.executionEvent.metadata?.some(m => m.key === 'processing_speed' && m.value === 'fast') || false;
  }
}
```

**When to use:**

- Integrations processing 1000-5000 items in <30 seconds
- API endpoints with very fast response times
- Low-latency database operations

### 2. Cost-Optimized Integration

Prioritize Firehose (lower cost) over DynamoDB:

```typescript
class CostOptimizedIntegration extends NoDeltaIntegrationFlow<LogEntry> {
  protected realtimeLoggingThreshold: number = 50; // Lower threshold
  protected maxRealtimeItems: number = 500; // Lower safety limit
}
```

**When to use:**

- High-volume integrations where cost is critical
- Non-critical logs where slight delay is acceptable
- Budget-constrained environments

### 3. Performance-Optimized Integration

Prioritize DynamoDB (lower latency) over Firehose:

```typescript
class PerformanceOptimizedIntegration extends DeltaIntegrationFlow<Transaction> {
  protected realtimeLoggingThreshold: number = 500;
  protected maxRealtimeItems: number = 5000;

  protected isKnownFastIntegration(): boolean {
    return this.executionEvent.metadata?.some(m => m.key === 'priority' && m.value === 'high');
  }
}
```

**When to use:**

- Real-time monitoring integrations
- Time-sensitive operations
- Critical business processes

### 4. Custom Thresholds Based on Data Type

Adjust thresholds based on expected data patterns:

```typescript
class CustomerSyncIntegration extends DeltaIntegrationFlow<Customer> {
  // Customer sync usually has high noDelta ratio (few changes)
  protected highNoDeltaRatioThreshold: number = 0.9; // 90%

  // Allow more items in realtime if noDelta is high
  protected realtimeLoggingThreshold: number = 200;
  protected maxRealtimeItems: number = 1500;
}

class TransactionSyncIntegration extends DeltaIntegrationFlow<Transaction> {
  // Transaction sync has low noDelta ratio (many new transactions)
  protected highNoDeltaRatioThreshold: number = 0.6; // 60%

  // Lower thresholds for performance
  protected realtimeLoggingThreshold: number = 100;
  protected maxRealtimeItems: number = 800;
}
```

## Use Case Examples

### Scenario 1: High-Volume Fast Integration

**Context:** ~1700 items, ~7 seconds duration, ~95% noDelta ratio

**Behavior:**

- **Default**: Batch mode (>1000 items)
- **With optimization**: Realtime mode (isKnownFastIntegration returns true)

**Result:**

- **Before**: 77 seconds total (70s overhead + 7s processing)
- **After**: 7 seconds total (no overhead)
- **Savings**: 70 seconds (91% improvement)

**Implementation:**

```typescript
class FastUserSync extends DeltaIntegrationFlow<User> {
  protected isKnownFastIntegration(): boolean {
    return true; // Known to process 1700 items in 7 seconds
  }

  // Alternative: Detect from execution
  protected isKnownFastIntegration(): boolean {
    const expectedItems = parseInt(this.parameters?.expected_items || '0');
    return expectedItems > 1000;
  }
}
```

### Scenario 2: Large Dataset Integration

**Context:** ~138,000 items, ~107 seconds duration

**Behavior:**

- **Strategy**: Batch mode (>1000 items, even with override)
- **Reason**: Firehose is designed for this scale
- **Note**: DynamoDB would be prohibitively expensive and slow

**Result:**

- Cost-effective logging at scale
- 99.9% log durability
- No WCU throttling

### Scenario 3: Medium-Volume Fast Integration

**Context:** ~140 items, ~7 seconds duration, ~90% noDelta ratio

**Behavior:**

- **Default**: Realtime mode (100-500 items with ≥80% noDelta)
- **Decision**: Automatic optimization without override

**Result:**

- Realtime logging with no overhead
- Cost-effective (under threshold)
- Immediate log visibility

### Scenario 4: Medium-Volume Slow Integration

**Context:** ~600 items, ~60 seconds duration, ~15% noDelta ratio

**Behavior:**

- **Default**: Batch mode (>500 items or <80% noDelta)
- **Reason**: Too many updates, processing is slow
- **Decision**: Optimize cost and performance

**Result:**

- Batch logging via Firehose
- Lower DynamoDB costs
- Faster overall execution

## Monitoring and Debugging

### Strategy Decision Logs

The SDK logs strategy decisions for monitoring:

```
[LogStrategy] Realtime for fast integration: 450 items (noDelta: 90%)
[LogStrategy] Batch mode: 1500 items > max 1000
[LogStrategy] Batch mode: 600 items (noDelta: 15%)
[LogStrategy] Realtime mode: 80 items < threshold 100
```

### Enabling Verbose Logging

```typescript
import {SDK} from '@tunnelhub/sdk';

// Enable verbose mode to see all strategy decisions
SDK.verbose = true;
```

### Custom Logging

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected getLoggingStrategy(): string {
    const totalItems = this.sourceSystemData.length;
    const strategy = super['getLoggingStrategy']();

    console.log(`[MyIntegration] Total items: ${totalItems}, Strategy: ${strategy}`);

    return strategy;
  }
}
```

## Performance Impact

### Resource Usage

**Realtime (DynamoDB):**

- **WCU**: 1 WCU per log entry
- **Cost**: Higher for high volumes
- **Latency**: ~5-20ms per write
- **Visibility**: Immediate

**Batch (Firehose):**

- **WCU**: 0 WCU (no DynamoDB writes)
- **Cost**: Lower for high volumes
- **Latency**: ~60-120s delay (Firehose buffering)
- **Visibility**: Delayed but guaranteed

### Optimization Benefits

**Strategy Selection:**

- **Intelligent**: Automatic strategy choice based on data characteristics
- **Adaptive**: Responds to integration performance over time
- **Predictable**: Consistent behavior for similar executions

**Overhead Elimination:**

- **Fast integrations**: 70s overhead eliminated
- **Large datasets**: 99.9% durability at lower cost
- **Medium volumes**: Balanced cost/performance

### Time Savings

| Volume     | Duration | noDelta % | Strategy | Time Saved           |
| ---------- | -------- | --------- | -------- | -------------------- |
| 1700 items | 7s       | 95%       | Realtime | 70s                  |
| 450 items  | 10s      | 90%       | Realtime | 5s                   |
| 140 items  | 7s       | 90%       | Realtime | 0s (under threshold) |
| 600 items  | 60s      | 15%       | Batch    | N/A (optimized)      |

## Best Practices

> **Note**: Customizing logging strategy based on these practices should only be done under direct supervision of a
> 4success TunnelHub specialist.

### 1. Understand Your Data Patterns

```typescript
class DataAwareIntegration extends DeltaIntegrationFlow<MyType> {
  // Adjust thresholds based on expected data
  protected realtimeLoggingThreshold: number = this.expectedUpdateRate() > 0.5 ? 50 : 200;

  private expectedUpdateRate(): number {
    // Calculate from historical data or parameters
    const rate = parseFloat(this.parameters?.update_rate || '0.3');
    return rate;
  }
}
```

### 2. Use Metadata for Configuration

```typescript
class MetadataConfiguredIntegration extends DeltaIntegrationFlow<MyType> {
  protected realtimeLoggingThreshold: number = parseInt(
    this.executionEvent.metadata?.find(m => m.key === 'log_threshold')?.value || '100',
  );

  protected isKnownFastIntegration(): boolean {
    return this.executionEvent.metadata?.some(m => m.key === 'fast_integration' && m.value === 'true') || false;
  }
}
```

### 3. Monitor and Adjust

```typescript
class MonitoredIntegration extends DeltaIntegrationFlow<MyType> {
  protected async afterIntegration(): Promise<void> {
    super.afterIntegration(); // Call parent

    // Log strategy choice for analysis
    const strategy = this['getLoggingStrategy']();
    console.log(`[Monitoring] Strategy: ${strategy}, Items: ${this.sourceSystemData.length}`);

    // Adjust thresholds based on metrics (if needed)
    // This could be stored in parameters for next execution
  }
}
```

### 4. Document Threshold Rationale

```typescript
/**
 * This integration processes product data with high update frequency.
 * Thresholds are adjusted to balance cost and visibility.
 *
 * - realtimeLoggingThreshold: 50 (most updates require batch)
 * - highNoDeltaRatioThreshold: 0.7 (lower than default due to high update rate)
 * - maxRealtimeItems: 800 (conservative safety limit)
 */
class ProductSyncIntegration extends DeltaIntegrationFlow<Product> {
  protected realtimeLoggingThreshold: number = 50;
  protected highNoDeltaRatioThreshold: number = 0.7;
  protected maxRealtimeItems: number = 800;
}
```

## Common Pitfalls

> **Note**: Understanding these pitfalls is important, but any customizations to avoid them should only be done under
> direct supervision of a 4success TunnelHub specialist.

### 1. Setting maxRealtimeItems Too High

```typescript
// BAD: Risk of WCU throttling
protected maxRealtimeItems: number = 10000;

// GOOD: Reasonable limit
protected maxRealtimeItems: number = 1000;
```

### 2. Overriding isKnownFastIntegration Incorrectly

```typescript
// BAD: Always returns true, defeats safety checks
protected isKnownFastIntegration(): boolean {
  return true;
}

// GOOD: Conditional logic
protected isKnownFastIntegration(): boolean {
  const isPriority = this.executionEvent.metadata?.some(m =>
    m.key === 'priority' && m.value === 'high'
  );

  const expectedDuration = parseInt(this.parameters?.expected_duration || '0');
  return isPriority || expectedDuration < 30;
}
```

### 3. Ignoring noDelta Ratio

```typescript
// BAD: Hardcoded thresholds without considering data patterns
protected realtimeLoggingThreshold: number = 200;

// GOOD: Adjust based on expected noDelta ratio
protected realtimeLoggingThreshold: number =
  this.expectedNoDeltaRatio > 0.8 ? 300 : 100;
```

### 4. Not Testing Thresholds

Always test new threshold values in a staging environment before production.

## Advanced Topics

> **Note**: Advanced topics below should only be implemented under direct supervision of a 4success TunnelHub
> specialist.

### Dynamic Threshold Adjustment

```typescript
class DynamicThresholdIntegration extends DeltaIntegrationFlow<MyType> {
  protected async beforeIntegration(): Promise<void> {
    await super.beforeIntegration();

    // Load historical performance metrics
    const paramManager = new AutomationParameter(this.tenantId, this.automationId, this.environmentId);

    const avgDuration = await paramManager.getParameter('avg_duration_seconds', '30');
    const avgNoDeltaRatio = await paramManager.getParameter('avg_no_delta_ratio', '0.5');

    // Adjust thresholds based on historical data
    this.realtimeLoggingThreshold = parseFloat(avgDuration) < 20 ? 300 : 100;

    this.highNoDeltaRatioThreshold = parseFloat(avgNoDeltaRatio);
  }

  protected async afterIntegration(): Promise<void> {
    await super.afterIntegration();

    // Save metrics for next execution
    const paramManager = new AutomationParameter(this.tenantId, this.automationId, this.environmentId);

    const duration = this.getExecutionDuration();
    const noDeltaRatio = this.getNoDeltaRatio();

    await paramManager.saveParameter('avg_duration_seconds', duration.toString());
    await paramManager.saveParameter('avg_no_delta_ratio', noDeltaRatio.toString());
  }
}
```

### Custom Strategy Selection

```typescript
class CustomStrategyIntegration extends DeltaIntegrationFlow<MyType> {
  protected defineLoggingStrategy(): LoggingStrategy {
    const totalItems = this.sourceSystemData.length;
    const estimatedDuration = this.estimateDuration();

    // Custom logic: Use realtime if fast, regardless of volume
    if (estimatedDuration < 15) {
      console.log(`[CustomStrategy] Fast integration (${estimatedDuration}s), using realtime`);
      return 'realtime';
    }

    // Otherwise, use default logic
    return super.defineLoggingStrategy();
  }

  private estimateDuration(): number {
    // Estimate based on historical data or item characteristics
    const avgProcessingTime = 0.01; // 10ms per item
    return this.sourceSystemData.length * avgProcessingTime;
  }
}
```

# Coding Standards - Phase 4 Implementation

## 🎯 TypeScript Coding Standards

### Type Safety Requirements
```typescript
// Strict TypeScript configuration
interface StrictTypeConfig {
  strict: true;
  noImplicitAny: true;
  noImplicitReturns: true;
  noImplicitThis: true;
  noUnusedLocals: true;
  noUnusedParameters: true;
  exactOptionalPropertyTypes: true;
  noUncheckedIndexedAccess: true;
}

// Explicit type definitions required
interface PoseKeypoint {
  readonly x: number;
  readonly y: number;
  readonly confidence: number;
  readonly name: KeypointName;
}

// No 'any' type usage - use proper types
type PoseDetectionResult = {
  poses: ReadonlyArray<PoseData>;
  processingTime: number;
  confidence: number;
  timestamp: number;
} | {
  error: PoseDetectionError;
  timestamp: number;
};

// Discriminated unions for error handling
type ServiceResult<T> = 
  | { success: true; data: T }
  | { success: false; error: ServiceError };
```

### Interface Design Patterns
```typescript
// Service interface pattern
export interface PoseDetectionService {
  readonly isInitialized: boolean;
  initialize(): Promise<void>;
  detectPoses(input: VideoFrame): Promise<PoseData[]>;
  cleanup(): Promise<void>;
}

// Configuration interface pattern
export interface PoseDetectionConfig {
  readonly modelType: 'MoveNet' | 'PoseNet';
  readonly confidenceThreshold: number;
  readonly maxPoses: number;
  readonly enableSmoothing: boolean;
  readonly performanceMode: 'accuracy' | 'speed' | 'balanced';
}

// Event interface pattern
export interface PoseDetectionEvent {
  readonly type: 'started' | 'completed' | 'error';
  readonly timestamp: number;
  readonly data?: unknown;
}

// Props interface pattern (React components)
export interface CameraViewProps {
  readonly cameraId?: string;
  readonly resolution: CameraResolution;
  readonly onStreamReady: (stream: MediaStream) => void;
  readonly onError: (error: CameraError) => void;
  readonly className?: string;
  readonly 'data-testid'?: string;
}
```

### Error Handling Standards
```typescript
// Custom error classes with proper inheritance
export class PoseDetectionError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly cause?: Error
  ) {
    super(message);
    this.name = 'PoseDetectionError';
    
    // Maintain proper stack trace
    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, PoseDetectionError);
    }
  }
}

export class CameraPermissionError extends PoseDetectionError {
  constructor(message = 'Camera permission denied') {
    super(message, 'CAMERA_PERMISSION_DENIED');
    this.name = 'CameraPermissionError';
  }
}

// Result type pattern for error handling
export type CameraResult<T> = 
  | { success: true; data: T }
  | { success: false; error: CameraError };

// Error boundary for React components
export const withErrorBoundary = <P extends object>(
  Component: React.ComponentType<P>
): React.ComponentType<P> => {
  return (props: P) => (
    <ErrorBoundary>
      <Component {...props} />
    </ErrorBoundary>
  );
};
```

## 🔧 React Component Standards

### Functional Component Pattern
```typescript
// Standard functional component structure
interface ComponentProps {
  readonly id: string;
  readonly className?: string;
  readonly children?: React.ReactNode;
  readonly onEvent?: (data: EventData) => void;
}

export const ComponentName: React.FC<ComponentProps> = React.memo(({
  id,
  className,
  children,
  onEvent
}) => {
  // 1. Hooks in consistent order
  const [state, setState] = useState<ComponentState>(initialState);
  const [loading, setLoading] = useState(false);
  const { data, error } = useCustomHook();
  
  // 2. Refs
  const elementRef = useRef<HTMLDivElement>(null);
  
  // 3. Memoized values
  const computedValue = useMemo(() => {
    return expensiveComputation(state);
  }, [state]);
  
  // 4. Callbacks
  const handleEvent = useCallback((event: Event) => {
    setState(prevState => ({ ...prevState, event }));
    onEvent?.(event.data);
  }, [onEvent]);
  
  // 5. Effects
  useEffect(() => {
    // Effect logic
    return () => {
      // Cleanup logic
    };
  }, [dependencies]);
  
  // 6. Early returns for conditional rendering
  if (loading) {
    return <LoadingSpinner aria-label="Loading component data" />;
  }
  
  if (error) {
    return (
      <ErrorDisplay 
        error={error} 
        onRetry={() => setState(initialState)}
      />
    );
  }
  
  // 7. Main render
  return (
    <div 
      id={id}
      ref={elementRef}
      className={cn('component-base', className)}
      role="region"
      aria-label="Component name"
    >
      {children}
    </div>
  );
});

// Display name for debugging
ComponentName.displayName = 'ComponentName';
```

### Custom Hook Standards
```typescript
// Custom hook pattern with proper return types
export interface UseCameraReturn {
  readonly stream: MediaStream | null;
  readonly isLoading: boolean;
  readonly error: CameraError | null;
  readonly startCamera: (constraints: MediaStreamConstraints) => Promise<void>;
  readonly stopCamera: () => void;
}

export const useCamera = (
  initialConstraints?: MediaStreamConstraints
): UseCameraReturn => {
  const [stream, setStream] = useState<MediaStream | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<CameraError | null>(null);
  
  const startCamera = useCallback(async (constraints: MediaStreamConstraints) => {
    setIsLoading(true);
    setError(null);
    
    try {
      const mediaStream = await navigator.mediaDevices.getUserMedia(constraints);
      setStream(mediaStream);
    } catch (err) {
      const cameraError = createCameraError(err);
      setError(cameraError);
      throw cameraError;
    } finally {
      setIsLoading(false);
    }
  }, []);
  
  const stopCamera = useCallback(() => {
    if (stream) {
      stream.getTracks().forEach(track => track.stop());
      setStream(null);
    }
  }, [stream]);
  
  // Cleanup on unmount
  useEffect(() => {
    return () => {
      stopCamera();
    };
  }, [stopCamera]);
  
  // Auto-start with initial constraints
  useEffect(() => {
    if (initialConstraints) {
      startCamera(initialConstraints);
    }
  }, [initialConstraints, startCamera]);
  
  return {
    stream,
    isLoading,
    error,
    startCamera,
    stopCamera
  };
};
```

### Context Pattern Standards
```typescript
// Context with proper type safety
interface ApplicationContextValue {
  readonly state: ApplicationState;
  readonly dispatch: React.Dispatch<ApplicationAction>;
  readonly services: ServiceContainer;
}

const ApplicationContext = React.createContext<ApplicationContextValue | null>(null);

// Provider component
export const ApplicationProvider: React.FC<{ children: React.ReactNode }> = ({ 
  children 
}) => {
  const [state, dispatch] = useReducer(applicationReducer, initialApplicationState);
  const services = useServices();
  
  const contextValue = useMemo(() => ({
    state,
    dispatch,
    services
  }), [state, dispatch, services]);
  
  return (
    <ApplicationContext.Provider value={contextValue}>
      {children}
    </ApplicationContext.Provider>
  );
};

// Custom hook for context consumption
export const useApplication = (): ApplicationContextValue => {
  const context = useContext(ApplicationContext);
  if (!context) {
    throw new Error('useApplication must be used within ApplicationProvider');
  }
  return context;
};
```

## 🏗️ Service Layer Standards

### Service Implementation Pattern
```typescript
// Service interface
export interface ServiceInterface {
  readonly name: string;
  readonly isReady: boolean;
  initialize(): Promise<void>;
  cleanup(): Promise<void>;
}

// Abstract base service
export abstract class BaseService implements ServiceInterface {
  private _isReady = false;
  
  constructor(public readonly name: string) {}
  
  get isReady(): boolean {
    return this._isReady;
  }
  
  protected setReady(ready: boolean): void {
    this._isReady = ready;
  }
  
  abstract initialize(): Promise<void>;
  abstract cleanup(): Promise<void>;
}

// Concrete service implementation
export class PoseDetectionService extends BaseService {
  private model: tf.LayersModel | null = null;
  private readonly config: PoseDetectionConfig;
  
  constructor(config: PoseDetectionConfig) {
    super('PoseDetectionService');
    this.config = { ...DEFAULT_CONFIG, ...config };
  }
  
  async initialize(): Promise<void> {
    try {
      this.model = await tf.loadLayersModel(this.config.modelUrl);
      await this.warmupModel();
      this.setReady(true);
    } catch (error) {
      throw new ServiceInitializationError(
        `Failed to initialize ${this.name}`,
        'INITIALIZATION_FAILED',
        error
      );
    }
  }
  
  async cleanup(): Promise<void> {
    if (this.model) {
      this.model.dispose();
      this.model = null;
    }
    this.setReady(false);
  }
  
  async detectPoses(input: tf.Tensor3D): Promise<PoseData[]> {
    if (!this.isReady || !this.model) {
      throw new ServiceNotReadyError('Service not initialized');
    }
    
    try {
      const predictions = await this.model.predict(input) as tf.Tensor;
      return this.parsePredictions(predictions);
    } catch (error) {
      throw new PoseDetectionError(
        'Pose detection failed',
        'DETECTION_FAILED',
        error
      );
    }
  }
  
  private async warmupModel(): Promise<void> {
    if (!this.model) return;
    
    const dummyInput = tf.zeros([1, 192, 192, 3]);
    await this.model.predict(dummyInput);
    dummyInput.dispose();
  }
  
  private parsePredictions(predictions: tf.Tensor): PoseData[] {
    // Implementation for parsing TensorFlow predictions
    return [];
  }
}
```

### Dependency Injection Pattern
```typescript
// Service container for dependency injection
export class ServiceContainer {
  private readonly services = new Map<string, ServiceInterface>();
  private readonly instances = new Map<string, any>();
  
  register<T extends ServiceInterface>(
    token: string,
    factory: () => T
  ): void {
    this.services.set(token, factory);
  }
  
  get<T extends ServiceInterface>(token: string): T {
    if (!this.instances.has(token)) {
      const factory = this.services.get(token);
      if (!factory) {
        throw new Error(`Service not registered: ${token}`);
      }
      this.instances.set(token, factory());
    }
    return this.instances.get(token);
  }
  
  async initializeAll(): Promise<void> {
    const services = Array.from(this.instances.values());
    await Promise.all(services.map(service => service.initialize()));
  }
  
  async cleanupAll(): Promise<void> {
    const services = Array.from(this.instances.values());
    await Promise.all(services.map(service => service.cleanup()));
  }
}

// Service registration
export const createServiceContainer = (): ServiceContainer => {
  const container = new ServiceContainer();
  
  container.register('camera', () => new CameraService());
  container.register('poseDetection', () => 
    new PoseDetectionService(POSE_DETECTION_CONFIG)
  );
  container.register('gaitAnalysis', () => 
    new GaitAnalysisService(GAIT_ANALYSIS_CONFIG)
  );
  
  return container;
};
```

## 🎨 CSS Standards

### CSS Module Organization
```css
/* Component.module.css */

/* 1. CSS Custom Properties (Variables) */
.component {
  --component-padding: var(--spacing-md);
  --component-border-radius: var(--border-radius-md);
  --component-background: var(--color-background-primary);
  --component-text: var(--color-text-primary);
  --component-shadow: var(--shadow-md);
}

/* 2. Layout Styles */
.container {
  display: flex;
  flex-direction: column;
  gap: var(--component-padding);
  padding: var(--component-padding);
  background: var(--component-background);
  border-radius: var(--component-border-radius);
  box-shadow: var(--component-shadow);
}

/* 3. Component States */
.container[data-state="loading"] {
  opacity: 0.7;
  pointer-events: none;
}

.container[data-state="error"] {
  border-color: var(--color-error);
  background: var(--color-error-background);
}

/* 4. Responsive Design */
@media (min-width: 768px) {
  .container {
    flex-direction: row;
    align-items: center;
  }
}

/* 5. Accessibility */
.container:focus-within {
  outline: 2px solid var(--color-focus);
  outline-offset: 2px;
}

@media (prefers-reduced-motion: reduce) {
  .container {
    transition: none;
  }
}

/* 6. High Contrast Mode */
@media (prefers-contrast: high) {
  .container {
    border: 2px solid var(--color-border);
  }
}
```

### CSS Utility Classes
```css
/* utils.css - Utility classes */

/* Layout utilities */
.flex { display: flex; }
.flex-col { flex-direction: column; }
.flex-row { flex-direction: row; }
.items-center { align-items: center; }
.justify-center { justify-content: center; }
.gap-sm { gap: var(--spacing-sm); }
.gap-md { gap: var(--spacing-md); }
.gap-lg { gap: var(--spacing-lg); }

/* Spacing utilities */
.p-sm { padding: var(--spacing-sm); }
.p-md { padding: var(--spacing-md); }
.p-lg { padding: var(--spacing-lg); }
.m-sm { margin: var(--spacing-sm); }
.m-md { margin: var(--spacing-md); }
.m-lg { margin: var(--spacing-lg); }

/* Typography utilities */
.text-sm { font-size: var(--font-size-sm); }
.text-md { font-size: var(--font-size-md); }
.text-lg { font-size: var(--font-size-lg); }
.font-medium { font-weight: var(--font-weight-medium); }
.font-bold { font-weight: var(--font-weight-bold); }

/* Color utilities */
.text-primary { color: var(--color-text-primary); }
.text-secondary { color: var(--color-text-secondary); }
.text-error { color: var(--color-error); }
.bg-primary { background-color: var(--color-background-primary); }
.bg-secondary { background-color: var(--color-background-secondary); }

/* Accessibility utilities */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

.focus-visible:focus {
  outline: 2px solid var(--color-focus);
  outline-offset: 2px;
}
```

## 📏 Performance Standards

### Performance Budgets
```typescript
// Performance monitoring configuration
export const PERFORMANCE_BUDGETS = {
  // Bundle size limits
  MAIN_BUNDLE_SIZE: 1024 * 1024, // 1MB
  VENDOR_BUNDLE_SIZE: 2048 * 1024, // 2MB
  
  // Runtime performance
  POSE_DETECTION_TIME: 33, // ms (30 FPS)
  COMPONENT_RENDER_TIME: 16, // ms (60 FPS)
  MEMORY_USAGE_LIMIT: 512 * 1024 * 1024, // 512MB
  
  // Network performance
  INITIAL_LOAD_TIME: 3000, // ms
  TIME_TO_INTERACTIVE: 5000, // ms
  LARGEST_CONTENTFUL_PAINT: 2500, // ms
};

// Performance measurement utility
export const measurePerformance = <T>(
  name: string,
  fn: () => T
): T => {
  const start = performance.now();
  const result = fn();
  const duration = performance.now() - start;
  
  if (duration > PERFORMANCE_BUDGETS.COMPONENT_RENDER_TIME) {
    console.warn(`Performance budget exceeded for ${name}: ${duration}ms`);
  }
  
  performance.mark(`${name}-end`);
  performance.measure(name, `${name}-start`, `${name}-end`);
  
  return result;
};
```

### Code Optimization Standards
```typescript
// Memoization patterns
export const ExpensiveComponent: React.FC<Props> = React.memo(({ data }) => {
  // Expensive computation memoization
  const processedData = useMemo(() => {
    return expensiveDataProcessing(data);
  }, [data]);
  
  // Callback memoization
  const handleEvent = useCallback((event: Event) => {
    // Event handling logic
  }, []);
  
  return <div>{processedData}</div>;
});

// Lazy loading pattern
const LazyComponent = React.lazy(() => import('./LazyComponent'));

export const ComponentWithLazyLoading: React.FC = () => (
  <Suspense fallback={<LoadingSpinner />}>
    <LazyComponent />
  </Suspense>
);

// Web Worker integration
export const useWebWorker = <T, R>(
  workerScript: string,
  data: T
): R | null => {
  const [result, setResult] = useState<R | null>(null);
  
  useEffect(() => {
    const worker = new Worker(workerScript);
    
    worker.postMessage(data);
    worker.onmessage = (event) => {
      setResult(event.data);
    };
    
    return () => {
      worker.terminate();
    };
  }, [workerScript, data]);
  
  return result;
};
```

This comprehensive coding standards document ensures consistent, maintainable, and performant code across the pose detection application.
# Component Architecture - Phase 2 Architecture

## 🏗️ Component Hierarchy

### React Component Tree Structure
```
App
├── ErrorBoundary
├── NotificationProvider
│   ├── Header
│   │   ├── Logo
│   │   ├── NavigationMenu
│   │   └── StatusIndicator
│   ├── MainLayout
│   │   ├── CameraSection
│   │   │   ├── CameraView
│   │   │   │   ├── VideoDisplay
│   │   │   │   ├── PoseOverlay
│   │   │   │   └── QualityIndicators
│   │   │   ├── CameraControls
│   │   │   │   ├── CameraSelector
│   │   │   │   ├── ResolutionSelector
│   │   │   │   └── StartStopButton
│   │   │   └── ControlPanel
│   │   │       ├── DetectionSettings
│   │   │       ├── VisualizationSettings
│   │   │       └── PerformanceSettings
│   │   ├── AnalysisSection
│   │   │   ├── GaitAnalysisPanel
│   │   │   │   ├── MetricsDisplay
│   │   │   │   ├── GaitVisualization
│   │   │   │   └── AnalysisControls
│   │   │   ├── PerformancePanel
│   │   │   │   ├── PerformanceMetrics
│   │   │   │   ├── PerformanceChart
│   │   │   │   └── PerformanceAlerts
│   │   │   └── ExportPanel
│   │   │       ├── ExportOptions
│   │   │       ├── DataPreview
│   │   │       └── ExportButton
│   │   └── SettingsSection
│   │       ├── SettingsPanel
│   │       │   ├── CameraSettings
│   │       │   ├── DetectionSettings
│   │       │   ├── PerformanceSettings
│   │       │   └── PrivacySettings
│   │       └── HelpPanel
│   │           ├── DocumentationViewer
│   │           ├── TutorialGuide
│   │           └── SupportContact
│   ├── Footer
│   │   ├── VersionInfo
│   │   ├── PrivacyNotice
│   │   └── AttributionLinks
│   └── GlobalComponents
│       ├── LoadingSpinner
│       ├── ErrorDisplay
│       ├── ConfirmationDialog
│       └── NotificationContainer
```

## 🔧 Core Component Specifications

### 1. Application Root Components

#### App Component
```typescript
interface AppProps {}

interface AppState {
  isInitialized: boolean;
  globalError: Error | null;
  theme: 'light' | 'dark' | 'auto';
}

export const App: React.FC<AppProps> = () => {
  const [state, setState] = useState<AppState>({
    isInitialized: false,
    globalError: null,
    theme: 'auto'
  });

  // Global initialization logic
  // Error boundary integration
  // Theme management
  // Service worker registration
};
```

#### ErrorBoundary Component
```typescript
interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ComponentType<{ error: Error; resetErrorBoundary: () => void }>;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
  errorInfo: React.ErrorInfo | null;
}

export class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  // Error catching and recovery logic
  // Error reporting integration
  // Graceful degradation
}
```

### 2. Camera Components

#### CameraView Component
```typescript
interface CameraViewProps {
  cameraId?: string;
  resolution: CameraResolution;
  onStreamReady: (stream: MediaStream) => void;
  onError: (error: CameraError) => void;
}

export const CameraView: React.FC<CameraViewProps> = ({
  cameraId,
  resolution,
  onStreamReady,
  onError
}) => {
  const videoRef = useRef<HTMLVideoElement>(null);
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const { stream, isLoading, error } = useCamera({ cameraId, resolution });

  // Camera stream management
  // Video element control
  // Canvas overlay rendering
  // Performance optimization
};
```

#### PoseOverlay Component
```typescript
interface PoseOverlayProps {
  poses: PoseData[];
  videoElement: HTMLVideoElement;
  settings: VisualizationSettings;
  className?: string;
}

export const PoseOverlay: React.FC<PoseOverlayProps> = ({
  poses,
  videoElement,
  settings,
  className
}) => {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  
  // Real-time pose rendering
  // Skeleton visualization
  // Keypoint confidence display
  // Motion trail rendering
};
```

### 3. Analysis Components

#### GaitAnalysisPanel Component
```typescript
interface GaitAnalysisPanelProps {
  poseHistory: PoseData[];
  analysisSettings: GaitAnalysisSettings;
  onExport: (data: GaitAnalysisData) => void;
}

export const GaitAnalysisPanel: React.FC<GaitAnalysisPanelProps> = ({
  poseHistory,
  analysisSettings,
  onExport
}) => {
  const [analysisData, setAnalysisData] = useState<GaitAnalysisData>();
  const [isAnalyzing, setIsAnalyzing] = useState(false);

  // Gait pattern analysis
  // Metrics calculation
  // Visualization rendering
  // Export functionality
};
```

#### PerformanceMonitor Component
```typescript
interface PerformanceMonitorProps {
  updateInterval?: number;
  onPerformanceChange: (metrics: PerformanceMetrics) => void;
}

export const PerformanceMonitor: React.FC<PerformanceMonitorProps> = ({
  updateInterval = 1000,
  onPerformanceChange
}) => {
  const [metrics, setMetrics] = useState<PerformanceMetrics>();

  // FPS monitoring
  // Memory usage tracking
  // Latency measurement
  // Performance alerts
};
```

### 4. Control Components

#### ControlPanel Component
```typescript
interface ControlPanelProps {
  detectionState: DetectionState;
  settings: ApplicationSettings;
  onSettingsChange: (settings: Partial<ApplicationSettings>) => void;
  onDetectionToggle: () => void;
}

export const ControlPanel: React.FC<ControlPanelProps> = ({
  detectionState,
  settings,
  onSettingsChange,
  onDetectionToggle
}) => {
  // Detection control
  // Settings management
  // Real-time adjustments
  // State synchronization
};
```

#### SettingsPanel Component
```typescript
interface SettingsPanelProps {
  settings: ApplicationSettings;
  onSettingsChange: (settings: Partial<ApplicationSettings>) => void;
  onReset: () => void;
}

export const SettingsPanel: React.FC<SettingsPanelProps> = ({
  settings,
  onSettingsChange,
  onReset
}) => {
  // Configuration management
  // Settings validation
  // Preference persistence
  // Reset functionality
};
```

## 🔄 Component Communication Patterns

### 1. Props Down, Events Up Pattern
```typescript
// Parent Component
export const ParentComponent: React.FC = () => {
  const [state, setState] = useState<ComponentState>();

  const handleChildEvent = (data: EventData) => {
    setState(prevState => ({
      ...prevState,
      data
    }));
  };

  return (
    <ChildComponent
      data={state.data}
      onEvent={handleChildEvent}
    />
  );
};

// Child Component
export const ChildComponent: React.FC<ChildComponentProps> = ({
  data,
  onEvent
}) => {
  const handleUserAction = () => {
    onEvent(newData);
  };

  return <div onClick={handleUserAction}>{data}</div>;
};
```

### 2. Context-Based State Management
```typescript
// Application Context
interface ApplicationContextValue {
  state: ApplicationState;
  dispatch: React.Dispatch<ApplicationAction>;
  services: ServiceContainer;
}

export const ApplicationContext = React.createContext<ApplicationContextValue>();

// Context Provider
export const ApplicationProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [state, dispatch] = useReducer(applicationReducer, initialState);
  const services = useServices();

  return (
    <ApplicationContext.Provider value={{ state, dispatch, services }}>
      {children}
    </ApplicationContext.Provider>
  );
};

// Context Consumer Hook
export const useApplication = () => {
  const context = useContext(ApplicationContext);
  if (!context) {
    throw new Error('useApplication must be used within ApplicationProvider');
  }
  return context;
};
```

### 3. Custom Hook Patterns
```typescript
// Camera Hook
export const useCamera = (options: CameraOptions) => {
  const [stream, setStream] = useState<MediaStream | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const initializeCamera = async () => {
      setIsLoading(true);
      try {
        const mediaStream = await navigator.mediaDevices.getUserMedia(options);
        setStream(mediaStream);
        setError(null);
      } catch (err) {
        setError(err as Error);
      } finally {
        setIsLoading(false);
      }
    };

    initializeCamera();

    return () => {
      if (stream) {
        stream.getTracks().forEach(track => track.stop());
      }
    };
  }, [options]);

  return { stream, isLoading, error };
};

// Pose Detection Hook
export const usePoseDetection = (videoElement: HTMLVideoElement | null) => {
  const [poses, setPoses] = useState<PoseData[]>([]);
  const [isDetecting, setIsDetecting] = useState(false);
  const { poseService } = useServices();

  useEffect(() => {
    if (!videoElement || !isDetecting) return;

    const detectPoses = async () => {
      try {
        const detectedPoses = await poseService.detectPoses(videoElement);
        setPoses(detectedPoses);
      } catch (error) {
        console.error('Pose detection failed:', error);
      }
    };

    const intervalId = setInterval(detectPoses, 1000 / 30); // 30 FPS

    return () => clearInterval(intervalId);
  }, [videoElement, isDetecting, poseService]);

  return { poses, isDetecting, setIsDetecting };
};
```

## 🎨 Component Styling Architecture

### CSS Module Structure
```css
/* Component.module.css */
.container {
  position: relative;
  width: 100%;
  height: 100%;
}

.videoContainer {
  position: relative;
  overflow: hidden;
  border-radius: var(--border-radius-lg);
  background: var(--color-background-secondary);
}

.canvas {
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
  z-index: 10;
}

.controls {
  display: flex;
  gap: var(--spacing-md);
  align-items: center;
  padding: var(--spacing-sm);
  background: var(--color-background-primary);
  border-radius: var(--border-radius-md);
}

.button {
  padding: var(--spacing-sm) var(--spacing-md);
  background: var(--color-primary);
  color: var(--color-text-on-primary);
  border: none;
  border-radius: var(--border-radius-sm);
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.button:hover {
  background: var(--color-primary-hover);
}

.button:disabled {
  background: var(--color-disabled);
  cursor: not-allowed;
}
```

### Responsive Design Pattern
```css
/* Responsive breakpoints */
.container {
  /* Mobile first approach */
  display: flex;
  flex-direction: column;
  gap: var(--spacing-md);
}

@media (min-width: 768px) {
  .container {
    flex-direction: row;
    gap: var(--spacing-lg);
  }
}

@media (min-width: 1024px) {
  .container {
    max-width: 1200px;
    margin: 0 auto;
  }
}

@media (min-width: 1440px) {
  .container {
    max-width: 1400px;
  }
}
```

## 🔧 Component Testing Strategy

### Unit Testing Pattern
```typescript
// Component.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { CameraView } from './CameraView';

describe('CameraView Component', () => {
  const mockProps = {
    cameraId: 'test-camera',
    resolution: { width: 640, height: 480 },
    onStreamReady: jest.fn(),
    onError: jest.fn()
  };

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('should render camera view with video element', () => {
    render(<CameraView {...mockProps} />);
    
    const videoElement = screen.getByRole('video');
    expect(videoElement).toBeInTheDocument();
  });

  it('should handle camera permission grant', async () => {
    const mockStream = new MediaStream();
    jest.spyOn(navigator.mediaDevices, 'getUserMedia')
      .mockResolvedValue(mockStream);

    render(<CameraView {...mockProps} />);

    await waitFor(() => {
      expect(mockProps.onStreamReady).toHaveBeenCalledWith(mockStream);
    });
  });

  it('should handle camera permission denial', async () => {
    const mockError = new Error('Permission denied');
    jest.spyOn(navigator.mediaDevices, 'getUserMedia')
      .mockRejectedValue(mockError);

    render(<CameraView {...mockProps} />);

    await waitFor(() => {
      expect(mockProps.onError).toHaveBeenCalledWith(
        expect.objectContaining({ message: 'Permission denied' })
      );
    });
  });
});
```

### Integration Testing Pattern
```typescript
// Integration.test.tsx
import { render, screen } from '@testing-library/react';
import { ApplicationProvider } from '../context/ApplicationContext';
import { MainLayout } from './MainLayout';

describe('MainLayout Integration', () => {
  const renderWithProviders = (component: React.ReactElement) => {
    return render(
      <ApplicationProvider>
        {component}
      </ApplicationProvider>
    );
  };

  it('should integrate camera and pose detection components', () => {
    renderWithProviders(<MainLayout />);
    
    expect(screen.getByTestId('camera-view')).toBeInTheDocument();
    expect(screen.getByTestId('pose-overlay')).toBeInTheDocument();
    expect(screen.getByTestId('control-panel')).toBeInTheDocument();
  });
});
```

This component architecture provides a scalable, maintainable, and testable foundation for the pose detection application with clear separation of concerns and efficient communication patterns.